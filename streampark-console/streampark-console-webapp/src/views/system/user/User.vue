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
  <a-card :bordered="false">
    <div
      class="table-page-search-wrapper">
      <a-form
        layout="inline">
        <a-row
          :gutter="48">
          <div
            class="fold">
            <a-col
              :md="8"
              :sm="24">
              <a-form-item
                label="User Name"
                :label-col="{span: 4}"
                :wrapper-col="{span: 18, offset: 2}">
                <a-input
                  v-model="queryParams.username" />
              </a-form-item>
            </a-col>
            <template>
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
            </template>
          </div>
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
              <a-button
                type="primary"
                shape="circle"
                icon="plus"
                v-permit="'user:add'"
                @click="handleAdd" />
            </span>
          </a-col>
        </a-row>
      </a-form>
    </div>

    <!-- table area -->
    <a-table
      ref="TableInfo"
      :columns="columns"
      :data-source="dataSource"
      :pagination="pagination"
      :loading="loading"
      :scroll="{ x: 900 }"
      @change="handleTableChange">
      <template
        slot="email"
        slot-scope="text">
        <a-popover
          placement="topLeft">
          <template
            slot="content">
            <div>
              {{ text }}
            </div>
          </template>
          <p
            style="width: 150px;margin-bottom: 0">
            {{ text }}
          </p>
        </a-popover>
      </template>
      <template
        slot="operation"
        slot-scope="text, record">
        <svg-icon
          v-permit="'user:update'"
          v-if="(record.username !== 'admin' || userName === 'admin')"
          name="edit"
          border
          @click.native="handleEdit(record)"
          title="modify" />
        <svg-icon
          name="see"
          border
          @click.native="handleView(record)"
          title="view" />
        <svg-icon
          v-permit="'user:reset'"
          v-if="(record.username !== 'admin' || userName === 'admin')"
          name="resetpass"
          border
          @click.native="resetPassword(record)"
          title="reset password" />
        <a-popconfirm
          v-permit="'user:delete'"
          v-if="record.username !== 'admin'"
          title="Are you sure delete this user ?"
          cancel-text="No"
          ok-text="Yes"
          @confirm="handleDelete(record)">
          <svg-icon name="remove" border/>
        </a-popconfirm>
      </template>
    </a-table>

    <!-- View user information -->
    <user-info
      :data="userInfo.data"
      :visible="userInfo.visible"
      @close="handleUserInfoClose" />
    <!-- New users -->
    <user-add
      @close="handleUserAddClose"
      @success="handleUserAddSuccess"
      :visible="userAdd.visible" />
    <!-- modify user -->
    <user-edit
      ref="userEdit"
      @close="handleUserEditClose"
      @success="handleUserEditSuccess"
      :visible="userEdit.visible" />
  </a-card>
</template>

<script>
import UserInfo from './UserInfo'
import UserAdd from './UserAdd'
import UserEdit from './UserEdit'
import RangeDate from '@/components/DateTime/RangeDate'
import SvgIcon from '@/components/SvgIcon'

import { list, deleteUser, reset as resetPassword } from '@/api/user'
import storage from '@/utils/storage'
import {USER_NAME} from '@/store/mutation-types'

export default {
  name: 'User',
  components: { UserInfo, UserAdd, UserEdit, RangeDate, SvgIcon },
  data () {
    return {
      userInfo: {
        visible: false,
        data: {}
      },
      userAdd: {
        visible: false
      },
      userEdit: {
        visible: false
      },
      queryParams: {},
      filteredInfo: null,
      sortedInfo: null,
      paginationInfo: null,
      dataSource: [],
      loading: false,
      pagination: {
        pageSizeOptions: ['10', '20', '30', '40', '100'],
        defaultCurrent: 1,
        defaultPageSize: 10,
        showQuickJumper: true,
        showSizeChanger: true,
        showTotal: (total, range) => `display ${range[0]} ~ ${range[1]} records，total ${total}`
      }
    }
  },
  computed: {
    columns () {
      let { sortedInfo, filteredInfo } = this
      sortedInfo = sortedInfo || {}
      filteredInfo = filteredInfo || {}
      return [{
        title: 'User Name',
        dataIndex: 'username',
        sorter: true,
        sortOrder: sortedInfo.columnKey === 'username' && sortedInfo.order
      }, {
        title: 'Nick Name',
        dataIndex: 'nickName'
      }, {
        title: 'User Type',
        dataIndex: 'userType'
      }, {
        title: 'Status',
        dataIndex: 'status',
        customRender: (text, row, index) => {
          switch (text) {
            case '0': return <a-tag color="red"> Locked </a-tag>
            case '1': return <a-tag color="cyan"> Effective </a-tag>
            default: return text
          }
        },
        filters: [
          { text: 'Effective', value: '1' },
          { text: 'Locked', value: '0' }
        ],
        filterMultiple: false,
        filteredValue: filteredInfo.status || null,
        onFilter: (value, record) => value
      }, {
        title: 'Create Time',
        dataIndex: 'createTime',
        sorter: true,
        sortOrder: sortedInfo.columnKey === 'createTime' && sortedInfo.order
      },
        {
          title: 'Operation',
          dataIndex: 'operation',
          scopedSlots: { customRender: 'operation' }
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
    handleView (record) {
      this.userInfo.data = record
      this.userInfo.visible = true
    },
    handleAdd () {
      this.userAdd.visible = true
    },
    handleUserAddClose () {
      this.userAdd.visible = false
    },
    handleUserAddSuccess () {
      this.userAdd.visible = false
      this.$message.success('add user successfully')
      this.search()
    },
    handleEdit (record) {
      this.$refs.userEdit.setFormValues(record)
      this.userEdit.visible = true
    },
    handleUserEditClose () {
      this.userEdit.visible = false
    },
    handleUserEditSuccess () {
      this.userEdit.visible = false
      this.$message.success('modify user successfully')
      this.search()
    },
    handleUserInfoClose () {
      this.userInfo.visible = false
    },
    handleDateChange (value) {
      if (value) {
        this.queryParams.createTimeFrom = value[0]
        this.queryParams.createTimeTo = value[1]
      }
    },
    handleDelete (record) {
      deleteUser({
        userId: record.userId
      }).then((resp) => {
        if (resp.status === 'success') {
          this.$message.success('delete successful')
          this.search()
        } else {
          this.$message.error('delete failed')
        }
      })
    },

    resetPassword (user) {
      this.$swal.fire({
        title: 'reset password, are yor sure?',
        showCancelButton: true,
        confirmButtonText: `Yes`,
      }).then((result) => {
        if (result.isConfirmed) {
          resetPassword( {
            usernames: user.username
          }).then((resp) => {
            if (resp.status === 'success') {
              this.$swal.fire(
                'reset password successful, user ['+ user.username + '] new password is streampark666', '',
                'success'
              )
            }
          })
        }
      })
    },

    search () {
      const { sortedInfo, filteredInfo } = this
      let sortField, sortOrder
      // Get the sorting of the current column and the filter rules of the column
      if (sortedInfo) {
        sortField = sortedInfo.field
        sortOrder = sortedInfo.order
      }
      this.fetch({
        sortField: sortField,
        sortOrder: sortOrder,
        ...this.queryParams,
        ...filteredInfo
      })
    },
    reset () {
      // Reset pagination
      this.$refs.TableInfo.pagination.current = this.pagination.defaultCurrent
      if (this.paginationInfo) {
        this.paginationInfo.current = this.pagination.defaultCurrent
        this.paginationInfo.pageSize = this.pagination.defaultPageSize
      }
      // Reset filteredInfo
      this.filteredInfo = null
      // Reset sortedInfo
      this.sortedInfo = null
      // Reset queryParams
      this.queryParams = {}
      this.$refs.createTime.reset()
      this.fetch()
    },
    handleTableChange (pagination, filters, sorter) {
      this.paginationInfo = pagination
      this.filteredInfo = filters
      this.sortedInfo = sorter
      this.userInfo.visible = false
      this.fetch({
        sortField: sorter.field,
        sortOrder: sorter.order,
        ...this.queryParams,
        ...filters
      })
    },
    fetch (params = {}) {
      // show loading
      this.loading = true
      if (this.paginationInfo) {
        // If the paging information is not empty, set the current page of the table, the number of items per page, and set the query paging parameters
        this.$refs.TableInfo.pagination.current = this.paginationInfo.current
        this.$refs.TableInfo.pagination.pageSize = this.paginationInfo.pageSize
        params.pageSize = this.paginationInfo.pageSize
        params.pageNum = this.paginationInfo.current
      } else {
        // If pagination information is empty, set to default
        params.pageSize = this.pagination.defaultPageSize
        params.pageNum = this.pagination.defaultCurrent
      }
      if(params.status != null && params.status.length>0) {
        params.status = params.status[0]
      } else {
        delete params.status
      }

      if (params.sortField === 'createTime') {
        params.sortField = 'create_time'
      }

      list({ ...params }).then((resp) => {
        const pagination = { ...this.pagination }
        pagination.total = parseInt(resp.data.total)
        this.dataSource = resp.data.records
        this.pagination = pagination
        // After the data is loaded, close the loading
        this.loading = false
      })
    }
  }
}
</script>
