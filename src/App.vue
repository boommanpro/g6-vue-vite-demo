<template>
  <div id="container" style="width: 100vw; height: 100vh; margin: 0; padding: 0;"></div>
</template>

<script setup>
import { onMounted } from 'vue';

import {
  iconfont
} from '@antv/g6';

import { fetchData, processAndRenderGraph } from './mindmap';

onMounted(() => {
  const style = document.createElement('style');
  style.innerHTML = `@import url('${iconfont.css}');`;
  document.head.appendChild(style);

  fetchData()
    .then(() => {
      processAndRenderGraph();
    });
});
</script>

<style></style>