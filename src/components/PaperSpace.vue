<template>
  <div ref="paperDiv"></div>
</template>

<script>
import { ref, onMounted } from 'vue';
import { dia } from '@joint/core';
import { settings } from '@/settings/settings.js';

export default {
  name: 'PaperSpace',
  props: {
    graph: {
      type: Object,
      required: true
    },
    namespace: {
      type: Object,
      required: true
    }
  },
  setup(props) {
    const paperDiv = ref(null);

    onMounted(() => {
      // 初始化画布
      const paper = new dia.Paper({
        el: paperDiv.value,
        model: props.graph,
        width: '100%',
        height: '100%',
        background: { color: '#F5F5F5' },
        cellViewNamespace: props.namespace
      });

      // 拖动事件
      let dragStartPosition = null;
      paper.on("blank:pointerdown", function (evt, x, y) {
        var scale = paper.scale();
        dragStartPosition = { x: x * scale.sx, y: y * scale.sy };
      });

      paper.on("cell:pointerup blank:pointerup", () => {
        dragStartPosition = null;
      });

      paperDiv.value.addEventListener('mousemove', (event) => {
        if (dragStartPosition != null) {
          paper.translate(
            event.offsetX - dragStartPosition.x,
            event.offsetY - dragStartPosition.y
          );
        }
      });

      // 缩放事件
      const onMouseWheel = (x, y, delta) => {
        const zoom = paper.scale().sx;
        const newZoom = Math.max(
          settings.ZOOM_MIN,
          Math.min(zoom + delta*settings.ZOOM_SPEED, settings.ZOOM_MAX)
        );
        if (zoom === newZoom) return;
        paper.scale(newZoom)

        // 由于 paper 中心点 (0,0) 在屏幕上的位置不变
        // 故此时指针位置对应的 paper 坐标变为
        const currentPointer = {
          x: x * zoom / newZoom,
          y: y * zoom / newZoom,
        };
        // 将指针位置的 paper 坐标移回到原来的位置
        const oldTranslate = paper.translate();
        paper.translate(
          oldTranslate.tx + (currentPointer.x - x),
          oldTranslate.ty + (currentPointer.y - y),
        );
      }
      paper.on('cell:mousewheel', (cellView, evt, x, y, delta) => {
        onMouseWheel(x, y, delta);
      });
      paper.on('blank:mousewheel', (evt, x, y, delta) => {
        onMouseWheel(x, y, delta);
      });
    });

    return {
      paperDiv,
    };
  }
}
</script>