<template>
  <el-dialog
    :title="!dataForm.key ? '新增' : '修改'"
    :close-on-click-modal="false"
    :visible.sync="visible">
    <el-form :model="dataForm" :rules="dataRules" ref="dataForm" @keyup.enter.native="dataFormSubmit()"
             label-width="80px">

      <el-form-item label="Key" prop="Key">
        <el-input v-model="dataForm.key" placeholder="Key" :disabled="keyDisable" @change="checkKey()"></el-input>
      </el-form-item>

      <el-form-item label="Type" size="mini" prop="Type">
        <el-radio-group v-model="dataForm.type">
          <el-radio :label="'0'">string</el-radio>
          <el-radio :label="'1'">list</el-radio>
          <el-radio :label="'2'">set</el-radio>
          <el-radio :label="'3'">zset</el-radio>
          <el-radio :label="'4'">hash</el-radio>
        </el-radio-group>
      </el-form-item>

      <el-form-item label="Value" prop="Value">
        <el-input v-model="dataForm.value" :placeholder="placeholder"></el-input>
      </el-form-item>


      <el-form-item label="设置TTL" size="mini" prop="isTTL">
        <el-radio-group v-model="dataForm.isTTL">
          <el-radio :label="'0'">不设置</el-radio>
          <el-radio :label="'1'">设置</el-radio>
        </el-radio-group>
      </el-form-item>

      <el-form-item label="TTL" prop="TTL" v-if="dataForm.isTTL === '1'">
        <el-input v-model="dataForm.ttl" placeholder="过期时间"></el-input>
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
  data() {
    return {
      // 控制表单是否显示
      visible: false,
      // 数据表单
      dataForm: {
        key: '',
        value: '',
        isTTL: '0',
        ttl: 0,
        type: '0'
      },
      datasource: '',
      // 输入时判断键是否已经存在
      isExits: false,
      // 输入时判断键是否已经存在
      keyDisable: false,
      // 表单项校验
      dataRules: {
        Key: [
          {required: true, message: '键不能为空', trigger: 'blur'}
        ],
        Value: [
          {required: true, message: '值不能为空', trigger: 'blur'}
        ],
        TTL: [
          {required: true, message: '过期时间不能为空', trigger: 'blur'},
          {
            validator: (rule, value, callback) => {
              if (!value) {
                callback(new Error('请填写过期时间'));
              } else {
                const trimmedValue = value.toString().trim(); // 转换为字符串后再去除空格
                const intValue = parseInt(trimmedValue, 10);
                if (isNaN(intValue) || intValue <= 0) {
                  callback(new Error('请输入大于0的整数'));
                } else {
                  callback();
                }
              }
            },
            trigger: 'blur'
          }
        ]
      }
    }
  },
  computed: {
    placeholder() {
      switch (this.dataForm.type) {
        case '0':
          return '请输入字符串类型数据，例如：hello';
        case '1':
          return '请输入集合类型数据，例如：[hello,world]';
        case '2':
          return '请输入集合类型数据并保证每个元素不同，例如：[hello,world]';
        case '3':
          return '请输入集合类型数据，保证每个元素不同并附加分数，例如：[hello:1,world:2]';
        case '4':
          return '请输入Hash类型数据，例如：{hello=world,hello-1=world-1}';
        default:
          return '';
      }
    }
  },
  methods: {
    // 初始化
    init(datasource, key) {
      this.dataForm = {
        key: '',
        value: '',
        isTTL: '0',
        ttl: 0,
        type: '0'
      }
      this.datasource = datasource
      this.visible = true
      this.dataForm.key = key || ''
      if (this.dataForm.key !== '') {
        this.keyDisable = true
        this.$http({
          url: this.$http.adornUrl(`/business/databases/info/${this.dataForm.key}`),
          method: 'get',
          params: this.$http.adornParams({
            'dataSource': this.datasource
          })
        }).then(({data}) => {
          if (data && data.code === 0) {
            // 回显数据
            if (data.RedisResponse.isTTL !== '-1') {
              this.dataForm.value = data.RedisResponse.value
              this.dataForm.isTTL = data.RedisResponse.isTTL
              this.dataForm.ttl = data.RedisResponse.ttl
              this.dataForm.type = data.RedisResponse.type
            } else {
              // 当前键已经过期，关闭弹框，给出用户刷新提示
              this.visible = false
              this.$message.error('当前键值对已经过期，请刷新数据后重新选择！')
            }
          }
        })
      }
    },
    // 检查输入的端口号是否在数据库中已经存在
    checkKey() {
      this.$http({
        url: this.$http.adornUrl(`/business/databases/checkKey`),
        method: 'get',
        params: this.$http.adornParams({
          'dataSource': this.datasource,
          'key': this.dataForm.key
        })
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.isExits = data.isExits
        }
      })
    },
    // 提交前检查
    beforSubmit() {
      if (this.isExits) {
        this.$confirm(`当前Key已在服务器中存在，新值将会覆写旧值，是否继续?`, '提示', {
          confirmButtonText: '继续',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          // 提交表单数据
          this.dataFormSubmit()
        }).catch(() => {
          // 终止表单提交
          return false
        });
      } else {
        // 如果校验通过，提交表单数据到后端
        this.dataFormSubmit()
      }
    },
    // 表单提交
    dataFormSubmit() {
      this.$http({
        url: this.$http.adornUrl(`/business/databases/save`),
        method: 'post',
        data: this.$http.adornData({
          'datasource': this.datasource,
          'key': this.dataForm.key,
          'value': this.dataForm.name,
          'type': this.dataForm.ip,
          'isTTL': this.dataForm.ip,
          'ttl': this.dataForm.port
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
    },
    handleTypeChange(value) {
      this.dataForm.value = [];
    },
    addValue() {
      if (['1', '2', '3'].includes(this.dataForm.type)) {
        this.dataForm.value.push({val: ''});
      } else if (this.dataForm.type === '4') {
        this.dataForm.value.push({key: '', val: ''});
      }
    },
    removeValue(index) {
      this.dataForm.value.splice(index, 1);
    }
  }
}
</script>
