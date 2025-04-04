<template>
  <div id="container"></div>
</template>

<script setup>
import { onMounted } from 'vue';

import { Rect, Text } from '@antv/g';
import {
  Badge,
  BaseBehavior,
  BaseNode,
  BaseTransform,
  CommonEvent,
  CubicHorizontal,
  ExtensionCategory,
  Graph,
  GraphEvent,
  iconfont,
  idOf,
  NodeEvent,
  positionOf,
  register,
  treeToGraphData,
} from '@antv/g6';

const firstLevelColor = [
  '#1783FF',
  '#F08F56',
  '#D580FF',
  '#00C9C9',
  '#7863FF',
  '#DB9D0D',
  '#60C42D',
  '#FF80CA',
  '#2491B3',
  '#17C76F',
];
const secondLevelColor = [
  '#F08F56',
  '#D580FF',
  '#00C9C9',
  '#7863FF',
  '#DB9D0D',
  '#60C42D',
  '#FF80CA',
  '#2491B3',
  '#17C76F',
  '#1783FF',

]

onMounted(() => {

  const style = document.createElement('style');
  style.innerHTML = `@import url('${iconfont.css}');`;
  document.head.appendChild(style);

  const RootNodeStyle = {
    fill: '#020409',
    labelFill: '#FFFFFF',
    labelFontSize: 24,
    labelFontWeight: 600,
    labelOffsetY: 8,
    labelPlacement: 'center',
    ports: [{ placement: 'right' }],
    radius: 8,
  };

  const NodeStyle = {
    labelOffsetY: 0,
    labelFill: '#262626',
    labelFontWeight: 600,
    labelOffsetY: 4,
    labelPlacement: 'center',
    ports: [{ placement: 'right' }],
    radius: 8,
    labelFontSize: 16,
    ports: [{ placement: 'right' }, { placement: 'left' }],
  };

  const TreeEvent = {
    COLLAPSE_EXPAND: 'collapse-expand',
  };

  let textShape;
  const measureText = (text) => {
    if (!textShape) textShape = new Text({ style: text });
    textShape.attr(text);
    return textShape.getBBox().width;
  };

  const getNodeWidth = (nodeId, isRoot) => {
    const padding = isRoot ? 40 : 30;
    const nodeStyle = isRoot ? RootNodeStyle : NodeStyle;
    return measureText({ text: nodeId, fontSize: nodeStyle.labelFontSize, fontFamily: 'Gill Sans' }) + padding;
  };

  const getNodeSize = (nodeId, isRoot) => {
    const width = getNodeWidth(nodeId, isRoot);
    const height = isRoot ? 48 : 32;
    return [width, height];
  };

  class MindmapNode extends BaseNode {
    static defaultStyleProps = {
      showIcon: true,
    };

    constructor(options) {
      Object.assign(options.style, MindmapNode.defaultStyleProps);
      super(options);
    }

    get childrenData() {
      return this.context.model.getChildrenData(this.id);
    }

    get rootId() {
      return idOf(this.context.model.getRootsData()[0]);
    }

    isShowCollapse(attributes) {
      const { collapsed } = attributes;
      // 添加判断条件，如果是根节点，则不显示折叠按钮
      if (this.id === this.rootId) return false;
      return !collapsed && this.childrenData?.length > 0;
    }

    getCollapseStyle(attributes) {
      // console.log("getCollapseStyle")
      const { showIcon, color, direction } = attributes;
      if (!this.isShowCollapse(attributes)) return false;
      const [width, height] = this.getSize(attributes);


      return {
        backgroundFill: color,
        backgroundHeight: 12,
        backgroundWidth: 12,
        cursor: 'pointer',
        fill: '#fff',
        fontFamily: 'iconfont',
        fontSize: 8,
        text: '\ue6e4',
        textAlign: 'center',
        transform: [['rotate', -90]],
        visibility: showIcon ? 'visible' : 'hidden',
        x: width + 6,
        y: height - 14,
      };
    }

    drawCollapseShape(attributes, container) {
      if (!attributes.collapsed) {
        const iconStyle = this.getCollapseStyle(attributes);
        const btn = this.upsert('collapse-expand', Badge, iconStyle, container);
        this.forwardEvent(btn, CommonEvent.CLICK, (event) => {
          event.stopPropagation();
          this.context.graph.emit(TreeEvent.COLLAPSE_EXPAND, {
            id: this.id,
            collapsed: !attributes.collapsed,
          });
        });
      }

    }

    getCountStyle(attributes) {
      const { collapsed, color, direction } = attributes;
      const count = this.context.model.getDescendantsData(this.id).length;
      if (!collapsed || count === 0) return false;
      const [width, height] = this.getSize(attributes);
      return {
        backgroundFill: color,
        backgroundHeight: 12,
        backgroundWidth: 12,
        cursor: 'pointer',
        fill: '#fff',
        fontSize: 8,
        text: count.toString(),
        textAlign: 'center',
        x: width + 6,
        y: height - 14,
        transform: [['rotate', 0]],
      };
    }

    drawCountShape(attributes, container) {
      if (attributes.collapsed) {
        const countStyle = this.getCountStyle(attributes);
        const btn = this.upsert('collapse-expand', Badge, countStyle, container);
        this.forwardEvent(btn, CommonEvent.CLICK, (event) => {
          event.stopPropagation();
          this.context.graph.emit(TreeEvent.COLLAPSE_EXPAND, {
            id: this.id,
            collapsed: false,
          });
        });
      }

    }


    forwardEvent(target, type, listener) {
      if (target && !Reflect.has(target, '__bind__')) {
        Reflect.set(target, '__bind__', true);
        target.addEventListener(type, listener);
      }
    }

    getKeyStyle(attributes) {
      const [width, height] = this.getSize(attributes);
      const keyShape = super.getKeyStyle(attributes);
      return { width, height, ...keyShape };
    }

    drawKeyShape(attributes, container) {
      const keyStyle = this.getKeyStyle(attributes);
      return this.upsert('key', Rect, keyStyle, container);
    }

    render(attributes = this.parsedAttributes, container = this) {
      console.log("badge render 开始", this.id);
      super.render(attributes, container);

      // 确保数据加载完成后再渲染折叠按钮
      this.drawCollapseShape(attributes, container);
      this.drawCountShape(attributes, container);
      console.log("badge render 结束", this.id);
    }
  }

  class MindmapEdge extends CubicHorizontal {
    get rootId() {
      return idOf(this.context.model.getRootsData()[0]);
    }

    getKeyPath(attributes) {
      const path = super.getKeyPath(attributes);
      const isRoot = this.targetNode.id === this.rootId;
      const labelWidth = getNodeWidth(this.targetNode.id, isRoot);

      const [, tp] = this.getEndpoints(attributes);
      const sign = this.sourceNode.getCenter()[0] < this.targetNode.getCenter()[0] ? 1 : -1;
      return [...path, ['L', tp[0] + labelWidth * sign, tp[1]]];
    }
  }

  class CollapseExpandTree extends BaseBehavior {
    constructor(context, options) {
      super(context, options);
      this.bindEvents();
    }

    update(options) {
      this.unbindEvents();
      super.update(options);
      this.bindEvents();
    }

    bindEvents() {
      this.context.graph.on(TreeEvent.COLLAPSE_EXPAND, this.onCollapseExpand);
    }

    unbindEvents() {
      this.context.graph.off(TreeEvent.COLLAPSE_EXPAND, this.onCollapseExpand);
    }




    onCollapseExpand = async (event) => {
      console.log("折叠事件开始", event);
      const { id, collapsed } = event;
      const { graph } = this.context;
      await graph.frontElement(id);
      if (collapsed) await graph.collapseElement(id);
      else await graph.expandElement(id);
      console.log("折叠事件结束", event);
    };
  }
  register(ExtensionCategory.NODE, 'mindmap', MindmapNode);
  register(ExtensionCategory.EDGE, 'mindmap', MindmapEdge);
  register(ExtensionCategory.BEHAVIOR, 'collapse-expand-tree', CollapseExpandTree);

  const getNodeSide = (nodeData, parentData) => {
    if (!parentData) return 'center';

    const nodePositionX = positionOf(nodeData)[0];
    const parentPositionX = positionOf(parentData)[0];
    return parentPositionX > nodePositionX ? 'left' : 'right';
  };

  fetch('./data.json')
    .then((res) => res.json())
    .then((data) => {
      let grapthData = treeToGraphData(data);
      //对grapthData进行预处理，根据层级增加color字段，
      // 第一层节点的color为firstLevelColor，第二层节点的color为secondLevelColor
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
      })

      const rootId = data.id;

      const graph = new Graph({
        data: grapthData,
        node: {
          type: 'mindmap',
          style: function (d) {
            const direction = getNodeSide(d, this.getParentData(idOf(d), 'tree'));
            const isRoot = idOf(d) === rootId;
            console.log(d)
            return {
              direction,
              labelText: idOf(d),
              size: getNodeSize(idOf(d), isRoot),
              labelFontFamily: 'Gill Sans',
              // 通过设置节点标签背景来扩大交互区域 | Expand the interaction area by setting the node label background
              labelBackground: true,
              labelBackgroundFill: 'transparent',
              labelPadding: [2, 40, 10, 0],
              ...(isRoot ? RootNodeStyle : NodeStyle),
              fill: d.style.color
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
