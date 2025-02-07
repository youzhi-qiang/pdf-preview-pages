<template>

  <div class="pdf-preview">

    <div class="pdf-header">
      <div class="pdf-header-left">
        <el-icon :style="{ color: 'gray', cursor: 'pointer' }" @click="showMulu = !showMulu">
          <font-awesome-icon :icon="fasIcon.faBars" />
        </el-icon>
        <el-icon :style="{ color: 'gray', cursor: 'pointer' }" @click="pageZoomIn">
          <font-awesome-icon :icon="fasIcon.faSearchMinus" />
        </el-icon>

        <el-select v-model="state.scale" placeholder="缩放比例" style="width: 110px; margin: 0 2px">
          <el-option label="50%" :value="0.5">50%</el-option>
          <el-option label="75%" :value="0.75">75%</el-option>
          <el-option label="100%" :value="1">100%</el-option>
          <el-option label="125%" :value="1.25">125%</el-option>
          <el-option label="150%" :value="1.5">150%</el-option>
          <el-option label="200%" :value="2">200%</el-option>
        </el-select>

        <el-icon :style="{ color: 'gray', cursor: 'pointer' }" @click="pageZoomOut">
          <font-awesome-icon :icon="fasIcon.faSearchPlus" />
        </el-icon>
      </div>
      <div class="pdf-header-center">
        <el-pagination v-if="!state.showAllPages" v-model:current-page="state.pageNum"
          layout="prev, pager, next, jumper,total" :page-size="1" :total="state.numPages" hide-on-single-page />
        <div v-else>共{{ state.numPages }}页</div>
      </div>
      <div class="pdf-header-right">
        <el-checkbox v-model="state.showAllPages" @change="showAllPagesChange">展示所有</el-checkbox>
        <el-tooltip effect="dark" content="下载">
          <el-icon @click="downloadPdf" :style="{ color: 'rgb(56, 148, 255)', cursor: 'pointer' }"
            v-loading.fullscreen.lock="fullscreenLoading">
            <font-awesome-icon :icon="fasIcon.faDownload" />
          </el-icon>
        </el-tooltip>
        <el-tooltip effect="dark" content="打印">
          <el-icon @click="printPdf" :style="{ color: 'rgb(56, 148, 255)', cursor: 'pointer' }">
            <font-awesome-icon :icon="fasIcon.faPrint" />
          </el-icon>
        </el-tooltip>
      </div>
    </div>

    <!-- Main Container -->
    <div class="pdf-content">
      <!-- 左侧缩略图导航 -->
      <div class="thumbnail-nav" v-show="showMulu">
        <el-scrollbar>
          <div class="thumbnail-list">
            <div v-for="pageNum in state.numPages" :key="pageNum" class="thumbnail-item"
              :class="{ active: state.pageNum === pageNum }" @click="goToPage(pageNum)">
              <div class="page-number">第 {{ pageNum }} 页</div>
              <div class="thumbnail-wrapper">
                <vue-pdf-embed :source="padUrl" :page="pageNum" class="vue-pdf-embed-thumbnail"
                  :style="{ transform: 'scale(0.75)' }" />
              </div>
            </div>
          </div>
        </el-scrollbar>
      </div>

      <!-- 右侧PDF预览 -->
      <div class="pdf-viewer" :class="{ 'expanded': !showMulu }">
        <el-scrollbar v-loading="loading">
          <div class="pdf-container" :class="{ 'expanded': !showMulu }">
            <vue-pdf-embed ref="pdfRef" :source="padUrl" :page="state.pageNum || undefined" :style="scale"
              class="vue-pdf-embed" />
          </div>
        </el-scrollbar>
      </div>
    </div>
  </div>
</template>
<script setup lang="ts">
import { reactive, onMounted, computed, onBeforeMount, ref, watch, watchEffect } from 'vue';
import VuePdfEmbed from "vue-pdf-embed";
import { createLoadingTask } from "vue3-pdfjs";
import { downFile } from '@/utils/system/commonUse';
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome';
import { ElMessage, ElMessageBox } from 'element-plus';

import { faPrint, faDownload, faExclamationCircle, faSearchPlus, faSearchMinus, faBars } from '@fortawesome/free-solid-svg-icons';
import axios from 'axios';

declare interface ImportMeta {
  env: {
    VITE_BASE_URL: string;
    [key: string]: any;
  }
}

interface PDFDocumentProxy {
  numPages: number;
  getPage: (pageNumber: number) => Promise<PDFPageProxy>;
}

interface PDFPageProxy {
  getViewport: (options: { scale: number }) => PDFPageViewport;
  render: (options: any) => { promise: Promise<void> };
}

interface PDFPageViewport {
  width: number;
  height: number;
}

const baseURL = import.meta.env.VITE_BASE_URL;
const loading = ref(false);
const showMulu = ref(false);
const pdfRef = ref();
const fasIcon = reactive({
  faDownload,
  faPrint,
  faExclamationCircle,
  faSearchPlus,
  faSearchMinus,
  faBars
});
const props = defineProps({
  url: {
    type: String,
    required: true
  },
  fileName: {
    type: String,
    default: '附件1'
  },
  isDangAn: {
    type: Boolean,
    default: false
  }
})
const showAllPagesChange = () => {
  loading.value = true;
  state.pageNum = state.showAllPages ? null : 1
  loading.value = false;
  showMulu.value = false
}
const padUrl = ref('')  //预览pdf文件地址
const state = reactive({
  isLoading: true,
  pageNum: 1,
  scale: 0.75,
  numPages: 0,
  showAllPages: false,
  showToolbar: false,
});

onMounted(async () => {
  try {
    loading.value = true;
    padUrl.value = props.url
    console.log('padUrl.value', padUrl.value)

    const loadingTask = createLoadingTask(padUrl.value);
    const pdf = await loadingTask.promise;

    state.numPages = pdf.numPages;
    state.pageNum = 1;
    showMulu.value = true;
  } catch (error) {
    console.error('Error loading PDF:', error);
  } finally {
    loading.value = false;
  }
});
// Handle page navigation
const goToPage = (pageNum: number) => {
  if (pageNum >= 1 && pageNum <= state.numPages) {
    state.pageNum = pageNum;
    state.showAllPages = false;
  }
};

const scale = computed(() => ({ transform: `scale(${state.scale})` }))
function lastPage() {
  if (state.pageNum && state.pageNum > 1) {
    state.pageNum -= 1;
  }
}
function nextPage() {
  if (state.pageNum && state.numPages && state.pageNum < state.numPages) {
    state.pageNum += 1;
  }
}
function pageZoomOut() {
  if (state.scale < 2) {
    state.scale += 0.25;
  }
}
function pageZoomIn() {
  if (state.scale > 1) {
    state.scale -= 0.25;
  }
}
const printPdf = () => {
  pdfRef.value.print(100, props.fileName, true)
}
const fullscreenLoading = ref(false);

const downloadPdf = () => {
  fullscreenLoading.value = true;
  const url = props.url
  downloadFileWithLink(url, { token: '' }, () => {
    setTimeout(() => {
      fullscreenLoading.value = false;

    }, 2000)

  });
  // ElMessage.success(`下载成功`);


}
function downloadFileWithLink(url: any, options: { [x: string]: string | number | boolean; }, callback: () => void) {
  // 构建过滤 undefined 的查询参数
  const params = Object.keys(options)
    .filter(key => options[key] !== undefined && options[key] !== null && options[key] !== '') // 过滤掉无效值
    .map(key => `${key}=${encodeURIComponent(options[key])}`) // 拼接成 key=value
    .join('&') // 使用 & 连接所有参数

  const downloadUrl = `${url}?${params}`

  // 创建一个隐形的 a 标签
  const a = document.createElement('a')
  a.style.display = 'none'
  a.href = downloadUrl

  // 设置 download 属性以便浏览器进行下载
  a.download = 'file' // 这里可以设置下载的文件名

  // 添加到文档中并模拟点击
  document.body.appendChild(a)
  a.click()

  // 移除 a 标签
  document.body.removeChild(a)

  // 如果有回调，执行回调
  if (callback) callback()
}
watch(() => state.showAllPages, (val) => {
  state.pageNum = val ? null : 1

})
 
watch(() => state.scale, (newValue) => {
  // 强制转换为数字类型
  state.scale = Number(newValue);
});
</script>
<style lang="css" scoped>
.pdf-preview {
  position: relative;
  height: 100%;
  box-sizing: border-box;
  background-color: e9e9e9;
}

.pdf-wrap {
  overflow-y: hidden;
  background: #e5e5e5;
  height: 100%;
  padding: 0px 10px;
}

.vue-pdf-embed {
  text-align: center;
  width: 540px;
  border: 1px solid #e5e5e5;
  margin: 0 auto;
  box-sizing: border-box;
}

.app-header {
  display: flex;
  justify-content: space-between;
  padding: 16px;
  box-shadow: 0 2px 8px 4px rgba(0, 0, 0, 0.1);
  background-color: #555;
  color: #ddd;
}

.app-header>* {
  display: flex;
  align-items: center;
  gap: 4px;
}

.app-content {
  padding: 24px 16px;
}

.vue-pdf-embed {
  margin: 0 auto;
}

.vue-pdf-embed__page {
  margin-bottom: 8px;
  box-shadow: 0 2px 8px 4px rgba(0, 0, 0, 0.1);
}

.pdf-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 20px;
  background-color: #f0f0f0;
}

.pdf-header-left {
  display: flex;
  align-items: center;
  gap: 15px;
}

.pdf-header-center {
  display: flex;
  align-items: center;
  gap: 15px;
}

.pdf-header-right {
  display: flex;
  align-items: center;
  gap: 15px;
}


.left-mulu {
  position: fixed;
  left: 0;
  top: 0;
  width: 200px;
  height: 100%;
  background-color: #fff;
  overflow-y: auto;
  box-shadow: 2px 0 8px rgba(0, 0, 0, 0.1);
  padding: 10px;
}

.mulu-pages {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.page-thumbnail {
  width: 80px;
  height: 100px;
  cursor: pointer;
  border: 1px solid #ddd;
  padding: 2px;
}

.right-content {
  flex: 1;
  margin-left: 200px;
  padding: 20px;
}

.pdf-content {
  display: flex;
  height: calc(100% - 60px);
  background: #f5f5f5;
  width: 100%;
  position: relative;
}

.thumbnail-nav {
  position: absolute;
  left: 0;
  top: 0;
  width: 200px;
  height: 100%;
  background: #f5f5f5;
  border-right: 1px solid #e0e0e0;
  z-index: 1;
}

.thumbnail-list {
  display: flex;
  flex-direction: column;
  gap: 1px;
}

.thumbnail-item {
  cursor: pointer;
  background: #fff;
  border: none;
  border-radius: 0;
  padding: 10px;
  transition: all 0.2s;
  margin: 15px;

}

.thumbnail-wrapper {
  width: 100%;
  height: 160px;
  position: relative;
  overflow: hidden;
  background: #fff;
}

.vue-pdf-embed-thumbnail {
  width: 100%;
  height: 100%;
  transform-origin: center;
  position: relative;
  background: #fff;
}

.vue-pdf-embed {
  width: auto !important;
  border: none !important;
}

.page-number {
  font-size: 12px;
  color: #606266;
  text-align: center;
  padding: 4px 0;
  background: #f5f5f5;
  margin: 0;
}

.thumbnail-item.active {
  background: #e6f1fc;
  border: none;
  box-shadow: none;
}

.thumbnail-item:hover {
  transform: none;
  background: #f0f7ff;
  box-shadow: none;
}

.pdf-viewer {
  flex: 1;
  padding: 20px;
  overflow: auto;
  transition: all 0.3s ease;
  width: calc(100% - 200px);
  margin-left: auto;
  margin-right: auto;
}

.pdf-viewer.expanded {
  width: 100%;
}

.pdf-container {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  width: 100%;
  max-width: 900px;
  margin: 0 auto;
  transition: all 0.3s ease;
}

.pdf-container.expanded {
  max-width: 1200px;
}

.vue-pdf-embed {
  width: 100% !important;
  background: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  border-radius: 4px;
  margin: 0 auto;
}

@media screen and (max-width: 1600px) {
  .pdf-container {
    max-width: 800px;
  }

  .pdf-container.expanded {
    max-width: 1000px;
  }
}

@media screen and (max-width: 1200px) {
  .pdf-container {
    max-width: 700px;
  }

  .pdf-container.expanded {
    max-width: 900px;
  }
}

@media screen and (max-width: 992px) {
  .pdf-container {
    max-width: 600px;
  }

  .pdf-container.expanded {
    max-width: 800px;
  }
}

.pdf-wrap {
  overflow-y: hidden;
  background: #e5e5e5;
  height: 100%;
  padding: 0;
}

.vue-pdf-embed__page {
  margin-bottom: 0;
  box-shadow: none;
}

@media print {

  /* 去掉每页的分页规则，避免插入额外的空白页 */
  .pdf-container {
    display: block;
  }

  /* 确保页面之间没有强制分页 */
  .pdf-container .pdf-page {
    page-break-before: auto;
    page-break-after: auto;
    break-inside: avoid;
  }

  /* 隐藏偶数页 */
  .pdf-container .pdf-page:nth-child(even) {
    display: none;
  }
}
</style>