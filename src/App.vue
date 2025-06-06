<template>
  <div class="container">
    <PaperSpaceVue :graph="graph" :namespace="namespace" />
  </div>
</template>
<script>
import { onMounted } from 'vue';
import { dia, shapes } from '@joint/core';
import { DirectedGraph } from '@joint/layout-directed-graph';

// 导入本地版本信息
// import localVersionInfo from '@/assets/version.json';
import {
  loadGameDataIndex,
  loadAllEvents,
  loadEventData,
  getChildEventIds,
} from './services/eventService';
import PaperSpaceVue from './components/PaperSpace.vue';

export default {
  name: 'App',
  components: {
    PaperSpaceVue,
  },
  setup() {
    // const typeMap = new Map([
    //   ['event', '事件'],
    //   ['rite', '仪式'],
    //   ['loot', '战利品'],
    //   ['over', '结局'],
    //   ['after_story', '后日谈'],
    //   ['card', '卡牌'],
    // ]);

    // paper space
    const namespace = shapes;
    const graph = new dia.Graph({}, { cellNamespace: namespace });

    const drawSequenceLink = (source, target) => {
      const link = new namespace.standard.Link({
        source: {
          id: source.id,
          anchor: { name: 'right' },
          connectionPoint: { name: 'anchor' },
        },
        target: {
          id: target.id,
          anchor: { name: 'left' },
          connectionPoint: { name: 'anchor' },
        },
        connector: {
          name: 'straight',
          args: {
            cornerType: 'cubic',
            cornerRadius: 20,
          },
        },
        router: {
          name: 'manhattan',
          args: {
            startDirections: ['right'],
            endDirections: ['left'],
            padding: {
              bottom: 20,
              top: 50,
              right: 20,
              left: 20,
            },
          }
        },
      });
      link.addTo(graph);
    };

    const formatEventShape = (shape, eventData) => {
      let text;
      let type;
      if (eventData.name) {
        text = eventData.name;
        shape.attr('root/title', eventData.text);
        type = '仪式';
      } else {
        text = eventData.text || '[无标题]';
        type = '事件';
      }
      if (eventData.type === 2) {
        type = '战利品';
      }
      shape.size({ width: Math.max(90, 14 * text.length + 10) });
      shape.attr(
        'label/text',
        `${text}\n${type}: ${eventData.id}`);
    };

    /**
     * 
     * @param eventData 
     * @param baseShape 
     * @param {Map} visited 
     */
    const visualizeChildrenRecursively = async (eventData, baseShape, visited) => {
      const childrenID = getChildEventIds(eventData);
      if (childrenID.length === 0) return;

      if (visited === undefined) {
        visited = new Map();
        visited.set(eventData.id, baseShape);
      }

      let unvisitedChild = []
      for (const childID of childrenID) {
        if (childID === eventData.id) {
          continue;    // 不为自引用绘制链接
        }
        if (visited.has(childID)) {
          const childShape = visited.get(childID);
          drawSequenceLink(baseShape, childShape);
        } else {
          unvisitedChild.push(childID);
        }
      }

      for (const childID of unvisitedChild) {
        const childEvent = await loadEventData(childID);
        const childShape = new namespace.standard.Rectangle({
          size: { width: 150, height: 50 },
        });
        formatEventShape(childShape, childEvent);
        childShape.addTo(graph);
        drawSequenceLink(baseShape, childShape);
        visited.set(childEvent.id, childShape);
        await visualizeChildrenRecursively(childEvent, childShape, visited);
      }
    }
    /* // 添加本地版本号
    const localVersionText =  ref(JSON.parse(localVersionInfo)?.version || '0.0.0');

    const checkVersion = async () => {
      try {
        // Get remote version info
        const response = await fetch('https://raw.githubusercontent.com/liwenhao0427/sudans-game-reader/refs/heads/main/src/assets/version.json?t=' + new Date().getTime());
        const remoteData = await response.json();
        
        // Get local version - ensure it exists
        const localVersion = JSON.parse(localVersionInfo)?.version || '0.0.0';
        const remoteVersion = remoteData?.version || '0.0.0';
        
        // Compare versions
        const isNewer = compareVersions(remoteVersion, localVersion) > 0;
        
        if (isNewer) {
          if (window.confirm(`发现新版本 ${remoteVersion}, 当前版本 ${localVersion}. 是否刷新页面更新到最新版本?`)) {
            //  // 添加时间戳作为缓存破坏参数
            //  const newUrl = "https://sudans-game-reader.pages.dev/?t=" + new Date().getTime();
            // window.location.href = newUrl;
            window.location.reload(true);
          } else {
            console.log('User chose to update later');
          }
        } else {
          console.log('Already on the latest version');
        }
      } catch (error) {
        console.error('Failed to check for updates:', error);
      }
    };
    
    // 版本号比较函数
    const compareVersions = (v1, v2) => {
      // Handle undefined or null values
      if (!v1) return -1; // If v1 is undefined/null, consider it older
      if (!v2) return 1;  // If v2 is undefined/null, consider it older
      
      const parts1 = v1.split('.').map(Number);
      const parts2 = v2.split('.').map(Number);
      
      for (let i = 0; i < Math.max(parts1.length, parts2.length); i++) {
        const part1 = parts1[i] || 0;
        const part2 = parts2[i] || 0;
        
        if (part1 > part2) return 1;
        if (part1 < part2) return -1;
      }
      
      return 0; // Versions are the same
    };
    */

    onMounted(async () => {
      let allEvents;
      try {
        // 预加载索引文件
        await loadGameDataIndex()
        allEvents = await loadAllEvents()

        // 检查版本更新
        // checkVersion();
      } catch (e) {
        console.error('初始化数据失败:', e);
      }
      console.log('allEvents:', allEvents);

      const rootEvent = await loadEventData('5300039', 'event');
      console.log('rootEvent:', rootEvent);

      new shapes.standard.Rectangle({
        size: { width: 150, height: 50 },
        position: { x: 25, y: 25 },
        attrs: {
          label: {
            text: `数据加载完成\n${allEvents.length} 条`,
          },
        },
      }).addTo(graph);

      const rectRoot = new shapes.standard.Rectangle({
        size: { width: 100, height: 50 },
      });
      formatEventShape(rectRoot, rootEvent);
      rectRoot.addTo(graph);

      await visualizeChildrenRecursively(rootEvent, rectRoot);
      DirectedGraph.layout(graph, {
        nodeSep: 50,
        edgeSep: 50,
        rankSep: 60,
        rankDir: "LR",
      });
    });

    return {
      graph,
      namespace,
      PaperSpaceVue,
      // checkVersion,
    };
  }
}
</script>
<style scoped>
.container {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  display: block;
  position: fixed;
  top: 0;
  left: 0;
}
</style>
