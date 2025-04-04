<template>
  <div id="container" style="width: 100vw; height: 100vh; margin: 0; padding: 0;"></div>
</template>

<script setup>
import { onMounted } from 'vue';

import {
  iconfont
} from '@antv/g6';

import { processAndRenderGraph } from './mindmap';

const fetchData = () => {
  return fetch('./data.json')
    .then((res) => res.json());
};

onMounted(() => {
  const style = document.createElement('style');
  style.innerHTML = `@import url('${iconfont.css}');`;
  document.head.appendChild(style);

  fetchData()
    .then((data) => {
      processAndRenderGraph(data);
    });
});
</script>

<style></style>