<script setup>
import { ref, watchEffect } from "vue";
import PDF from "./pdf/pdf-vue3.vue";

const isMobile = ref(false);
const page = ref(1);

const handlePageChange = (newPage) => {
  page.value = newPage;
};

const resize = () => {
  isMobile.value = window.innerWidth < 768;
};

watchEffect(() => {
  resize();
  window.addEventListener("resize", resize);
  return () => {
    window.removeEventListener("resize", resize);
  };
});

const handlePdfInit = (pdf) => {
  };
</script>

<template>
  <div style="width: 100%">
    <PDF
      :page="page"
      src="/present.pdf"
      @on-pdf-init="handlePdfInit"
      @on-page-change="handlePageChange"
    >
    </PDF>
  </div>
</template>

<style scoped>
</style>
