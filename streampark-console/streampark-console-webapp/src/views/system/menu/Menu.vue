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

<template>
  <a-card
    :bordered="false">
    <div
      class="table-page-search-wrapper">
      <a-form
        layout="inline">
        <a-row
          :gutter="48">
          <div
            :class="advanced ? null: 'fold'">
            <a-col
              :md="8"
              :sm="24">
              <a-form-item
                label="Name"
                :label-col="{span: 4}"
                :wrapper-col="{span: 18, offset: 2}">
                <a-input
                  v-model="queryParams.menuName" />
              </a-form-item>
            </a-col>
            <a-col
              :md="8"
              :sm="24">
              <a-form-item
                label="Create Time"
                :label-col="{span: 4}"
                :wrapper-col="{span: 18, offset: 2}">
                <range-date
                  @change="handleDateChange"
                  ref="createTime" />
              </a-form-item>
            </a-col>
            <a-col
              :md="8"
              :sm="24">
              <span
                class="table-page-search-bar">
                <a-button
                  type="primary"
                  shape="circle"
                  icon="search"
                  @click="search" />
                <a-button
                  type="primary"
                  shape="circle"
                  icon="rest"
                  @click="reset" />
                <a-popconfirm
                  title="Please select a creation type"
                  ok-text="button"
                  cancel-text="menu"
                  @cancel="() => createMenu()"
                  @confirm="() => createButton()">
                  <a-icon
                    slot="icon"
                    type="question-circle-o"
                    style="color: orangered" />
                  <a-button
                    v-permit="'menu:add'"
                    type="primary"
                    shape="circle"
                    icon="plus" />
                </a-popconfirm>
              </span>
            </a-col>
          </div>
        </a-row>
      </a-form>
    </div>

    <div>
      <!-- Table Info -->
      <a-table
        :columns="columns"
        :key="key"
        :data-source="dataSource"
        :pagination="pagination"
        :loading="loading"
        @change="handleTableChange"
        :scroll="{ x: 1650 }">
        <template
          slot="icon"
          slot-scope="text">
          <a-icon
            :type="text" />
        </template>
        <template
          slot="operation"
          slot-scope="text, record">
          <svg-icon
            v-permit="'menu:update'"
            name="edit"
            border
            @click.native="edit(record)"
            title="edit" />
          <a-badge
            v-noPermit="'menu:update'"
            status="warning"
            text="No permission" />
        </template>
      </a-table>
    </div>

    <!-- Add menu -->
    <menu-add
      @close="handleMenuAddClose"
      @success="handleMenuAddSuccess"
      :menu-add-visiable="menuAddVisiable" />
    <!-- Modify menu -->
    <menu-edit
      ref="menuEdit"
      @close="handleMenuEditClose"
      @success="handleMenuEditSuccess"
      :menu-edit-visiable="menuEditVisiable" />
    <!-- Add button -->
    <button-add
      @close="handleButtonAddClose"
      @success="handleButtonAddSuccess"
      :button-add-visiable="buttonAddVisiable" />
    <!-- Modify button -->
    <button-edit
      ref="buttonEdit"
      @close="handleButtonEditClose"
      @success="handleButtonEditSuccess"
      :button-edit-visiable="buttonEditVisiable" />

  </a-card>
</template>

<script>
import RangeDate from '@/components/DateTime/RangeDate'
import MenuAdd from './MenuAdd'
import MenuEdit from './MenuEdit'
import ButtonAdd from './ButtonAdd'
import ButtonEdit from './ButtonEdit'
import SvgIcon from '@/components/SvgIcon'
import { list } from '@/api/menu'
import storage from '@/utils/storage'
import {USER_NAME} from '@/store/mutation-types'

export default {
  name: 'Menu',
  components: { ButtonAdd, ButtonEdit, RangeDate, MenuAdd, MenuEdit, SvgIcon },
  data () {
    return {
      advanced: false,
      key: +new Date(),
      queryParams: {},
      filteredInfo: {},
      dataSource: [],
      pagination: {
        defaultPageSize: 10000000,
        hideOnSinglePage: true,
        indentSize: 100
      },
      loading: false,
      menuAddVisiable: false,
      menuEditVisiable: false,
      buttonAddVisiable: false,
      buttonEditVisiable: false
    }
  },
  computed: {
    columns () {
      return [{
        title: 'Name',
        dataIndex: 'text',
        width: 200,
        fixed: 'left'
      }, {
        title: 'Icon',
        dataIndex: 'icon',
        scopedSlots: { customRender: 'icon' }
      }, {
        title: 'Type',
        dataIndex: 'type',
        customRender: (text) => {
          switch (text) {
            case '0': return <a-tag color = "cyan" >menu</a-tag>
            case '1': return <a-tag color = "pink"> button </a-tag>
            default: return text
          }
        }
      }, {
        title: 'Path',
        dataIndex: 'path'
      }, {
        title: 'Vue Component',
        dataIndex: 'component'
      }, {
        title: 'Permission',
        dataIndex: 'permission'
      }, {
        title: 'Order By',
        dataIndex: 'order'
      }, {
        title: 'Create Time',
        dataIndex: 'createTime'
      }, {
        title: 'Modify Time',
        dataIndex: 'modifyTime'
      }, {
        title: 'Operation',
        dataIndex: 'operation',
        width: 120,
        scopedSlots: { customRender: 'operation' },
        fixed: 'right'
      }]
    },
    userName() {
      return storage.get(USER_NAME)
    }
  },
  mounted () {
    this.fetch()
  },
  methods: {
    handleMenuEditClose () {
      this.menuEditVisiable = false
    },
    handleMenuEditSuccess () {
      this.menuEditVisiable = false
      this.$message.success('Modify the menu successfully')
      this.fetch()
    },
    handleButtonEditClose () {
      this.buttonEditVisiable = false
    },
    handleButtonEditSuccess () {
      this.buttonEditVisiable = false
      this.$message.success('Modify button successfully')
      this.fetch()
    },
    edit (record) {
      if (record.type === '0') {
        this.$refs.menuEdit.setFormValues(record)
        this.menuEditVisiable = true
      } else {
        this.$refs.buttonEdit.setFormValues(record)
        this.buttonEditVisiable = true
      }
    },
    handleButtonAddClose () {
      this.buttonAddVisiable = false
    },
    handleButtonAddSuccess () {
      this.buttonAddVisiable = false
      this.$message.success('Add button successfully')
      this.fetch()
    },
    createButton () {
      this.buttonAddVisiable = true
    },
    handleMenuAddClose () {
      this.menuAddVisiable = false
    },
    handleMenuAddSuccess () {
      this.menuAddVisiable = false
      this.$message.success('Added menu successfully')
      this.fetch()
    },
    createMenu () {
      this.menuAddVisiable = true
    },
    handleDateChange (value) {
      if (value) {
        this.queryParams.createTimeFrom = value[0]
        this.queryParams.createTimeTo = value[1]
      }
    },
    search () {
      const { filteredInfo } = this
      this.fetch({
        ...this.queryParams,
        ...filteredInfo
      })
    },
    reset () {
      // Reset filteredInfo
      this.filteredInfo = {}
      // Reset queryParams
      this.queryParams = {}
      // Reset createTime
      this.$refs.createTime.reset()
      this.fetch()
    },
    handleTableChange (pagination, filters, sorter) {
      this.filteredInfo = filters || {}
      this.fetch({
        sortField: sorter.field,
        sortOrder: sorter.order,
        ...this.queryParams,
        ...filters
      })
    },
    fetch (params = {}) {
      this.loading = true
      list({
        ...params
      }).then((resp) => {
        this.loading = false
        const data = resp.data
        if (Object.is(data.rows.children, undefined)) {
          this.dataSource = data.rows
        } else {
          this.dataSource = data.rows.children
        }
      })
    }

  }
}
</script>

<style>

.table-expanded-icon {
  margin-right: 8px;
}

.table-expanded-icon .anticon {
  font-size: 10px;
  color: #666;
}
</style>
