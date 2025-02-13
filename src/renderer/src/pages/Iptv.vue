<template>
  <div class="iptv view-container">
    <common-nav title="电视" :list="iptvConfig.data" :active="active.nav" @change-key="changeDefaultIptvEvent" />
    <div class="content">
      <header class="header">
        <div class="header-nav">
          <TagNav :list="classConfig.data" :active="active.class" @change-key="changeClassEvent"/>
        </div>
      </header>
      <div class="container">
        <div class="content-wrapper">
          <div class="container-flow-wrap">
            <div v-for="item in channelData.list" :key="item.id" class="card-wrap">
              <div class="card" @click="playEvent(item)" @contextmenu="conButtonClick(item, $event)">
                <div v-show="iptvConfig.ext.status" class="card-header">
                  <div class="status">
                    <span v-if="item.status && item.status < 500" class="status-item sucess">{{ item.status }}ms</span>
                    <span v-else class="status-item error">{{ item.status ? `${item.status}ms` : '超时' }}</span>
                  </div>
                </div>
                <div class="card-main">
                  <t-image
                    class="card-main-item"
                    :src="iptvConfig.ext.thumbnail? item.thumbnail: generateLogo(item)"
                    :style="{
                      width: '180px',
                      height: '100px',
                      background: 'none',
                      padding: iptvConfig.ext.thumbnail
                        ? 'none'
                        : '35px 60px'
                    }"
                    :lazy="true"
                    :loading="renderLoading"
                    :error="renderError"
                  >
                  </t-image>
                </div>
                <div class="card-footer">
                  <span class="card-footer-title">{{ item.name }}</span>
                </div>
              </div>
            </div>
            <context-menu :show="isVisible.contentMenu" :options="optionsComponent" @close="isVisible.contentMenu = false">
              <context-menu-item label="拷贝频道链接" @click="copyChannelEvent" />
              <context-menu-item label="删除频道" @click="delChannelEvent" />
            </context-menu>
          </div>
          <infinite-loading
            v-if="isVisible.infiniteLoading"
            :identifier="infiniteId"
            style="text-align: center; margin-bottom: 2em"
            :duration="200"
            @infinite="load"
          >
            <template #complete>{{ infiniteCompleteTip }}</template>
            <template #error>哎呀，出了点差错</template>
          </infinite-loading>
        </div>
      </div>
    </div>
    <t-back-top
      container=".container"
      :visible-height="200"
      size="small"
      :offset="['1.4rem', '0.5rem']"
      :duration="2000"
      :firstload="false"
    ></t-back-top>
  </div>
</template>

<script setup lang="tsx">
import '@imengyu/vue3-context-menu/lib/vue3-context-menu.css';
import 'v3-infinite-loading/lib/style.css';
import lazyImg from '@/assets/lazy.png';

import { ContextMenu, ContextMenuItem } from '@imengyu/vue3-context-menu';
import { useClipboard, useEventBus } from '@vueuse/core';

import _ from 'lodash';
import PQueue from 'p-queue';
import { ArticleIcon } from 'tdesign-icons-vue-next';
import { MessagePlugin } from 'tdesign-vue-next';
import InfiniteLoading from 'v3-infinite-loading';
import { computed, onMounted, ref, reactive } from 'vue';

import { checkUrlIpv6 } from '@/utils/tool';
import { usePlayStore, useSettingStore } from '@/store';
import { fetchIptvActive, clearChannel, addChannel, fetchChannelList, delChannelItem } from '@/api/iptv';
import { setDefault } from '@/api/setting';
import { parseChannel, checkChannel, stopCheckChannel } from '@/utils/channel';

import CommonNav from '../components/common-nav/index.vue';
import TagNav from '../components/tag-nav/index.vue';

const storePlayer = usePlayStore();
const storeSetting = useSettingStore();

const renderError = () => {
  return (
    <div class="renderIcon" style="width: 100%; height: 100px; overflow: hidden;">
      <img src={ lazyImg } style="width: 100%; height: 100%; object-fit: cover;"/>
    </div>
  );
};
const renderLoading = () => {
  return (
    <div class="renderIcon" style="width: 100%; height: 100px; overflow: hidden;">
      <img src={ lazyImg } style="width: 100%; height: 100%; object-fit: cover;"/>
    </div>
  );
};

const isVisible = reactive({
  infiniteLoading: false, // 筛选
  contentMenu: false
});
const searchTxt = ref('');
const infiniteId = ref(+new Date());
const infiniteCompleteTip = ref('没有更多内容了');
const pagination = ref({
  pageIndex: 0,
  pageSize: 32,
  count: 0,
});

const iptvConfig = ref({
  default: {},
  data: [],
  ext: {
    epg: "http://diyp.112114.xyz/",
    logo: "https://epg.112114.eu.org/logo/",
    skipIpv6: true,
    status: false,
    thumbnail: false
  },
  ua: ""
})

const active = ref({
  nav: '',
  class: '全部'
})

const channelData = ref({
  list: [],
  total: 0,
})

const classConfig = ref({
  data: [{ type_id: '全部', type_name: '全部' }]
})

const mode = computed(() => {
  return storeSetting.displayMode;
});

const optionsComponent = ref({
  zIndex: 15,
  width: 160,
  x: 500,
  y: 200,
  theme: mode.value === 'light' ? 'default' : 'mac dark',
});

const channelItem = ref(null);

const queue = new PQueue({ concurrency: 5 }); // 设置并发限制为5

onMounted(() => {
  getSetting();
});

// 获取配置
const getSetting = async () => {
  try {
    const data = await fetchIptvActive();
    if (_.has(data, 'default') && !_.isEmpty(data["default"])) {
      iptvConfig.value.default = data["default"];
      active.value.nav = data["default"]["id"];
    }
    if (_.has(data, 'ext') && !_.isEmpty(data["ext"])) {
      iptvConfig.value.ext = data["ext"];
    }
    if (_.has(data, 'data') && !_.isEmpty(data["data"])) {
      iptvConfig.value.data = data["data"];
    }

    isVisible.infiniteLoading = true;
  } catch (err) {
    isVisible.infiniteLoading = false;
  }
};

// 获取直播列表
const getChannel = async () => {
  const { pageIndex, pageSize } = pagination.value;
  const { skipIpv6, status, thumbnail } = iptvConfig.value.ext;
  const { list } = channelData.value;

  const res = await fetchChannelList(pageIndex, pageSize, searchTxt.value, active.value.class);
  pagination.value.count = res["total"];
  classConfig.value.data = res["class"];

  const sourceLength = res.data.length;

  if (skipIpv6) res.data = await checkChannelListIpv6(res.data);
  const restultLength = res.data.length;
  channelData.value.list = _.unionWith(list, res.data, _.isEqual);

  if (status) checkChannelDelay(pageIndex, pageSize);
  if (thumbnail) generateThumbnail(pageIndex, pageSize);

  // 判断是否开启检查；判断原数据；判断原和目的
  // 1. 开启检查：为0  1).返回原或0
  // 2. 开启检查：非0  1).相等直接返回  2).不相等[原非0:目的0再次请求；原小于pagesise:目的小于pagesise返回原]  3).
  // 3. 非开启检查     1).直接返回原
  const length = sourceLength;

  if (skipIpv6 && sourceLength && sourceLength !== restultLength) {
    if (res.data.length === 0) {
      pagination.value.pageIndex++;
      await getChannel();
    }
  }

  pagination.value.pageIndex++;
  console.log(`[iptv] load data length: ${length}`);
  return length;
};

const load = async ($state: { complete: () => void; loaded: () => void; error: () => void }) => {
  console.log('[iptv] loading...');
  try {
    const resLength = await getChannel();

    if (_.isEmpty(active.value.nav) && _.isEmpty(channelData.value.list) ) {
      infiniteCompleteTip.value = '暂无数据,请前往设置-直播源设置默认源!';
      $state.complete();
      // return;
    }

    if (resLength === 0) {
      $state.complete();
    } else $state.loaded();
  } catch (err) {
    console.log(err);
    $state.error();
  }
};

// 搜索
const searchEvent = async () => {
  console.log(`[iptv] search keyword: ${searchTxt.value}`);
  channelData.value = { list: [], total: 0 };
  infiniteId.value++;
  pagination.value.pageIndex = 0;
};

// 切换分类
const changeClassEvent = async (id) => {
  clearQueue();
  infiniteCompleteTip.value = '没有更多内容了!';
  active.value.class = id;

  channelData.value = { list: [], total: 0 };
  infiniteId.value++;
  pagination.value.pageIndex = 0;
};

// 播放
const playEvent = (item: { name: any }) => {
  const playerType = storePlayer.getSetting.broadcasterType
  if (playerType === 'iina') window.open(`iina://weblink?url=${item.url}`, '_self');
  else if (playerType === 'potplayer') window.open(`potplayer://${item.url}`, '_self')
  else if (playerType === 'vlc') window.open(`vlc://${item.url}`, '_self');
  else {
    const { epg, skipIpv6 } = iptvConfig.value.ext;
    storePlayer.updateConfig({
      type: 'iptv',
      data: {
        info: item,
        ext: { epg, skipIpv6: skipIpv6 },
      },
    });
    window.electron.ipcRenderer.send('openPlayWindow', item.name);
  }
};

// 检查ipv6
const checkChannelListIpv6 = async (data) => {
  const newdata = await Promise.allSettled(
    data.map(async (item) => {
      try {
        const checkStatus = await checkUrlIpv6(item.url);
        if (checkStatus !== 'IPv6') return item;
        return false;
      } catch (err) {
        console.log(err);
        return false;
      }
    }),
  );

  const filteredData = newdata
    .filter((result) => result.status === 'fulfilled' && result.value !== false)
    .map((result) => {
      if (result.status === 'fulfilled') return result.value;
      return null;
    })
    .filter((item) => item !== null);

  return filteredData;
};

// 检查状态
const checkChannelDelay = async (pageIndex: number, pageSize: number) => {
  console.log(`[channel] checkChannelDelay`);
  const start = pageIndex * pageSize;
  const end = (pageIndex + 1) * pageSize;
  const dataList = _.slice(channelData.value.list, start, end); // 从原数组中截取需要处理的数据段

  const updateStatus = async (item) => {
    try {
      const result = await checkChannel(item.url);
      return result;
    } catch (error) {
      return false;
    }
  };

  // 将任务加入队列
  const results = await Promise.all(dataList.map(item => queue.add(() => updateStatus(item))));

  // 更新频道状态
  results.forEach((result, i) => {
    const index = start + i;
    if (index < channelData.value.list.length) {
      channelData.value.list[index].status = result;
    }
  });
};

// 缩略图
const generateThumbnail = async (pageIndex: number, pageSize: number) => {
  console.log(`[channel] generateThumbnail`);
  const start = pageIndex * pageSize;
  const end = (pageIndex + 1) * pageSize;
  const dataList = _.slice(channelData.value.list, start, end); // 从原数组中截取需要处理的数据段

  const updateThumbnail = async (item) => {
    window.electron.ipcRenderer.invoke('ffmpeg-thumbnail', item.url, item.id).then(res => {
      if (res) {
        const index = _.findIndex(channelData.value.list, {id: res.key});
        channelData.value.list[index]["thumbnail"] = res.url;
      }
    });
  };

  for (let i = 0; i < dataList.length; i++) {
    const item = dataList[i];
    // eslint-disable-next-line no-await-in-loop
    queue.add(() => {
      updateThumbnail(item);
    });
  }
}

// 清空队列，并终止请求
const clearQueue = () => {
  if (queue.size > 0) {
    console.log(`[queue] clear queuectasks and cancel axios request`);
    queue.pause(); // 暂停队列，阻止新的任务加入
    stopCheckChannel(); // 中止正在执行的任务
    queue.clear(); // 清空队列中的任务
    queue.start(); // 继续接管任务加入
  }
};

// 切换源
const changeDefaultIptvEvent = async (id: string) => {
  try {
    isVisible.infiniteLoading = true;
    infiniteCompleteTip.value = '没有更多内容了!';
    channelData.value = { list: [], total: 0 };
    classConfig.value.data = [{ type_id: '全部', type_name: '全部' }];
    active.value.class = '全部';
    active.value.nav = id;

    const { url, type } = _.find(iptvConfig.value.data, { id });
    await clearChannel();
    const docs = await parseChannel(type, url);
    await addChannel(docs);
    await setDefault('defaultIptv', id);

    MessagePlugin.success('设置成功');
    infiniteId.value++;
    pagination.value.pageIndex = 0;
  } catch (err) {
    MessagePlugin.error(`设置失败, 错误信息:${err}`);
  }
};

// 监听设置默认源变更
const iptvReloadeventBus = useEventBus<string>('iptv-reload');
const channelSearcheventBus = useEventBus<string>('channel-search');

channelSearcheventBus.on((kw: string)=>{
  searchTxt.value = kw;
  searchEvent();
});

iptvReloadeventBus.on(async () => {
  infiniteCompleteTip.value = '没有更多内容了!';
  searchTxt.value = '';
  classConfig.value.data = [{ type_id: '全部', type_name: '全部' }];
  active.value.class = '全部';
  active.value.nav = '';
  channelData.value = { list: [], total: 0 };
  await getSetting();
  infiniteId.value++;
  pagination.value.pageIndex = 0;
});

// 右键
const conButtonClick = (item: any, { x, y }: any) => {
  isVisible.contentMenu = true;
  Object.assign(optionsComponent.value, { x, y });
  channelItem.value = item;
};

// 删除
const delChannelEvent = () => {
  const index = channelData.value.list.indexOf(channelItem.value);
  if (index > -1) {
    channelData.value.list.splice(index, 1);
    delChannelItem(channelItem.value.id);
  }
  isVisible.contentMenu = false;
};

// 拷贝
const copyChannelEvent = () => {
  const { isSupported, copy } = useClipboard();
  if (isSupported) copy(channelItem.value.url);
  isVisible.contentMenu = false;
};

// 生成台标
const generateLogo = (item) => {
  let url = item.logo;
  if (iptvConfig.value.ext.logo) {
    url = `${iptvConfig.value.ext.logo}${item.name}.png`;
  };
  return url;
}
</script>

<style lang="less" scoped>
.iptv {
  height: 100%;
  display: flex;
  position: relative;
  flex-direction: row;
  overflow: hidden;

  .membership-wrapper {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 40px;
    width: 148px;
    border: 2px solid rgba(132, 133, 141, 0.16);
    transition: all .3s ease;
    font-size: 14px;
    border-radius: 5px;
    cursor: pointer;
    font-size: 20px;
    .member-name {
      font-size: 12px;
      margin-left: 4px;
    }
  }
  .nav-sub-tab-member-info {
    margin-top: 16px;
  }

  .content {
    flex: 1;
    width: 100%;
    position: relative;
    overflow: hidden;
    padding: var(--td-comp-paddingTB-xs) var(--td-comp-paddingTB-s);
    .header {
      display: flex;
      align-items: center;
      margin-bottom: 10px;
      justify-content: space-between;
      white-space: nowrap;
      flex-shrink: 0;
      width: 100%;
      .header-nav {
        width: 100%;
        overflow: hidden;
      }
    }
    .container {
      flex: 1;
      height: calc(100% - 58px);
      width: 100%;
      .content-wrapper {
        overflow-y: auto;
        width: 100%;
        height: 100%;
        display: flex;
        flex-direction: column;
        position: relative;
        flex-grow: 1;
        .container-flow-wrap {
          display: grid;
          padding: 10px 0;
          grid-template-columns: repeat(auto-fill, 190px);
          grid-column-gap: 45px;
          grid-row-gap: 10px;
          justify-content: center;
          width: inherit;
          .card-wrap {
            flex-direction: column;
            position: relative;
            .card {
              box-sizing: border-box;
              width: 190px;
              height: 110px;
              position: relative;
              box-shadow: var(--td-shadow-1);
              border-radius: var(--td-radius-medium);
              border: 5px solid #211f20;
              background-color: #373536;
              cursor: pointer;
              .card-header {
                .status {
                  z-index: 10;
                  display: flex;
                  align-content: center;
                  flex-direction: row;
                  align-items: center;
                  justify-content: center;
                  width: 40px;
                  height: 15px;
                  background-color: rgb(0 0 0 / 60%);
                  border-radius: 5px;
                  box-shadow: var(--td-shadow-1);
                  position: absolute;
                  top: 5px;
                  right: 5px;
                  .status-item {
                    font-size: 10px;
                  }
                  .error {
                    color: var(--td-error-color);
                  }
                  .sucess {
                    color: var(--td-success-color);
                  }
                }
                
                .card-header-title {
                  background-color: rgba(0, 0, 0, 0.5);
                  border-radius: 5px;
                  padding: 0 5px;
                  position: absolute;
                  top: 5px;
                  left: 5px;
                  color: #fbfbfb;
                  font-size: 10px;
                }
              }
              .card-main {
                width: 100%;
                height: 100%;
                .card-main-item {
                  left: 50%;
                  top: 50%;
                  transform: translate(-50%, -50%);
                  border-radius: 5px;
                }
              }
              .card-footer {
                .card-footer-title {
                  background-color: rgb(0 0 0 / 60%);
                  border-radius: 4px;
                  color: rgba(255, 255, 255, 0.8);
                  font-size: 10px;
                  box-shadow: var(--td-shadow-1);
                  z-index: 10;
                  position: absolute;
                  right: 2px;
                  bottom: 2px;
                  padding: 0 5px;
                  max-width: 90%;
                  font-weight: bold;
                  white-space: nowrap;
                  overflow: hidden;
                  text-overflow: clip;
                }
              }
            }
            .card:hover {
              .card-footer-title {
                color: var(--td-brand-color);
              }
            }
          }
        }
      }
    }
  }
}

.content-items {
  overflow: hidden;
  width: 100%;
  .content-item {
    float: left;
    box-sizing: border-box;
    width: 92px;
    padding-left: 30px;
    height: 46px;
    cursor: pointer;
    span {
      text-shadow: 0 0 0 rgba(0, 0, 0, 0.2);
      font-size: 15px;
      font-weight: 500;
      display: inline-block;
      width: 62px;
      max-width: 62px;
      overflow: hidden;
      white-space: nowrap;
      text-overflow: ellipsis;
      &:hover {
        color: var(--td-brand-color);
      }
    }
  }
}
</style>
