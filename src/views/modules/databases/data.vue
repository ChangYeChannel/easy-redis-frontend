<template>
  <div>
    <el-form :inline="true" :model="dataForm" @keyup.enter.native="connectServer()">
      <el-form-item>
        <el-select v-model="dataForm.ip" placeholder="请选择IP地址" @change="showPort()">
          <el-option v-for="option in this.ips" :key="option" :label="option" :value="option"></el-option>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-select v-model="dataForm.port" placeholder="请选择端口">
          <el-option v-for="option in this.ports" :key="option" :label="option" :value="option"></el-option>
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button @click="connectServer()">连接</el-button>
      </el-form-item>
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
        width="95"
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
        fixed="right"
        header-align="center"
        align="center"
        width="180"
        label="操作">
        <template slot-scope="scope">
          <el-button type="text" size="small" @click="connect(scope.row.ip,scope.row.port)">连接</el-button>
          <el-button type="text" size="small" @click="addOrUpdateHandle(scope.row.id)">修改</el-button>
          <el-button type="text" size="small" @click="deleteHandle(scope.row.id)">删除</el-button>
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
  </div>
</template>

<script>
import router from '../../../router'

export default {
  data () {
    return {
      ips: [],
      ports: [],
      dataForm: {
        ip: '',
        port: ''
      },
      dataList: [],
      pageIndex: 1,
      pageSize: 10,
      totalPage: 0,
      dataListLoading: false,
      dataListSelections: []
    }
  },
  created () {
    // 先初始化IP地址列表
    this.initIp()
    // 将路由参数绑定到查询条件
    this.dataForm.ip = this.$route.params.ip
    // 因为上面语句的绑定，手动刷新以下端口列表
    this.showPort()
    // 将路由参数绑定到查询条件
    this.dataForm.port = this.$route.params.port
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
          // 连接成功
          this.$message({
            message: '连接成功',
            type: 'success',
            duration: 1500,
            onClose: () => {
              this.getConnectedData()
            }
          })
        } else {
          // 连接失败
          this.$message.error('连接失败！请检查连接配置')
        }
      })
    },
    // 获取连接后页面数据
    getConnectedData () {
      this.dataListLoading = true
      this.$http({
        url: this.$http.adornUrl('/business/databases/list'),
        method: 'get'
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.dataList = data.page.list
          this.totalPage = data.page.totalCount
        } else {
          this.dataList = []
          this.totalPage = 0
        }
        this.dataListLoading = false
      })
    },
    // 每页数
    sizeChangeHandle (val) {
      this.pageSize = val
      this.pageIndex = 1
      this.getDataList()
    },
    // 当前页
    currentChangeHandle (val) {
      this.pageIndex = val
      this.getDataList()
    },
    // 多选
    selectionChangeHandle (val) {
      this.dataListSelections = val
    },
    // 新增 / 修改
    addOrUpdateHandle (id) {
      this.addOrUpdateVisible = true
      this.$nextTick(() => {
        this.$refs.addOrUpdate.init(id)
      })
    },
    // 删除
    deleteHandle (id) {
      var ids = id ? [id] : this.dataListSelections.map(item => {
        return item.id
      })
      this.$confirm(`确定对[id=${ids.join(',')}]进行[${id ? '删除' : '批量删除'}]操作?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl('/business/base/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.$message({
              message: '操作成功',
              type: 'success',
              duration: 1500,
              onClose: () => {
                this.getDataList()
              }
            })
          } else {
            this.$message.error(data.msg)
          }
        })
      }).catch(() => {
      })
    },
    // 连接当前选中数据库
    connect (ip, port) {
      console.log(`连接到 IP: ${ip}, 端口: ${port}`)
      router.push({name: 'databases-data', params: {ip, port}})
    }
  }
}
</script>
