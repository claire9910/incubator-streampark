<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<script lang="ts">
  import { defineComponent } from 'vue';
  export default defineComponent({
    name: 'ApplicationDetail',
  });
</script>
<script setup lang="ts" name="ApplicationDetail">
  import { PageWrapper } from '/@/components/Page';
  import { Description, useDescription } from '/@/components/Description';
  import { Icon } from '/@/components/Icon';
  import { useRoute, useRouter } from 'vue-router';
  import { fetchBackUps, fetchGet, fetchOptionLog, fetchYarn } from '/@/api/flink/app/app';
  import { onUnmounted, reactive, h, unref, ref } from 'vue';
  import { useIntervalFn, useClipboard } from '@vueuse/core';
  import { AppListRecord } from '/@/api/flink/app/app.type';
  import { Tooltip, Divider } from 'ant-design-vue';
  import { handleView } from './utils';
  import { Button } from '/@/components/Button';
  import { getDescSchema } from './data/detail.data';
  import { fetchCheckToken, fetchCopyCurl } from '/@/api/sys/token';
  import { useMessage } from '/@/hooks/web/useMessage';
  import { baseUrl } from '/@/adapter/api/baseUrl';
  import { fetchListVer } from '/@/api/flink/config';
  import { fetchSavePonitHistory } from '/@/api/flink/app/savepoint';
  import Mergely from './components/Mergely.vue';
  import DetailTab from './components/AppDetail/DetailTab.vue';
  import { createDetailProviderContext } from './hooks/useDetailContext';
  import { useDrawer } from '/@/components/Drawer';

  const route = useRoute();
  const router = useRouter();

  const { createMessage } = useMessage();
  const { copy } = useClipboard();

  const app = reactive<Partial<AppListRecord>>({});

  createDetailProviderContext({ app });

  const yarn = ref('');
  const detailTabs = reactive({
    showConf: false,
    showSaveOption: false,
    showBackup: false,
    showOptionLog: false,
  });

  const [registerDescription] = useDescription({
    schema: [
      ...(getDescSchema() as any),
      {
        field: 'resetApi',
        label: h('div', null, [
          'Rest Api',
          h(
            Tooltip,
            {
              title:
                'Rest API external call interface,other third-party systems easy to access StreamPark',
              placement: 'top',
            },
            () =>
              h(Icon, {
                icon: 'ant-design:question-circle-outlined',
                class: 'pl-5px',
                color: 'red',
              }),
          ),
        ]),
        render: () => [
          h(
            Button,
            {
              type: 'primary',
              shape: 'round',
              class: 'mx-3px px-5px',
              onClick: () => handleCopyCurl('/flink/app/start'),
            },
            () => [h(Icon, { icon: 'ant-design:copy-outlined' }), 'Copy Start cURL'],
          ),
          h(
            Button,
            {
              type: 'primary',
              shape: 'round',
              class: 'mx-3px px-5px',
              onClick: () => handleCopyCurl('/flink/app/cancel'),
            },
            () => [h(Icon, { icon: 'ant-design:copy-outlined' }), 'Copy Cancel cURL'],
          ),
          h(
            Button,
            {
              type: 'link',
              shape: 'round',
              class: 'mx-3px px-5px',
              onClick: () => handleDocPage(),
            },
            () => [h(Icon, { icon: 'ant-design:link-outlined' }), 'Api Doc Center'],
          ),
        ],
      },
    ],
    data: app,
    layout: 'vertical',
    column: 3,
  });

  const [registerConfDrawer] = useDrawer();

  /* Flink Web UI */
  function handleFlinkView() {
    handleView(app as any, unref(yarn));
  }

  const { pause } = useIntervalFn(
    () => {
      handleGetAppInfo();
    },
    5000,
    { immediateCallback: true },
  );

  async function handleGetAppInfo() {
    if (!route.query.appId) router.back();
    const res = await fetchGet({
      id: route.query.appId as string,
    });
    // Get data for the first time
    if (Object.keys(app).length == 0) {
      if ([2, 3, 4].includes(res.executionMode)) {
        handleYarn();
      }
      handleDetailTabs();
    }
    Object.assign(app, res);
  }

  async function handleDetailTabs() {
    const commonParams = {
      appId: route.query.appId as string,
      pageNum: 1,
      pageSize: 10,
    };

    const confList = await fetchListVer(commonParams);
    const pointHistory = await fetchSavePonitHistory(commonParams);
    const backupList = await fetchBackUps(commonParams);
    const optionList = await fetchOptionLog(commonParams);

    if (confList.records.length > 0) detailTabs.showConf = true;
    if (pointHistory.records.length > 0) detailTabs.showSaveOption = true;
    if (backupList.records.length > 0) detailTabs.showBackup = true;
    if (optionList.records.length > 0) detailTabs.showOptionLog = true;
  }

  /* Get yarn data */
  async function handleYarn() {
    const res = await fetchYarn();
    yarn.value = res;
  }

  /* copyCurl */
  async function handleCopyCurl(urlPath) {
    const resp = await fetchCheckToken({});
    const result = parseInt(resp);
    if (result === 0) {
      createMessage.error('access token is null,please contact the administrator to add.');
    } else if (result === 1) {
      createMessage.error('access token is invalid,please contact the administrator.');
    } else {
      const res = await fetchCopyCurl({
        appId: app.id,
        baseUrl: baseUrl(),
        path: urlPath,
      });
      copy(res);
      createMessage.success('copy successful');
    }
  }

  /* Documentation page */
  function handleDocPage() {
    const res = window.origin.split(':')[1] + ':10000/doc.html';
    window.open(res);
  }

  onUnmounted(() => {
    pause();
  });
</script>
<template>
  <PageWrapper content-full-height content-background contentClass="p-24px">
    <div class="mb-15px">
      <span class="app-bar">Application Info</span>
      <a-button type="primary" shape="circle" @click="router.back()" class="float-right -mt-8px">
        <Icon icon="ant-design:arrow-left-outlined" />
      </a-button>
      <a-button
        type="danger"
        @click="handleFlinkView"
        :disabled="app.state !== 5 || (yarn === null && app.flinkRestUrl === null)"
        class="float-right -mt-8px mr-20px"
      >
        <Icon icon="ant-design:cloud-outlined" />
        Flink Web UI
      </a-button>
    </div>
    <Description @register="registerDescription" />
    <Divider class="mt-20px -mb-17px" />
    <DetailTab :app="app" :tabConf="detailTabs" />

    <Mergely @register="registerConfDrawer" :readOnly="true" />
  </PageWrapper>
</template>
<style lang="less">
  @import url('./styles/Detail.less');
</style>
