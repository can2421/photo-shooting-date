<template>
  <ElConfigProvider :locale="zhCn">
    <div>
      <div class="upload-box">
        <ElUpload
          class="upload-demo"
          v-model:file-list="files"
          drag
          action="#"
          multiple
          accept=".jpeg,.jpg"
          :auto-upload="false"
          :before-upload="onBeforeUpload"
        >
          <ElIcon class="el-icon--upload"><UploadFilled /></ElIcon>
          <div class="el-upload__text">
            拖动图片到区域内或 <em>点击选择图片</em>
          </div>
          <template #tip>
            <div class="el-upload__tip">jpg/jepg</div>
          </template>
        </ElUpload>
      </div>
      <ElRow>
        <ElCol :span="24">
          <ElDatePicker
            style="margin: 10px 10px 10px 0;"
            v-model="dateTime"
            type="datetime"
            placeholder="选择拍摄日期"
          ></ElDatePicker>
        </ElCol>
        <ElCol :span="24">
          <ElButton type="danger" @click="handleClear">清空图片</ElButton>
          <ElButton type="primary" :loading="updateLoading" @click="handleChangeDate"
            >批量修改拍摄日期</ElButton
          >
          <ElButton v-if="result.length > 0" :loading="updateLoading || downloading" type="primary" @click="downloadZip"
            >批量下载</ElButton
          >
        </ElCol>
      </ElRow>
    </div>
  </ElConfigProvider>
</template>

<script setup>
import zhCn from 'element-plus/dist/locale/zh-cn.mjs'
import { ref, unref } from "vue";
import { UploadFilled } from "@element-plus/icons-vue";
import JSZip from "jszip";
import piexifjs from "piexifjs";
import { ElMessage } from 'element-plus'

const files = ref([]);
const result = ref([]);
const dateTime = ref("");
const updateLoading = ref(false)
const downloading = ref(false)

function handleClear() {
  files.value = [];
  result.value = [];
}

function handleChangeDate() {
  if(!dateTime.value) {
    ElMessage.error('未选择拍摄日期时间')
    return
  }
  updateLoading.value = true
  try {
    const date = new Date(dateTime.value).toLocaleString().replace(/\//g, ':')
    console.log(date);
    unref(files.value).forEach(async (item, index) => {
      await insertPhoto(date, item.raw, item.name)
      if(index === files.value.length - 1) {
        updateLoading.value = false
      }
    });
  } catch(e) {
    console.error(e)
    updateLoading.value = false
  }
}

async function downloadZip() {
  downloading.value = true
  const zip = new JSZip();

  result.value.forEach((file) => {
    zip.file(file.name, file.raw);
  });

  const zipBlob = await zip.generateAsync({ type: "blob" });

  const url = URL.createObjectURL(zipBlob);
  const link = document.createElement("a");
  link.href = url;
  link.download = "photos.zip";
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
  downloading.value = false
}

function base64ToFile(base64String, fileName, mimeType) {
  // 提取Base64编码部分
  const base64Content = base64String.split(";base64,").pop();

  // 将Base64字符串转换为字节数组（ArrayBuffer）
  const byteCharacters = atob(base64Content);
  const byteNumbers = new Array(byteCharacters.length);
  for (let i = 0; i < byteCharacters.length; i++) {
    byteNumbers[i] = byteCharacters.charCodeAt(i);
  }
  const byteArray = new Uint8Array(byteNumbers);

  // 创建新的File对象
  const file = new File([byteArray], fileName, { type: mimeType });

  return file;
}

function insertPhoto(dateTime, file, fileName) {
  return new Promise((resolve) => {
    const exif = {
      [piexifjs.ExifIFD.DateTimeOriginal]: dateTime,
    };
    const exifStr = piexifjs.dump({ Exif: exif });
    const reader = new FileReader();
    reader.onload = function (e) {
      const inserted = piexifjs.insert(exifStr, e.target.result);
      result.value.push({
        name: fileName,
        raw: base64ToFile(inserted, file.type, fileName),
      });
      resolve()
    };
    reader.readAsDataURL(file);
  })
}
</script>

<script>
export default {
  name: "App",
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

.upload-box {
  display: flex;
  justify-content: center;
}

.upload-demo {
  min-width: 40%;
  max-width: 50%;
}

.el-upload-list {
  max-height: 55vh;
  overflow-y: auto;
}

.avatar-uploader .avatar {
  width: 178px;
  height: 178px;
  display: block;
}

.avatar-uploader .el-upload {
  border: 1px dashed var(--el-border-color);
  border-radius: 6px;
  cursor: pointer;
  position: relative;
  overflow: hidden;
  transition: var(--el-transition-duration-fast);
}

.avatar-uploader .el-upload:hover {
  border-color: var(--el-color-primary);
}

.el-icon.avatar-uploader-icon {
  font-size: 28px;
  color: #8c939d;
  width: 178px;
  height: 178px;
  text-align: center;
}
.el-upload__tip {
  font-size: 14px;
}
</style>
