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
          accept=".jpeg,.jpg,.png"
          :auto-upload="false"
          :before-upload="onBeforeUpload"
          :on-change="handleUploadChange"
        >
          <ElIcon class="el-icon--upload"><UploadFilled /></ElIcon>
          <div class="el-upload__text">
            拖动图片到区域内或 <em>点击选择图片</em>
          </div>
          <template #tip>
            <div class="el-upload__tip">jpg/jpeg/png</div>
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
const IMAGE_TYPE_JPEG = "jpeg"
const IMAGE_TYPE_PNG = "png"
const JPEG_SIGNATURE = [0xff, 0xd8, 0xff]
const PNG_SIGNATURE = [137, 80, 78, 71, 13, 10, 26, 10]
const PNG_CHUNK_TYPE_EXIF = "eXIf"
const PNG_CHUNK_TYPE_IDAT = "IDAT"

function handleClear() {
  files.value = [];
  result.value = [];
}

function isJpegData(bytes) {
  return JPEG_SIGNATURE.every((byte, index) => bytes[index] === byte);
}

function isPngData(bytes) {
  return PNG_SIGNATURE.every((byte, index) => bytes[index] === byte);
}

function getImageTypeFromBytes(bytes) {
  if (isJpegData(bytes)) {
    return IMAGE_TYPE_JPEG;
  }

  if (isPngData(bytes)) {
    return IMAGE_TYPE_PNG;
  }

  return "";
}

async function getImageFileType(file) {
  const bytes = new Uint8Array(await file.slice(0, PNG_SIGNATURE.length).arrayBuffer());

  return getImageTypeFromBytes(bytes);
}

function isSupportedImageType(imageType) {
  return imageType === IMAGE_TYPE_JPEG || imageType === IMAGE_TYPE_PNG;
}

async function onBeforeUpload(file) {
  const imageType = await getImageFileType(file);
  const isSupportedImage = isSupportedImageType(imageType);

  if (!isSupportedImage) {
    ElMessage.error("仅支持 jpg/jpeg/png 图片");
  }

  return isSupportedImage;
}

async function handleUploadChange(uploadFile) {
  const rawFile = uploadFile.raw;

  if (!rawFile || isSupportedImageType(await getImageFileType(rawFile))) {
    return;
  }

  files.value = files.value.filter((file) => file.uid !== uploadFile.uid);
  ElMessage.error("仅支持 jpg/jpeg/png 图片");
}

function formatExifDateTime(value) {
  const date = new Date(value);
  const pad = (num) => String(num).padStart(2, "0");

  return `${date.getFullYear()}:${pad(date.getMonth() + 1)}:${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}:${pad(date.getSeconds())}`;
}

async function handleChangeDate() {
  if(!dateTime.value) {
    ElMessage.error('未选择拍摄日期时间')
    return
  }
  updateLoading.value = true
  result.value = []
  try {
    const date = formatExifDateTime(dateTime.value)
    const updatedFiles = await Promise.all(unref(files.value).map((item) => insertPhoto(date, item.raw, item.name)));
    result.value = updatedFiles
  } catch(e) {
    console.error(e)
    ElMessage.error(e.message || '图片处理失败')
  } finally {
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

function createExifString(dateTime) {
  const exif = {
    [piexifjs.ExifIFD.DateTimeOriginal]: dateTime,
  };

  return piexifjs.dump({ Exif: exif });
}

function readFileAsDataURL(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();

    reader.onload = (e) => resolve(e.target.result);
    reader.onerror = () => reject(reader.error);
    reader.readAsDataURL(file);
  });
}

function binaryStringToUint8Array(value) {
  const bytes = new Uint8Array(value.length);

  for (let index = 0; index < value.length; index += 1) {
    bytes[index] = value.charCodeAt(index);
  }

  return bytes;
}

function getImageMimeType(imageType, fallback) {
  if (imageType === IMAGE_TYPE_PNG) {
    return "image/png";
  }

  if (imageType === IMAGE_TYPE_JPEG) {
    return "image/jpeg";
  }

  return fallback;
}

function readUint32(bytes, offset) {
  return new DataView(bytes.buffer, bytes.byteOffset + offset, 4).getUint32(0);
}

function writeUint32(bytes, offset, value) {
  bytes[offset] = (value >>> 24) & 0xff;
  bytes[offset + 1] = (value >>> 16) & 0xff;
  bytes[offset + 2] = (value >>> 8) & 0xff;
  bytes[offset + 3] = value & 0xff;
}

function getPngChunkType(bytes, offset) {
  return String.fromCharCode(bytes[offset + 4], bytes[offset + 5], bytes[offset + 6], bytes[offset + 7]);
}

function getCrc32(bytes) {
  let crc = 0xffffffff;

  for (const byte of bytes) {
    crc ^= byte;

    for (let bit = 0; bit < 8; bit += 1) {
      crc = (crc >>> 1) ^ (crc & 1 ? 0xedb88320 : 0);
    }
  }

  return (crc ^ 0xffffffff) >>> 0;
}

function createPngChunk(type, data) {
  const typeBytes = binaryStringToUint8Array(type);
  const chunk = new Uint8Array(12 + data.length);

  writeUint32(chunk, 0, data.length);
  chunk.set(typeBytes, 4);
  chunk.set(data, 8);
  writeUint32(chunk, 8 + data.length, getCrc32(chunk.slice(4, 8 + data.length)));

  return chunk;
}

function insertPngExif(bytes, exifStr) {
  const exifBytes = binaryStringToUint8Array(exifStr.slice(6));
  const exifChunk = createPngChunk(PNG_CHUNK_TYPE_EXIF, exifBytes);
  const chunks = [];
  let insertedExif = false;
  let offset = PNG_SIGNATURE.length;

  chunks.push(bytes.slice(0, offset));

  while (offset < bytes.length) {
    const length = readUint32(bytes, offset);
    const chunkEnd = offset + 12 + length;
    const chunkType = getPngChunkType(bytes, offset);

    if (!insertedExif && chunkType === PNG_CHUNK_TYPE_IDAT) {
      chunks.push(exifChunk);
      insertedExif = true;
    }

    if (chunkType !== PNG_CHUNK_TYPE_EXIF) {
      chunks.push(bytes.slice(offset, chunkEnd));
    }

    offset = chunkEnd;
  }

  if (!insertedExif) {
    chunks.splice(2, 0, exifChunk);
  }

  return new Blob(chunks, { type: "image/png" });
}

async function insertPhoto(dateTime, file, fileName) {
  const exifStr = createExifString(dateTime);
  const imageType = await getImageFileType(file);

  if (imageType === IMAGE_TYPE_PNG) {
    const inserted = insertPngExif(new Uint8Array(await file.arrayBuffer()), exifStr);

    return {
      name: fileName,
      raw: new File([inserted], fileName, { type: getImageMimeType(imageType, file.type) }),
    };
  }

  if (imageType !== IMAGE_TYPE_JPEG) {
    throw new Error(`${fileName} 不是支持的图片格式`);
  }

  const inserted = piexifjs.insert(exifStr, await readFileAsDataURL(file));

  return {
    name: fileName,
    raw: base64ToFile(inserted, fileName, getImageMimeType(imageType, file.type)),
  };
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
