<template>
  <el-container>
    <el-header>
      <h1>蓝牙数据采集</h1>
    </el-header>
    <el-main>
      <el-button type="primary" @click="connectDevice">连接蓝牙设备</el-button>
      <el-button type="success" @click="exportAndUploadCSV">完成采集</el-button>

      <el-table :data="imuDataArray" border style="margin-top: 20px">
        <el-table-column prop="AccX" label="加速度 X (g)" />
        <el-table-column prop="AccY" label="加速度 Y (g)" />
        <el-table-column prop="AccZ" label="加速度 Z (g)" />
        <el-table-column prop="AsX" label="角速度 X (°/s)" />
        <el-table-column prop="AsY" label="角速度 Y (°/s)" />
        <el-table-column prop="AsZ" label="角速度 Z (°/s)" />
        <el-table-column prop="AngX" label="角度 X (°)" />
        <el-table-column prop="AngY" label="角度 Y (°)" />
        <el-table-column prop="AngZ" label="角度 Z (°)" />
      </el-table>
    </el-main>
  </el-container>
</template>

<script setup>
import { ref } from "vue";
import { ElMessage } from "element-plus";

const SERVICE_UUID = "0000ffe5-0000-1000-8000-00805f9a34fb";
const CHARACTERISTIC_UUID_NOTIFY = "0000ffe4-0000-1000-8000-00805f9a34fb";

let device, characteristic;
const imuDataArray = ref([]);

// 连接蓝牙设备
async function connectDevice() {
  try {
    device = await navigator.bluetooth.requestDevice({
      filters: [{ namePrefix: "WT" }],
      optionalServices: [SERVICE_UUID],
    });

    const server = await device.gatt.connect();
    const service = await server.getPrimaryService(SERVICE_UUID);
    characteristic = await service.getCharacteristic(CHARACTERISTIC_UUID_NOTIFY);
    await characteristic.startNotifications();

    characteristic.addEventListener("characteristicvaluechanged", handleData);

    ElMessage.success("蓝牙设备已连接，开始监听数据...");
  } catch (error) {
    ElMessage.error(`连接失败: ${error.message}`);
  }
}

// 处理蓝牙数据
function handleData(event) {
  const value = event.target.value;
  const bytes = new Uint8Array(value.buffer);
  if (bytes.length !== 20 || bytes[0] !== 0x55) return;

  if (bytes[1] === 0x61) {
    const getInt16 = (i) => {
      let val = bytes[i + 1] << 8 | bytes[i];
      return val >= 32768 ? val - 65536 : val;
    };

    const data = {
      AccX: (getInt16(2) / 32768 * 16).toFixed(3),
      AccY: (getInt16(4) / 32768 * 16).toFixed(3),
      AccZ: (getInt16(6) / 32768 * 16).toFixed(3),
      AsX: (getInt16(8) / 32768 * 2000).toFixed(3),
      AsY: (getInt16(10) / 32768 * 2000).toFixed(3),
      AsZ: (getInt16(12) / 32768 * 2000).toFixed(3),
      AngX: (getInt16(14) / 32768 * 180).toFixed(3),
      AngY: (getInt16(16) / 32768 * 180).toFixed(3),
      AngZ: (getInt16(18) / 32768 * 180).toFixed(3),
    };

    imuDataArray.value.push(data);
    ElMessage.info("收到新数据");
  }
}

// 导出并上传 CSV 文件
function exportAndUploadCSV() {
  if (imuDataArray.value.length === 0) {
    ElMessage.warning("没有数据可以导出！");
    return;
  }

  // 构建 CSV 文件内容
  const headers = Object.keys(imuDataArray.value[0]).join(",") + "\n";
  const rows = imuDataArray.value.map(row => Object.values(row).join(",")).join("\n");
  const csvContent = headers + rows;

  // 创建 Blob 对象
  const blob = new Blob([csvContent], { type: "text/csv" });
  const formData = new FormData();
  formData.append("file", blob, "imu_data.csv");

  // 上传 CSV 文件到后端
  fetch("/upload-csv", {
    method: "POST",
    body: formData,
  })
    .then(response => response.json())
    .then(result => {
      ElMessage.success(`上传成功: ${result.message}`);
    })
    .catch(error => {
      ElMessage.error(`上传失败: ${error.message}`);
    });
}
</script>

<style scoped>
.el-header {
  background-color: #409eff;
  color: white;
  text-align: center;
  line-height: 60px;
}
.el-main {
  padding: 20px;
}
</style>