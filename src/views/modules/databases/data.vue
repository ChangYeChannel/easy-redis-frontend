<template>
  <div>
    <el-form :inline="true" :model="dataForm" @keyup.enter.native="connectServer()">
      <el-form-item>
        <el-select v-model="dataForm.ip" placeholder="请选择IP地址" @change="showPort()">
          <el-option v-for="option in this.ips" :key="option" :label="option" :value="option"></el-option>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-select v-model="dataForm.port" placeholder="请选择端口" @change="cleanData()">
          <el-option v-for="option in this.ports" :key="option" :label="option" :value="option"></el-option>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button @click="connectServer()">连接</el-button>
        <el-button type="primary" :disabled="!dataForm.datasource" @click="addOrUpdateHandle(dataForm.datasource)">
          新增数据
        </el-button>
        <el-button type="danger" @click="deleteHandle()" :disabled="dataListSelections.length <= 0">批量删除</el-button>
      </el-form-item>
      <div class="new-line">
        <el-form-item>
          <el-select v-model="dataForm.datasource" placeholder="请选择数据库" @change="getConnectedData()">
            <el-option v-for="(option) in options" :key="option.value" :label="option.label" :value="option.value"/>
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-button :disabled="!dataForm.datasource" @click="getConnectedData()">刷新</el-button>
        </el-form-item>
        <el-form-item>
          <el-input v-model="dataForm.likeKey" placeholder="模糊查询" clearable></el-input>
        </el-form-item>
        <el-form-item>
          <el-button :disabled="!dataForm.likeKey" @click="getConnectedDataByLike()">查询</el-button>
        </el-form-item>
      </div>
    </el-form>
    <el-table
      :data="dataList"
      border
      v-loading="dataListLoading"
      @selection-change="selectionChangeHandle"
      style="width: 100%;">
      <el-table-column
        type="selection"
        header-align="center"
        align="center"
        width="50">
      </el-table-column>
      <el-table-column
        prop="key"
        header-align="center"
        align="center"
        width="400"
        label="key">
      </el-table-column>
      <el-table-column
        prop="type"
        header-align="center"
        align="center"
        label="type">
      </el-table-column>
      <el-table-column
        prop="value"
        header-align="center"
        align="center"
        width="500"
        label="value">
      </el-table-column>
      <el-table-column
        prop="ttl"
        header-align="center"
        align="center"
        width="300"
        label="剩余过期时间（秒）">
        <template slot-scope="{ row }">
          <span>
            <!-- 根据 row.ttl 的值来动态展示对应的文本 -->
            {{ row.ttl === -1 ? '永不过期' : (row.ttl === -99 ? '当前键已经过期' : row.ttl) }}
          </span>
        </template>
      </el-table-column>
      <el-table-column
        fixed="right"
        header-align="center"
        align="center"
        width="180"
        label="操作">
        <template slot-scope="scope">
          <el-button type="text" size="small" @click="addOrUpdateHandle(dataForm.datasource,scope.row.key)">修改
          </el-button>
          <el-button type="text" size="small" @click="deleteHandle(scope.row.key)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination
      @size-change="sizeChangeHandle"
      @current-change="currentChangeHandle"
      :current-page="pageIndex"
      :page-sizes="[10, 20, 50, 100]"
      :page-size="pageSize"
      :total="totalPage"
      layout="total, sizes, prev, pager, next, jumper">
    </el-pagination>
    <add-or-update ref="addOrUpdate" @refreshDataList="getConnectedData()"></add-or-update>
  </div>
</template>

<script>
import AddOrUpdate from './data-add-or-update.vue'

export default {
  data () {
    return {
      ips: [],
      ports: [],
      dataForm: {
        ip: '',
        port: '',
        datasource: '',
        likeKey: ''
      },
      pageIndex: 1,
      pageSize: 10,
      totalPage: 0,
      dataCount: 0,
      // 数据库选择下拉框
      options: [],
      dataList: [],
      dataListLoading: false,
      dataListSelections: [],
      addOrUpdateVisible: false
    }
  },
  created () {
    // 先初始化IP地址列表
    this.initIp()
    // 将路由参数绑定到查询条件
    this.dataForm.ip = this.$route.params.ip
    if (this.dataForm.ip !== '' && this.dataForm.ip !== undefined) {
      // 因为上面语句的绑定，手动刷新以下端口列表
      this.showPort()
      // 将路由参数绑定到查询条件
      this.dataForm.port = this.$route.params.port
      if (this.dataForm.port !== '' && this.dataForm.port !== undefined) {
        // 连接数据库
        this.connectServer()
      }
    }
  },
  components: {
    AddOrUpdate
  },
  methods: {
    // 初始化查询表单框的Ip地址框
    initIp () {
      this.$http({
        url: this.$http.adornUrl('/business/base/init'),
        method: 'get',
        params: this.$http.adornParams()
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.ips = data.ips
        } else {
          this.ips = []
        }
      })
    },
    // 根据当前选中的Ip地址，变更端口选择框
    showPort () {
      this.$http({
        url: this.$http.adornUrl('/business/base/showPort'),
        method: 'get',
        params: this.$http.adornParams({
          'ip': this.dataForm.ip
        })
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.ports = data.ports
        } else {
          this.ports = []
        }
      })
    },
    // 连接当前选中的服务
    connectServer () {
      this.$http({
        url: this.$http.adornUrl('/business/databases/connectServer'),
        method: 'get',
        params: this.$http.adornParams({
          'ip': this.dataForm.ip,
          'port': this.dataForm.port
        })
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.showCount(data)
          // 成功提醒
          this.$message({
            message: '连接成功',
            type: 'success',
            duration: 1500,
            onClose: () => {
              // 连接成功选中库数据
              this.getConnectedData()
            }
          })
        } else {
          // 连接失败
          this.$message.error('连接失败！请检查连接配置')
        }
      })
    },
    // 渲染当前连接的Redis服务器中的数据库数量
    showCount (data) {
      // 清空当前数据库数量列表
      this.options = []
      // 连接建立成功，拿到当前连接的Redis服务器中的数据库数量
      this.dataCount = data.count
      // 解析数量，将其渲染到下拉列表
      for (let i = 0; i < this.dataCount; i++) {
        this.options.push({value: i.toString(), label: i + '号库'})
      }
      // 渲染完成后默认选中第一个
      if (this.options.length > 0) {
        this.dataForm.datasource = this.options[0].value
      }
    },
    // 获取连接后页面数据
    getConnectedData () {
      this.dataListLoading = true
      this.$http({
        url: this.$http.adornUrl('/business/databases/list'),
        method: 'get',
        params: this.$http.adornParams({
          'dataSource': this.dataForm.datasource,
          'page': this.pageIndex,
          'limit': this.pageSize,
          'likeKey': this.dataForm.likeKey
        })
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.dataList = data.datalist.list
          this.totalPage = data.datalist.totalCount
        } else {
          this.dataList = []
          this.totalPage = 0
        }
        this.dataListLoading = false
      })
    },
    // 模糊查询
    getConnectedDataByLike () {
      this.pageIndex = 1
      this.getConnectedData()
    },
    // 每页数
    sizeChangeHandle (val) {
      this.pageSize = val
      this.pageIndex = 1
      this.getConnectedData()
    },
    // 当前页
    currentChangeHandle (val) {
      this.pageIndex = val
      this.getConnectedData()
    },
    // 多选
    selectionChangeHandle (val) {
      this.dataListSelections = val
    },
    // 新增 / 修改
    addOrUpdateHandle (datasource, key) {
      this.checkConnected()
      this.addOrUpdateVisible = true
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init(datasource, key)
      })
    },
    // 当端口号发生变动的时候，清理数据
    cleanData () {
      this.dataForm.datasource = ''
      this.dataForm.likeKey = ''
      this.dataList = []
      this.options = []
    },
    // 删除
    deleteHandle (id) {
      this.checkConnected()
      var ids = id ? [id] : this.dataListSelections.map(item => {
        return item.key
      })
      this.$confirm(`确定对[id=${ids.join(',')}]进行[${id ? '删除' : '批量删除'}]操作?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 将当前执行删除操作的库追加到请求参数中
        ids.unshift(this.dataForm.datasource)
        this.$http({
          url: this.$http.adornUrl('/business/databases/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.$message({
              message: data.message,
              duration: 1500,
              onClose: () => {
                this.getConnectedData()
              }
            })
          }
        })
      })
    },
    // 检查当前连接情况
    checkConnected () {
      this.$http({
        url: this.$http.adornUrl('/business/databases/checkConnected'),
        method: 'get'
      }).then(({data}) => {
        if (data.code === 500) {
          // 连接失败
          this.$message.error('与redis服务连接已断开，正在重新建立连接')
          this.addOrUpdateVisible = false
          this.connectServer()
        }
      })
    }
  }
}
</script>
<style>
.new-line {
  display: block;
  margin-top: 10px;
}
</style>
