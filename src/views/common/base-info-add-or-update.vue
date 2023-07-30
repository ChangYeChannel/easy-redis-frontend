<template>
  <el-dialog
    :title="!dataForm.id ? '新增' : '修改'"
    :close-on-click-modal="false"
    :visible.sync="visible">
    <el-form :model="dataForm" :rules="dataRule" ref="dataForm" @keyup.enter.native="dataFormSubmit()"
             label-width="80px">
      <el-form-item label="别名" prop="name">
        <el-input v-model="dataForm.name" placeholder="数据库别名"></el-input>
      </el-form-item>
      <el-form-item label="IP地址" prop="ip">
        <el-input v-model="dataForm.ip" placeholder="IP地址" @blur="handleIpChange"></el-input>
      </el-form-item>
      <el-form-item label="端口号" prop="port">
        <el-input v-model="dataForm.port" placeholder="端口号" :disabled="!this.isIpValid" @blur="checkPort()"></el-input>
      </el-form-item>
      <el-form-item label="有无密码" size="mini" prop="isPassword">
        <el-radio-group v-model="dataForm.isPassword">
          <el-radio :label="0">无密码</el-radio>
          <el-radio :label="1">有密码</el-radio>
        </el-radio-group>
      </el-form-item>
      <el-form-item label="密码" prop="password" v-if="dataForm.isPassword === 1">
        <el-input v-model="dataForm.password" type="password" placeholder="密码"></el-input>
      </el-form-item>
      <el-form-item label="备注" prop="remark">
        <el-input v-model="dataForm.remark" placeholder="备注"></el-input>
      </el-form-item>
    </el-form>
    <span slot="footer" class="dialog-footer">
      <el-button @click="visible = false">取消</el-button>
      <el-button type="primary" @click="beforSubmit()">确定</el-button>
    </span>
  </el-dialog>
</template>

<script>
export default {
  data () {
    return {
      // 控制表单是否显示
      visible: false,
      // 数据表单
      dataForm: {
        id: 0,
        name: '',
        ip: '',
        port: '',
        isPassword: 1,
        password: '',
        remark: ''
      },
      // 输入时判断端口是否已经占用
      isExits: false,
      // 控制如果IP地址未输入，则无法输入端口号
      isIpValid: false,
      // 表单项校验
      dataRule: {
        name: [
          {required: true, message: '数据库别名不能为空', trigger: 'blur'}
        ],
        ip: [
          {required: true, message: 'IP地址不能为空', trigger: 'blur'}
        ],
        port: [
          {required: true, message: '端口号不能为空', trigger: 'blur'}
        ],
        password: [
          {required: true, message: '密码不能为空', trigger: 'blur'}
        ]
      }
    }
  },
  methods: {
    // 初始化
    init (id) {
      this.dataForm = {
        id: 0,
        name: '',
        ip: '',
        port: '',
        isPassword: 1,
        password: '',
        remark: ''
      }
      this.visible = true
      this.dataForm.id = id || 0
      if (this.dataForm.id) {
        this.$http({
          url: this.$http.adornUrl(`/business/base/info/${this.dataForm.id}`),
          method: 'get',
          params: this.$http.adornParams()
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.dataForm.name = data.baseInfo.name
            this.dataForm.ip = data.baseInfo.ip
            this.dataForm.port = data.baseInfo.port
            this.dataForm.isPassword = data.baseInfo.isPassword
            this.dataForm.password = data.baseInfo.password
            this.dataForm.remark = data.baseInfo.remark
          }
        })
      }
    },
    // 控制先填写IP地址才能填写端口号操作
    handleIpChange () {
      // 检查 IP 地址是否填写
      this.isIpValid = this.dataForm.ip !== ''
    },
    // 检查输入的端口号是否在数据库中已经存在
    checkPort () {
      this.$http({
        url: this.$http.adornUrl(`/business/base/checkPort`),
        method: 'get',
        params: this.$http.adornParams({
          'ip': this.dataForm.ip,
          'port': this.dataForm.port
        })
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.isExits = data.isExits
        }
      })
    },
    // 提交前检查
    beforSubmit () {
      // 在提交表单时，检查 this.isExits 的值
      if (this.isExits) {
        // 如果为 true，给出界面警告
        this.$message.error('当前IP和端口号的组合在数据库中已经存在，请勿重复添加')
        // 终止表单提交行为
        return false
      } else {
        // 如果校验通过，提交表单数据到后端
        this.dataFormSubmit()
      }
    },
    // 表单提交
    dataFormSubmit () {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          this.$http({
            url: this.$http.adornUrl(`/business/base/${!this.dataForm.id ? 'save' : 'update'}`),
            method: 'post',
            data: this.$http.adornData({
              'id': this.dataForm.id || undefined,
              'name': this.dataForm.name,
              'ip': this.dataForm.ip,
              'port': this.dataForm.port,
              'isPassword': this.dataForm.isPassword,
              'password': this.dataForm.password,
              'remark': this.dataForm.remark
            })
          }).then(({data}) => {
            if (data && data.code === 0) {
              this.$message({
                message: '操作成功',
                type: 'success',
                duration: 1500,
                onClose: () => {
                  this.visible = false
                  this.$emit('refreshDataList')
                }
              })
            } else {
              this.$message.error(data.msg)
            }
          })
        }
      })
    }
  }
}
</script>
