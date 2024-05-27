<template>
  <div class="pdf-vue3-main" style="height: 100%; position: relative; min-height: 10px">
    <div class="pdf-vue3-container" style="height: 100%">
      <div ref="scroller" class="pdf-vue3-scroller" style="height: 100%; overflow-y: auto"
        :style="{ maxHeight: `${viewportHeight}px` }" @scroll="handleScroll">
        <div class="pdf-vue3-canvas-container" ref="container" style="margin: 0 auto; width: 100%; position: relative;">
          <canvas style="display: block; margin-left: auto; margin-right: auto; width: 100%;" v-for="item in totalPages"
            :key="item" :ref="canvasRefs[item - 1]"></canvas>
        </div>
      </div>
    </div>
    <div class="pdf-vue3-progress"
      style="position: absolute; left: 0; top: 0; width: 100%; user-select: none; pointer-events: none;">
      <div style="width: 0%; height: 4px; transition: all 0.2s"
        :style="{ width: `${loadRatio}%`, opacity: loadRatio < 100 ? '1' : '0', backgroundColor: '#87ceeb' }">
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onUnmounted, Ref, computed, watch, onBeforeMount } from "vue";
import type { PDFDocumentProxy } from "./index";

let GlobalWorkerOptions: any, getDocument: any;
const dpr = ref(1);
const linkAnnotations = ref(['test', 'test2']);


const props = withDefaults(
  defineProps<{
    src: string | Uint8Array;
    httpHeaders?: Record<string, any>;
    withCredentials?: boolean;
    password?: string;
    useSystemFonts?: boolean;
    stopAtErrors?: boolean;
    disableFontFace?: boolean;
    disableRange?: boolean;
    disableStream?: boolean;
    disableAutoFetch?: boolean;
    // --custom--
    scrollThreshold?: number;
    page?: number;
    cMapUrl?: string;
  }>(),
  {
    src: undefined,
    httpHeaders: undefined,
    withCredentials: undefined,
    password: undefined,
    useSystemFonts: undefined,
    stopAtErrors: undefined,
    disableFontFace: undefined,
    disableRange: undefined,
    disableStream: undefined,
    disableAutoFetch: undefined,
    scrollThreshold: 300,
    page: 1,
    cMapUrl: "https://unpkg.com/pdfjs-dist@3.7.107/cmaps/",
  }
);

const emit = defineEmits<{
  (e: "onProgress", loadRatio: number): void;
  (e: "onComplete"): void;
  (e: "onScroll", scrollOffset: number): void;
  (e: "onPageChange", page: number): void;
  (e: "onPdfInit", pdf: PDFDocumentProxy): void;
}>();

const canvasRefs = ref<Array<Ref<Array<HTMLCanvasElement>>>>([]);

interface Option extends Record<string, any> {
  url?: string;
  data?: Uint8Array;
  httpHeaders?: Record<string, any>;
  withCredentials?: boolean;
  password?: string;
  useSystemFonts?: boolean;
  stopAtErrors?: boolean;
  disableFontFace?: boolean;
  disableRange?: boolean;
  disableStream?: boolean;
  disableAutoFetch?: boolean;
}

const loadRatio = ref(0);
const loadingTask = ref<any>(null);
const getDoc = () => {
  const option: Option = {
    httpHeaders: props.httpHeaders,
    withCredentials: props.withCredentials,
    password: props.password,
    useSystemFonts: props.useSystemFonts,
    stopAtErrors: props.stopAtErrors,
    disableFontFace: props.disableFontFace,
    disableRange: props.disableRange,
    disableStream: props.disableStream,
    disableAutoFetch: props.disableAutoFetch,
    cMapUrl: props.cMapUrl,
  };
  if (props.src instanceof Uint8Array) {
    option.data = props.src;
  } else if (props.src.endsWith(".pdf")) {
    option.url = props.src;
  } else {
    const binaryData = atob(
      props.src.includes(",") ? props.src.split(",")[1] : props.src
    );
    const byteArray = new Uint8Array(binaryData.length);
    for (let i = 0; i < binaryData.length; i++) {
      byteArray[i] = binaryData.charCodeAt(i);
    }
    option.data = byteArray;
  }

  for (const key in option) {
    if (option[key] === undefined) {
      delete option[key];
    }
  }
  loadRatio.value = 0;
  loadingTask.value = getDocument(option);
  loadingTask.value.onProgress = (progressData: any) => {
    const ratio = (progressData.loaded / progressData.total) * 100;
    loadRatio.value = ratio >= 100 ? 100 : ratio;
    emit("onProgress", loadRatio.value);
  };
  loadingTask.value.promise.then(() => {
    emit("onComplete");
  });
};

const totalPages = ref(0);
const currentPage = ref(1);
const scrollOffset = ref(0);
const itemHeightList = ref<Array<number>>([]);

const scroller = ref<HTMLDivElement>() as Ref<HTMLDivElement>;
const container = ref<HTMLDivElement>() as Ref<HTMLDivElement>;

let pdf: PDFDocumentProxy;
const cancelRendering = ref(false);
const renderComplete = ref(false);

const extractHyperlinksFromPage = async (page) => {
  const annotations = await page.getAnnotations();
  console.log(annotations)
  const hyperlinks = annotations
    .filter((annotation) => annotation.subtype === 'Link')
    .map((annotation) => {
      return {
        url: annotation.url || annotation.dest || 'No Text',
        boundingBox: annotation.rect, // Bounding box of the linked element
      };
    });

  return hyperlinks;
};
const renderPDF = async () => {
  renderComplete.value = false;
  try {
    if (!pdf) {
      pdf = await loadingTask.value.promise;
      const refs = [];
      for (let i = 0; i < pdf.numPages; i++) {
        refs.push(ref() as Ref<Array<HTMLCanvasElement>>);
      }
      canvasRefs.value = refs;
      totalPages.value = pdf.numPages;
      emit("onPdfInit", pdf);
    }
  } catch (error) {
    console.error("Error loadingTask PDF:", error);
  }
  let calcH = 0;
  for (let i = 0; i < totalPages.value; i++) {
    try {
      const page = await pdf.getPage(i + 1);

      // Render the page content onto the canvas
      const canvas = canvasRefs.value[i].value[0];
      var viewport = page.getViewport({ scale: 1 });
      var scale =
        ((canvas.parentNode as HTMLDivElement).clientWidth - 4) /
        viewport.width;
      const context = canvas.getContext("2d");
      const scaledViewport = page.getViewport({ scale: scale * dpr.value });
      canvas.width = scaledViewport.width;
      canvas.height = scaledViewport.height;
      itemHeightList.value[i] = calcH +=
        scaledViewport.height / dpr.value;
      await page.render({
        canvasContext: context as CanvasRenderingContext2D,
        viewport: scaledViewport,
      });

      // Render hyperlinks on the canvas container
      const hyperlinks = await extractHyperlinksFromPage(page);
hyperlinks.forEach((link) => {
  const { url, boundingBox } = link;
  // Ensure URL is properly encoded
  const encodedUrl = encodeURI(url);
  const linkElement = document.createElement('a');
  // Correction for web use: check if the URL is relative or absolute
  linkElement.href = `#${encodedUrl}`;
  linkElement.textContent = url; // Or any other text you want to display
  linkElement.style.position = 'absolute';
  linkElement.style.left = boundingBox[0] + 'px'; // Adjust positioning based on bounding box
  linkElement.style.top = boundingBox[1] + 'px'; // Adjust positioning based on bounding box
  linkElement.style.width = boundingBox[2] - boundingBox[0] + 'px'; // Adjust width based on bounding box
  linkElement.style.height = boundingBox[3] - boundingBox[1] + 'px'; // Adjust height based on bounding box
  // Apply styles to make it look like a hyperlink
  linkElement.style.color = 'blue';
  linkElement.style.textDecoration = 'underline';
  linkElement.style.cursor = 'pointer';
  // Append the link to the canvas container
  canvas.parentNode.appendChild(linkElement);
});

    } catch (error) {
      console.error("Error rendering PDF:", error);
    }
    if (
      props.page &&
      (i === props.page - 1 ||
        (props.page > totalPages.value && i === totalPages.value - 1))
    ) {
      scroller.value.scrollTo(0, (itemHeightList.value[i - 1] ?? 0) + 2);
    }
    if (i === totalPages.value - 1) {
      renderComplete.value = true;
    }
  }
};


const viewportHeight = ref(0);
const isScrolling = ref(false);

let scrollTimer: number;
const handleScroll = (event: any) => {
  isScrolling.value = true;
  clearTimeout(scrollTimer);
  scrollTimer = setTimeout(() => {
    isScrolling.value = false;
  }, 1000);
  scrollOffset.value = event.target.scrollTop;
  emit("onScroll", event.target.scrollTop);
  if (
    scroller.value.scrollTop + scroller.value.offsetHeight >=
    scroller.value.scrollHeight - 10
  ) {
    currentPage.value = itemHeightList.value.length;
    return;
  }

  for (let i = 0; i < itemHeightList.value.length; i++) {
    const height = itemHeightList.value[i];
    if (height > event.target.scrollTop) {
      currentPage.value = i + 1;
      break;
    }
  }
};

let timer: number;
const renderPDFWithDebounce = () => {
  viewportHeight.value = window.innerHeight;
  if (
    Math.abs(innerWidth.value - window.innerWidth) > 1 &&
    Math.abs(containerWidth.value - container.value.offsetWidth) > 1
  ) {
    setWidth();
  } else {
    setWidth();
    return;
  }
  cancelRendering.value = true;
  clearTimeout(timer);
  timer = setTimeout(() => {
    renderComplete.value && renderPDF();
  }, 500);
};

const innerWidth = ref<number>(0);
const containerWidth = ref<number>(0);
const setWidth = () => {
  innerWidth.value = window.innerWidth;
  containerWidth.value = container.value.offsetWidth;
};
const isAddEvent = ref(false);
onBeforeMount(async () => {
  const pdfjs = await import("pdfjs-dist/legacy/build/pdf.min.js");
  GlobalWorkerOptions = pdfjs.GlobalWorkerOptions;
  getDocument = pdfjs.getDocument;
  const workerSrc = new URL(
    "../../node_modules/pdfjs-dist/legacy/build/pdf.worker.min.js",
    import.meta.url
  ).href;
  GlobalWorkerOptions.workerSrc = workerSrc;
  dpr.value = window.devicePixelRatio || 1;
  viewportHeight.value = window.innerHeight;
  setWidth();
  if (
    (typeof props.src === "string" && props.src.length > 0) ||
    props.src instanceof Uint8Array
  ) {
    getDoc();
    renderPDF();
    window.addEventListener("resize", renderPDFWithDebounce);
    isAddEvent.value = true;
  }

  watch(
    () => props.src,
    (src: string | Uint8Array) => {
      if (
        (typeof src === "string" && src.length > 0) ||
        src instanceof Uint8Array
      ) {
        getDoc();
        renderPDF();
        if (!isAddEvent.value) {
          window.addEventListener("resize", renderPDFWithDebounce);
          isAddEvent.value = true;
        }
      }
    }
  );
});

defineExpose({
  reload: () => {
    innerWidth.value = window.innerWidth - 2;
    renderPDFWithDebounce();
    setWidth();
  },
});

onUnmounted(() => {
  clearTimeout(timer);
  clearTimeout(scrollTimer);
  cancelAnimationFrame(animFrameId);
  isAddEvent.value &&
    window.removeEventListener("resize", renderPDFWithDebounce);
});
// --- back to top ---
let animFrameId: number;
const easeOutCubic = (progress: number) => {
  return 1 - Math.pow(1 - progress, 3);
};

let waitToPageFun: Function | null = null;
watch(
  () => props.page,
  (page: number) => {
    if (props.page === currentPage.value) {
      return;
    }
    if (page > itemHeightList.value.length) {
      page = itemHeightList.value.length;
    }
    if (renderComplete.value) {
      scroller.value.scrollTo(0, (itemHeightList.value[page - 2] ?? 0) + 2);
    } else {
      waitToPageFun = () => {
        scroller.value.scrollTo(0, (itemHeightList.value[page - 2] ?? 0) + 2);
      };
    }
  }
);

 
watch(
  () => renderComplete.value,
  (complete: boolean) => {
    complete && waitToPageFun?.();
    waitToPageFun = null;
  }
);
watch(
  () => currentPage.value,
  (page: number) => {
    emit("onPageChange", page);
  }
);
</script>



<style>
.pdf-vue3-scroller::-webkit-scrollbar {
  display: none;  
}
</style>
