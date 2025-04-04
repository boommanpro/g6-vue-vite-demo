<template>
  <div id="container" style="width: 100vw; height: 100vh; margin: 0; padding: 0;"></div>
</template>

<script setup>
import { onMounted } from 'vue';

import { Rect, Text } from '@antv/g';
import {
  Badge,
  BaseBehavior,
  BaseNode,
  CubicHorizontal,
  ExtensionCategory,
  Graph,
  GraphEvent,
  iconfont,
  idOf,
  NodeEvent,
  positionOf,
  register,
  treeToGraphData
} from '@antv/g6';

import { firstLevelColor, secondLevelColor } from './mindmap';

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
    cursor: 'pointer',
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
  // 用于测量文本宽度的变量
  let textShape;
  /**
   * 测量文本的宽度
   * @param {Object} text - 文本的样式对象
   * @returns {number} - 文本的宽度
   */
  const measureText = (text) => {
    if (!textShape) textShape = new Text({ style: text });
    textShape.attr(text);
    return textShape.getBBox().width;
  };
  /**
   * 获取节点的宽度
   * @param {string} nodeId - 节点的ID
   * @param {boolean} isRoot - 是否为根节点
   * @returns {number} - 节点的宽度
   */
  const getNodeWidth = (nodeId, isRoot) => {
    const padding = isRoot ? 40 : 30;
    const nodeStyle = isRoot ? RootNodeStyle : NodeStyle;
    return measureText({ text: nodeId, fontSize: nodeStyle.labelFontSize, fontFamily: 'Gill Sans' }) + padding;
  };
  /**
   * 获取节点的大小
   * @param {string} nodeId - 节点的ID
   * @param {boolean} isRoot - 是否为根节点
   * @returns {Array<number>} - 节点的宽度和高度
   */
  const getNodeSize = (nodeId, isRoot) => {
    const width = getNodeWidth(nodeId, isRoot);
    const height = isRoot ? 48 : 32;
    return [width, height];
  };
  /**
   * 思维导图节点类
   */
  class MindmapNode extends BaseNode {
    // 默认样式属性
    static defaultStyleProps = {
      showIcon: true,
    };
    /**
     * 构造函数
     * @param {Object} options - 节点的配置选项
     */
    constructor(options) {
      Object.assign(options.style, MindmapNode.defaultStyleProps);
      super(options);
    }
    /**
     * 获取子节点数据
     * @returns {Array<Object>} - 子节点数据数组
     */
    get childrenData() {
      return this.context.model.getChildrenData(this.id);
    }
    /**
     * 获取根节点的ID
     * @returns {string} - 根节点的ID
     */
    get rootId() {
      return idOf(this.context.model.getRootsData()[0]);
    }
    /**
     * 判断是否显示折叠按钮
     * @param {Object} data - 节点的数据
     * @returns {boolean} - 是否显示折叠按钮
     */
    isShowCollapse(data) {
      const { collapsed } = data;
      // 如果是根节点，则不显示折叠按钮
      if (this.id === this.rootId) return false;
      return !collapsed && this.childrenData?.length > 0;
    }
    /**
     * 获取折叠按钮的样式
     * @param {Object} data - 节点的数据
     * @returns {Object|boolean} - 折叠按钮的样式对象，如果不显示则返回false
     */
    getCollapseStyle(data) {
      const { showIcon, color, direction } = data;
      if (!this.isShowCollapse(data)) return false;
      const [width, height] = this.getSize(data);
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
    /**
     * 绘制折叠按钮形状
     * @param {Object} data - 节点的数据
     * @param {Object} container - 容器对象
     */
    drawCollapseShape(data, container) {
      const iconStyle = this.getCollapseStyle(data);
      this.upsert('collapse-expand', Badge, iconStyle, container);
    }
    /**
     * 获取计数按钮的样式
     * @param {Object} data - 节点的数据
     * @returns {Object|boolean} - 计数按钮的样式对象，如果不显示则返回false
     */
    getCountStyle(data) {
      const { collapsed, color, direction } = data;
      const count = this.context.model.getDescendantsData(this.id).length;
      if (!collapsed || count === 0) return false;
      const [width, height] = this.getSize(data);
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
    /**
     * 绘制计数按钮形状
     * @param {Object} data - 节点的数据
     * @param {Object} container - 容器对象
     */
    drawCountShape(data, container) {
      const { collapsed } = data;
      if (collapsed) {
        const countStyle = this.getCountStyle(data);
        this.upsert('collapse-expand', Badge, countStyle, container);
      }
    }
    /**
     * 转发事件
     * @param {Object} target - 目标对象
     * @param {string} type - 事件类型
     * @param {Function} listener - 事件监听器
     */
    forwardEvent(target, type, listener) {
      if (target && !Reflect.has(target, '__bind__')) {
        Reflect.set(target, '__bind__', true);
        target.addEventListener(type, listener);
      }
    }
    /**
     * 获取关键形状的样式
     * @param {Object} data - 节点的数据
     * @returns {Object} - 关键形状的样式对象
     */
    getKeyStyle(data) {
      const [width, height] = this.getSize(data);
      const keyShape = super.getKeyStyle(data);
      return { width, height, ...keyShape };
    }
    /**
     * 绘制关键形状
     * @param {Object} data - 节点的数据
     * @param {Object} container - 容器对象
     * @returns {Object} - 绘制的关键形状对象
     */
    drawKeyShape(data, container) {
      const keyStyle = this.getKeyStyle(data);
      return this.upsert('key', Rect, keyStyle, container);
    }
    /**
     * 渲染节点
     * @param {Object} data - 节点的数据
     * @param {Object} container - 容器对象
     */
    render(data = this.parsedAttributes, container = this) {
      super.render(data, container);
      // 确保数据加载完成后再渲染折叠按钮
      this.drawCollapseShape(data, container);
      this.drawCountShape(data, container);
    }
  }
  /**
   * 思维导图边类
   */
  class MindmapEdge extends CubicHorizontal {
    /**
     * 获取根节点的ID
     * @returns {string} - 根节点的ID
     */
    get rootId() {
      return idOf(this.context.model.getRootsData()[0]);
    }
    /**
     * 获取关键路径
     * @param {Object} data - 边的数据
     * @returns {Array<Array<number>>} - 关键路径数组
     */
    getKeyPath(data) {
      const path = super.getKeyPath(data);
      const isRoot = this.targetNode.id === this.rootId;
      const labelWidth = getNodeWidth(this.targetNode.id, isRoot);
      const [, tp] = this.getEndpoints(data);
      const sign = this.sourceNode.getCenter()[0] < this.targetNode.getCenter()[0] ? 1 : -1;
      return [...path, ['L', tp[0] + labelWidth * sign, tp[1]]];
    }
  }
  /**
   * 折叠展开树行为类
   */
  class CollapseExpandTree extends BaseBehavior {
    /**
     * 构造函数
     * @param {Object} context - 上下文对象
     * @param {Object} options - 配置选项
     */
    constructor(context, options) {
      super(context, options);
      this.bindEvents();
    }
    /**
     * 更新配置选项
     * @param {Object} options - 配置选项
     */
    update(options) {
      this.unbindEvents();
      super.update(options);
      this.bindEvents();
    }
    /**
     * 绑定事件
     */
    bindEvents() {
      let graph = this.context.graph;
      this.context.graph.on(NodeEvent.CLICK, (e) => {
        let id = e.target.id;
        // 如果是root节点，则不执行折叠操作
        if (id === this.context.model.getRootsData()[0].id) return;
        const nodeData = graph.getNodeData(id);
        const { collapsed } = nodeData;
        graph.emit(TreeEvent.COLLAPSE_EXPAND, {
          id,
          collapsed: !collapsed,
        });
      });
      this.context.graph.on(TreeEvent.COLLAPSE_EXPAND, this.onCollapseExpand);
    }
    /**
     * 解绑事件
     */
    unbindEvents() {
      this.context.graph.off(TreeEvent.COLLAPSE_EXPAND, this.onCollapseExpand);
    }
    /**
     * 处理折叠展开事件
     * @param {Object} event - 事件对象
     */
    onCollapseExpand = async (event) => {
      const { id, collapsed } = event;
      const { graph } = this.context;
      // 获取collapsed的状态
      const nodeData = graph.getNodeData(id);
      nodeData.collapsed = collapsed;
      await graph.frontElement(id);
      if (collapsed) await graph.collapseElement(id);
      else await graph.expandElement(id);
    };
  }
  // 注册思维导图节点
  register(ExtensionCategory.NODE, 'mindmap', MindmapNode);
  // 注册思维导图边
  register(ExtensionCategory.EDGE, 'mindmap', MindmapEdge);
  // 注册折叠展开树行为
  register(ExtensionCategory.BEHAVIOR, 'collapse-expand-tree', CollapseExpandTree);
  /**
   * 获取节点的侧边位置
   * @param {Object} nodeData - 节点的数据
   * @param {Object} parentData - 父节点的数据
   * @returns {string} - 节点的侧边位置（left或right）
   */
  const getNodeSide = (nodeData, parentData) => {
    if (!parentData) return 'center';
    const nodePositionX = positionOf(nodeData)[0];
    const parentPositionX = positionOf(parentData)[0];
    return parentPositionX > nodePositionX ? 'left' : 'right';
  };
  // 从data.json文件中获取数据
  fetch('./data.json')
    .then((res) => res.json())
    .then((data) => {
      let grapthData = treeToGraphData(data);
      let cnt = 0;
      // 遍历节点数据，设置节点样式
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
      // 创建图对象
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
              // 通过设置节点标签背景来扩大交互区域 | Expand the interaction area by setting the node label background
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
      // 图渲染完成后，调整视图以适应窗口
      graph.once(GraphEvent.AFTER_RENDER, () => {
        graph.fitView();
      });
      // 渲染图
      graph.render();
    });
});
</script>

<style></style>