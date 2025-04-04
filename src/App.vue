<template>
  <div id="container" style="width: 100vw; height: 100vh; margin: 0; padding: 0;"></div>
</template>

<script setup>
import { onMounted } from 'vue';

import {
  Graph,
  GraphEvent,
  iconfont,
  idOf,
  treeToGraphData
} from '@antv/g6';

import { firstLevelColor, getNodeSide, getNodeSize, getNodeWidth, NodeStyle, RootNodeStyle, secondLevelColor } from './mindmap';

onMounted(() => {
  const style = document.createElement('style');
  style.innerHTML = `@import url('${iconfont.css}');`;
  document.head.appendChild(style);


  // 删除配置项相关代码
  fetch('./data.json')
    .then((res) => res.json())
    .then((data) => {
      let grapthData = treeToGraphData(data);
      let cnt = 0;
      grapthData.nodes.forEach((node) => {
        if (node.depth === 0) {
          node.style = { color: RootNodeStyle.fill };
        }
        if (node.depth === 1) {
          node.style = { color: firstLevelColor[cnt % firstLevelColor.length] };
          node.children.forEach((childId) => {
            const childNode = grapthData.nodes.find((node) => node.id === childId);
            childNode.style = { color: secondLevelColor[cnt % secondLevelColor.length] };
          });
          cnt++;
        } else {
          node.style = { color: firstLevelColor[cnt % firstLevelColor.length] };
        }
        node.collapsed = false;
      });
      const rootId = data.id;
      const graph = new Graph({
        data: grapthData,
        node: {
          type: 'mindmap',
          style: function (d) {
            const direction = getNodeSide(d, this.getParentData(idOf(d), 'tree'));
            const isRoot = idOf(d) === rootId;
            return {
              direction,
              labelText: idOf(d),
              size: getNodeSize(idOf(d), isRoot),
              labelFontFamily: 'Gill Sans',
              labelBackground: true,
              labelBackgroundFill: 'transparent',
              labelPadding: [2, 40, 10, 0],
              ...(isRoot ? RootNodeStyle : NodeStyle),
              fill: d.style.color,
            };
          },
        },
        edge: {
          type: 'mindmap',
          style: {
            lineWidth: 3,
            stroke: function (data) {
              return this.getNodeData(data.target).style.color || '#99ADD1';
            },
          },
        },
        layout: {
          type: 'mindmap',
          direction: 'LR',
          getHeight: () => 30,
          getWidth: (node) => getNodeWidth(node.id, node.id === rootId),
          getVGap: () => 6,
          getHGap: () => 60,
          animation: false,
        },
        behaviors: ['drag-canvas', 'zoom-canvas', 'collapse-expand-tree'],
        animation: false,
      });
      graph.once(GraphEvent.AFTER_RENDER, () => {
        graph.fitView();
      });
      graph.render();
    });
});
</script>

<style></style>