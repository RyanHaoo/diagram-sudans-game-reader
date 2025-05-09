<template>
  <div class="container">
    <div id="paper"></div>
  </div>
</template>

<script>
import { ref, shallowRef, onMounted } from 'vue';
import { dia, shapes } from '@joint/core';

// 导入本地版本信息
// import localVersionInfo from '@/assets/version.json';
import { 
  loadCardsData,
  loadGameDataIndex,
  getEventsByType,
  loadOversData,
  getAfterStories
} from './services/eventService';

export default {
  name: 'App',
  components: {
  },
  setup() {
    const getTypeLabel = (type) => {
      const typeMap = {
        'event': '事件',
        'rite': '仪式',
        'loot': '战利品',
        'over': '结局',
        'after_story': '后日谈',
        'card': '卡牌'
      };
      return typeMap[type] || type;
    };
    
    // 搜索和分页相关
    const allEvents = shallowRef([]);
    const allEventsCache = shallowRef({}); // 缓存所有类型的数据
    const isLoading = ref(false);
    
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

    // 一次性加载所有事件数据
    const loadAllEvents = async () => {
      if (isLoading.value) return;
      
      isLoading.value = true;
      try {
        // 加载所有类型的数据
        const types = ['event', 'rite', 'loot'];
        let combinedEvents = [];
        
        // 对每种类型加载数据（使用缓存）
        for (const type of types) {
          if (!allEventsCache.value[type]) {
            // 如果缓存中没有该类型的数据，则加载
            const countResult = await getEventsByType(type, 1, 1);
            const allResult = await getEventsByType(type, 1, countResult.total);
            allEventsCache.value[type] = allResult.items;
          }
          
          // 从缓存中获取数据
          combinedEvents = [...combinedEvents, ...allEventsCache.value[type]];
        }
        
        // 加载结局数据
        if (!allEventsCache.value['over']) {
          try {
            const oversData = await loadOversData();
            const overItems = Object.entries(oversData).map(([id, data]) => ({
              id,
              name: data.name || data.text || `结局 ${id}`,
              text: data.text || '',
              type: 'over'
            }));
            allEventsCache.value['over'] = overItems;
            combinedEvents = [...combinedEvents, ...overItems];
          } catch (e) {
            console.error('加载结局数据失败:', e);
          }
        } else {
          combinedEvents = [...combinedEvents, ...allEventsCache.value['over']];
        }
        
        // 加载卡片数据
        if (!allEventsCache.value['card']) {
          try {
            const cardsData = await loadCardsData();
            const cardItems = Object.entries(cardsData).map(([id, data]) => ({
              id,
              name: data.name || data.text || `卡片 ${id}`,
              text: data.text || '',
              type: 'card'
            }));
            allEventsCache.value['card'] = cardItems;
            combinedEvents = [...combinedEvents, ...cardItems];
          } catch (e) {
            console.error('加载卡片数据失败:', e);
          }
        } else {
          combinedEvents = [...combinedEvents, ...allEventsCache.value['card']];
        }
        
        // 加载后日谈数据
        if (!allEventsCache.value['after_story']) {
          try {
            const afterStories = await getAfterStories();
            allEventsCache.value['after_story'] = afterStories;
            combinedEvents = [...combinedEvents, ...afterStories];
          } catch (e) {
            console.error('加载后日谈数据失败:', e);
          }
        } else {
          combinedEvents = [...combinedEvents, ...allEventsCache.value['after_story']];
        }

        // 更新事件列表
        allEvents.value = combinedEvents;
        
      } catch (e) {
        console.error('加载事件数据失败:', e);
      } finally {
        console.log('所有事件数据加载完成:', allEvents.value);
        isLoading.value = false;
      }
    };

    onMounted(async () => {
      try {
        // 预加载索引文件
        await loadGameDataIndex()
        await loadAllEvents()

        // 检查版本更新
        // checkVersion();
      } catch (e) {
        console.error('初始化数据失败:', e);
      }

      const namespace = shapes;

      const graph = new dia.Graph({}, { cellNamespace: namespace });

      const paper = new dia.Paper({
        el: document.getElementById('paper'),
        model: graph,
        width: '100%',
        height: '100%',
        background: { color: '#F5F5F5' },
        cellViewNamespace: namespace
      });
      console.log(paper);

      const rect1 = new shapes.standard.Rectangle();
      rect1.position(25, 25);
      rect1.resize(180, 50);
      rect1.addTo(graph);

      const rect2 = new shapes.standard.Rectangle();
      rect2.position(95, 225);
      rect2.resize(180, 50);
      rect2.addTo(graph);

      rect1.attr('label', { text: 'Hello!', fill: '#353535' });
      rect2.attr('label', { text: allEvents.value.length, fill: '#353535' });
    });

    return {
      getTypeLabel,
      allEvents,
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
