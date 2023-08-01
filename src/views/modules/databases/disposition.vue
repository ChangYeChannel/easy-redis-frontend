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
        <el-button :disabled="!flushButtonFlag" @click="getDispositionData()">刷新</el-button>
      </el-form-item>
    </el-form>
    <el-table
      :data="dataList"
      border
      v-loading="dataListLoading"
      style="width: 100%;">

      <el-table-column
        prop="key"
        header-align="center"
        align="center"
        width="500"
        label="配置项">
      </el-table-column>

      <el-table-column
        prop="value"
        header-align="center"
        align="center"
        label="值">
      </el-table-column>

    </el-table>
  </div>
</template>

<script>
export default {
  data () {
    return {
      ips: [],
      ports: [],
      dataForm: {
        ip: '',
        port: ''
      },
      // 刷新按钮启用标记
      flushButtonFlag: false,
      // 下拉框选项
      options: [],
      // 主体数据
      dataList: [],
      // 主体表格加载状态
      dataListLoading: false
    }
  },
  created () {
    // 初始化IP地址列表
    this.initIp()
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
          // 成功提醒
          this.$message({
            message: '连接成功',
            type: 'success',
            duration: 1500,
            onClose: () => {
              // 连接成功选中库数据
              this.getDispositionData()
            }
          })
        } else {
          // 连接失败
          this.$message.error('连接失败！请检查连接配置')
        }
      })
    },
    // 获取连接后页面数据
    getDispositionData () {
      this.dataListLoading = true
      this.$http({
        url: this.$http.adornUrl('/business/databases/dispositionList'),
        method: 'get'
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.flushButtonFlag = true
          this.dataList = data.datalist
        } else {
          this.dataList = []
        }
        this.dataListLoading = false
      })
    },
    // 当端口号发生变动的时候，清理数据
    cleanData () {
      this.flushButtonFlag = false
      this.dataList = []
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
