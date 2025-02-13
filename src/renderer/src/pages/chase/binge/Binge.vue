<template>
  <div class="binge-container">
    <div class="main" infinite-wrapper>
      <div class="main-flow-wrap">
        <div v-for="item in bingeConfig.data" :key="item.id" class="card-wrap">
          <div class="card" @click="playEvent(item)">
            <div v-if="item.videoUpdate" class="card-header">
              <span class="card-header-tag card-header-tag-orange">
                <span class="card-header-tag-tagtext">有更新哟</span>
              </span>
            </div>
            <div class="card-main">
              <div class="card-close" @click.stop="removeEvent(item)"></div>
              <t-image
                class="card-main-item"
                :src="item.videoImage"
                :style="{ width: '190px', height: '105px' }"
                :lazy="true"
                fit="cover"
                :loading="renderLoading"
                :error="renderError"
              >
                <template #overlayContent>
                  <div class="op">
                    <span v-if="item.siteName"> {{ item.siteName }}</span>
                  </div>
                </template>
              </t-image>
            </div>
            <div class="card-footer">
              <div class="card-footer-title">
                <span class="card-footer-title-name">{{ item.videoName }}</span>
              </div>
              <p class="card-footer-desc">{{ item.videoRemarks }}</p>
            </div>
          </div>
        </div>
      </div>
      <infinite-loading
        :identifier="infiniteId"
        style="text-align: center; margin-bottom: 2em"
        :duration="200"
        @infinite="load"
      >
        <template #complete>人家是有底线的</template>
        <template #error>哎呀，出了点差错</template>
      </infinite-loading>
      <detail-view v-model:visible="isVisible.detail" :site="siteData" :data="formDetailData"/>
    </div>
  </div>
</template>

<script setup lang="tsx">
import 'v3-infinite-loading/lib/style.css';
import lazyImg from '@/assets/lazy.png';

import { useEventBus } from '@vueuse/core';
import PQueue from 'p-queue';
import _ from 'lodash';
import { MessagePlugin } from 'tdesign-vue-next';
import InfiniteLoading from 'v3-infinite-loading';
import { ref, reactive } from 'vue';

import { fetchStarList, delStar } from '@/api/star';
import { fetchFilmDetail, fetchSiteList } from '@/api/site';
import { fetchDetail } from '@/utils/cms';
import { usePlayStore } from '@/store';
import DetailView from '../../film/Detail.vue';

const store = usePlayStore();

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

const queue = new PQueue({ concurrency: 5 }); // 设置并发限制为5

const pagination = ref({
  pageIndex: 0,
  pageSize: 32,
  count: 0,
});
const formDetailData = ref({
  neme: '',
  key: '',
  type: 1,
}); //  详情组件源传参
const siteData = ref();
const isVisible = reactive({
  detail: false
});
const bingeConfig = ref({
  data: []
});
const siteConfig = ref({
  data: []
})

const infiniteId = ref(+new Date());

const getBingeList = async () => {
  let length = 0;
  try {
    const { pageIndex, pageSize } = pagination.value;

    const [star_res, site_res] = await Promise.all([
      fetchStarList(pageIndex, pageSize),
      fetchSiteList()
    ]);

    if (_.has(site_res, "data") && !_.isEmpty(site_res["data"])) {
      siteConfig.value.data = site_res["data"]
    }

    for (const item of star_res.data) {
      const findItem = siteConfig.value.data.find(({ id }) => id === item.relateId);
      if (findItem) item.site = { ...findItem };
      item.siteName = findItem ? findItem.name : '该源应该被删除了哦';
    }

    bingeConfig.value.data = _.unionWith(bingeConfig.value.data, star_res.data, _.isEqual);

    pagination.value.count = star_res.total;
    pagination.value.pageIndex++;
    
    length = _.size(star_res.data);
    return length;
  } catch (err) {
    console.error(err);
    length = 0;
  } finally {
    console.log(`[binge] load data length: ${length}`);
    return length;
  } 
};

const load = async ($state) => {
  console.log('[binge] loading...');
  try {
    const resLength = await getBingeList();
    if (resLength === 0) $state.complete();
    else $state.loaded();
  } catch (error) {
    $state.error();
  }
};

// 播放
const playEvent = async (item) => {
  try {
    const { videoName, relateId, videoId } = item;
    const site = siteConfig.value.data.find(({ id }) => id === item.relateId);
    siteData.value = site;
    if ( !('vod_play_from' in item && 'vod_play_url' in item) ) {
      const [detailItem] = await fetchFilmDetail(relateId, videoId);
      item = detailItem;
    }
    console.log(item);

    const playerType = store.getSetting.broadcasterType;

    if (playerType === 'iina' || playerType === 'potplayer' || playerType === 'vlc' ) {
      formDetailData.value = item;
      isVisible.detail = true;
    } else {
      const config = {
        type: 'film',
        data: {
          info: item,
          ext: { site: { key: relateId, type: site.type } },
        },
      };

      store.updateConfig(config);
      window.electron.ipcRenderer.send('openPlayWindow', videoName);
    }
  } catch (err) {
    console.error(err);
    MessagePlugin.warning('请求资源站失败，请检查网络!');
  }
};

// 删除
const removeEvent = async (item) => {
  const { id } = item;
  await delStar(id);
  bingeConfig.value.data = _.reject(bingeConfig.value.data, { id });
  pagination.value.count--;
};

// 更新
const updateVideoRemarks = (item, res) => {
  const index = _.findIndex(bingeConfig.value.data, { ...item });
  const isUpdate = res.vod_remarks !== bingeConfig.value.data[index].videoRemarks;
  bingeConfig.value.data[index].videoUpdate = isUpdate;
  bingeConfig.value.data[index].videoRemarks = res.vod_remarks;
};

const checkUpdaterEvent = async () => {
  const fetchAndUpdateVideoRemarks = async (item) => {
    if (item.siteName = '该源应该被删除了哦') return;
    const { site, videoId } = item;
    try {
      const [res] = await fetchDetail(site, videoId);
      if (res.vod_remarks) {
        updateVideoRemarks(item, res);
      }
    } catch (err) {
      console.error(err);
    }
  };
  await Promise.all(
    bingeConfig.value.data.map(async (item) => {
      await queue.add(() => fetchAndUpdateVideoRemarks(item));
    }),
  );
};

// 监听播放器变更
const eventBus = useEventBus('binge-reload');
eventBus.on(() => {
  bingeConfig.value.data = [];
  if (!_.size(bingeConfig.value.data)) infiniteId.value++;
  pagination.value.pageIndex = 0;
});

// 对父组件暴露
defineExpose({
  checkUpdaterEvent,
});
</script>

<style lang="less" scoped>
.binge-container {
  height: inherit;
  .main {
    &-flow-wrap {
      .card-wrap {
        display: inline-block;
        width: 190px;
        margin: 10px 15px 10px 0;
        position: relative;
        &:hover {
          .card-main-item {
            :deep(img) {
              transition: all 0.25s ease-in-out;
              transform: scale(1.05);
            }
          }
        }
        .card {
          box-sizing: border-box;
          width: 190px;
          height: 160px;
          position: relative;
          display: inline-block;
          vertical-align: top;
          .card-close {
            display: none;
            position: absolute;
            right: -9px;
            top: -9px;
            height: 22px;
            width: 22px;
            background: url(../../../assets/close.png) 0 0 no-repeat;
            z-index: 1000;
            cursor: pointer;
            background-size: 100%;
          }
          .card-header {
            position: absolute;
            color: #fff;
            font-size: 12px;
            z-index: 15;
            height: 18px;
            line-height: 18px;
            left: 0;
            top: 0;
            &-tag {
              height: 18px;
              line-height: 18px;
              padding: 1px 6px;
              border-radius: 5px 0 5px 0;
              background: #03c8d4;
              display: block;
              &-tagtext {
                display: inline-block;
                font-size: 12px;
                overflow: hidden;
                white-space: nowrap;
                text-overflow: ellipsis;
                max-width: 100px;
              }
            }
            &-tag-orange {
              background: #ffdd9a;
              color: #4e2d03;
            }
          }
          .card-main {
            width: 100%;
            overflow: hidden;
            border-radius: 5px;
            &:hover .card-close {
              display: block !important;
            }
            .card-main-item {
              .op {
                background-color: rgba(22, 22, 23, 0.8);
                border-radius: 0 0 5px 5px;
                width: 100%;
                color: rgba(255, 255, 255, 0.8);
                position: absolute;
                bottom: 0px;
                display: flex;
                justify-content: center;
                span {
                  font-size: 12px;
                  font-weight: bolder;
                }
              }
            }
          }
          .card-footer {
            overflow: hidden;
            height: auto;
            line-height: 26px;
            .card-footer-title {
              display: flex;
              align-items: center;
              &-name {
                text-overflow: ellipsis;
                white-space: nowrap;
                overflow: hidden;
                font-weight: 500;
              }
            }
            .card-footer-desc {
              line-height: 10px;
              font-size: 10px;
              color: var(--td-gray-color-7);
              font-weight: normal;
            }
          }
        }
      }
    }
  }
}
</style>
