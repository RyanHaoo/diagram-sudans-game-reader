<template>
  <div class="container">
    <PaperSpaceVue :graph="graph" :namespace="namespace" />
  </div>
</template>

<script>
import { onMounted } from 'vue';
import { dia, shapes } from '@joint/core';

// 导入本地版本信息
// import localVersionInfo from '@/assets/version.json';
import { 
  loadGameDataIndex,
  loadAllEvents,
} from './services/eventService';
import PaperSpaceVue from './components/PaperSpace.vue';

export default {
  name: 'App',
  components: {
    PaperSpaceVue,
  },
  setup() {
    // const getTypeLabel = (type) => {
    //   const typeMap = {
    //     'event': '事件',
    //     'rite': '仪式',
    //     'loot': '战利品',
    //     'over': '结局',
    //     'after_story': '后日谈',
    //     'card': '卡牌'
    //   };
    //   return typeMap[type] || type;
    // };

    // paper space
    const namespace = shapes;
    const graph = new dia.Graph({}, { cellNamespace: namespace });

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

      const rect1 = new shapes.standard.Rectangle();
      rect1.position(25, 25);
      rect1.resize(180, 50);
      rect1.addTo(graph);

      const rect2 = new shapes.standard.Rectangle();
      rect2.position(95, 225);
      rect2.resize(180, 50);
      rect2.addTo(graph);

      rect1.attr('label', { text: 'Hello!', fill: '#353535' });
      rect2.attr('label', { text: allEvents.length, fill: '#353535' });
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
