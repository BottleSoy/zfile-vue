<template>
  <div class="zfile-pdf-viewer">
    <div class="app-header">
      <div v-if="isLoading">
        <span v-if="loadingProgress < 100">
          加载 {{ loadingProgress }}%
        </span>
        <span v-else>
          渲染所有页面中...
        </span>
      </div>
      <div v-else>
        <span v-if="showAllPages">
          共 {{ pageCount }} 页
        </span>

        <span v-else>
          <button :disabled="page <= 1" @click="page--">❮</button>
          {{ page }} / {{ pageCount }}
          <button :disabled="page >= pageCount" @click="page++">❯</button>
        </span>
      </div>

      <span v-show="!isLoading">
        <button class="px-1 sm:px-4" @click="pdfViewCanvasWidthNum-=5">-</button>
        <span>
          <span class="hidden sm:inline">缩放比例</span>
          {{pdfViewCanvasWidthNum}} %
        </span>
        <button class="px-1 sm:px-4" @click="pdfViewCanvasWidthNum+=5">+</button>
      </span>

      <label :class="isLoading ? 'invisible' : 'visible'">
        <input v-model="showAllPages" type="checkbox">
        显示所有页
      </label>
    </div>

    <div class="app-content">
      <vue-pdf-embed
        annotationLayer
        textLayer
        ref="pdfRef"
        :source="doc"
        :page="page"
        @rendered="handleDocumentRender"
      />
    </div>
  </div>
</template>

<script setup>
import VuePdfEmbed, { useVuePdfEmbed } from 'vue-pdf-embed'
import { useMagicKeys } from '@vueuse/core'
const { ArrowLeft, ArrowRight, NumpadAdd, NumpadSubtract } = useMagicKeys()

// optional styles
import 'vue-pdf-embed/dist/styles/annotationLayer.css'
import 'vue-pdf-embed/dist/styles/textLayer.css'

// 组件接收的属性：
//  fileUrl:    文件下载路径
//  fileName:   文件名
const props = defineProps({
  fileUrl: String,
  fileName: String
});

const pdfRef = ref();

const pdfViewCanvasWidthNum = ref(100);
// 图片外部容器宽度, 100 / 图片列数
let pdfViewCanvasWidth = computed(() => {
  return `${pdfViewCanvasWidthNum.value}%`;
})

let isLoading = ref(true);
let page = ref(null);
let pageCount = ref(1);
let showAllPages = ref(true);

watch(() => showAllPages.value, () => {
  page.value = showAllPages.value ? null : 1;
});

const handleDocumentRender = (a) => {
  pageCount.value = pdfRef.value.doc.numPages;
  isLoading.value = false;
};

const handlerDocumentError = (error) => {
  isLoading.value = false;
  ElMessage.error("PDF 加载失败");
};

const loadingProgress = ref(0);
const handleDocumentLoadProgress = (progressData) => {
  // 保留2位小数
  loadingProgress.value = (progressData.loaded / progressData.total * 100).toFixed(2);
};

// 支持按键翻页
watch(() => [ArrowLeft.value, ArrowRight.value], (value) => {
  if (isLoading.value) return;
  if (showAllPages.value) return;
  if (value[0] && page.value > 1) {
    page.value--;
  }
  if (value[1] && page.value < pageCount.value) {
    page.value++;
  }
})

// 支持按键缩放
watch(() => [NumpadSubtract.value, NumpadAdd.value], (value) => {
  if (value[0]) {
    pdfViewCanvasWidthNum.value -= 5;
  }
  if (value[1]) {
    pdfViewCanvasWidthNum.value += 5;
  }
})

const { doc } = useVuePdfEmbed( {
  source: props.fileUrl,
  onProgress: handleDocumentLoadProgress,
  onError: handlerDocumentError,
})
</script>

<style scoped lang="scss">
.vue-pdf-embed {
  :deep(> div) {
    box-shadow: 0 2px 8px 4px rgba(0, 0, 0, 0.1);
    @apply w-full h-full mt-2;
  }

  :deep(canvas) {
    width: v-bind('pdfViewCanvasWidth') !important;
    height: 100% !important;
    margin: 0 auto;
  }
}

.app-header {
  box-shadow: 0 2px 8px 4px rgba(0, 0, 0, 0.1);
  background-color: #555;
  color: #ddd;
  @apply flex content-between justify-between p-2 sm:p-4 bg-gray-600;

}

.app-content {
  padding: 24px 0;
}
</style>