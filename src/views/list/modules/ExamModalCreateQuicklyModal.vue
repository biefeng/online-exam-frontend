<template>
  <a-modal title="创建考试" :width="640" :visible="visible" :confirmLoading="confirmLoading" @cancel="handleCancel">
    <a-spin :spinning="confirmLoading">
      <a-form :form="form">
        <!-- step1 -->
        <div>
          <a-form-item label="题库" :labelCol="labelCol" labelAlign="left" :wrapperCol="wrapperCol">
            <a-select :size="size" placeholder="请选择题库" style="width: 100%" v-decorator="['bank']">
              <a-select-option v-for="bank in banks" :value="bank.id" :key="bank.id">
                {{ bank.name }}
              </a-select-option>
            </a-select>
          </a-form-item>
        </div>
      </a-form>
    </a-spin>
    <template slot="footer">
      <a-button key="cancel" @click="handleCancel">取消</a-button>
      <a-button key="forward" :loading="confirmLoading" type="primary" @click="handleNext()">
        创建
      </a-button>
    </template>
  </a-modal>
</template>

<script>
import '../../../plugins/summernote'
import $ from 'jquery'
import { getExamQuestionTypeList, getExamQuestionBankList, examRandomCreate } from '../../../api/exam'

export default {
  name: 'StepByStepExamModal',
  data () {
    return {
      labelCol: {
        xs: { span: 24 },
        sm: { span: 7 }
      },
      size: 'default',
      wrapperCol: {
        xs: { span: 24 },
        sm: { span: 13 }
      },
      visible: false,
      confirmLoading: false,

      mdl: {},

      form: this.$form.createForm(this),

      banks: []
    }
  },
  updated () {
    this.initSummernote()
  },
  methods: {
    initSummernote () {
      console.log('初始化富文本插件')
      $('#summernote-exam-avatar-create').summernote({
        lang: 'zh-CN',
        placeholder: '粘贴截图到这即可，图片最好不要大于80*80',
        height: 200,
        width: 320,
        htmlMode: true,
        toolbar: [],
        fontSizes: ['8', '9', '10', '11', '12', '14', '18', '24', '36'],
        fontNames: [
          'Arial',
          'Arial Black',
          'Comic Sans MS',
          'Courier New',
          'Helvetica Neue',
          'Helvetica',
          'Impact',
          'Lucida Grande',
          'Tahoma',
          'Times New Roman',
          'Verdana'
        ]
      })
    },
    create () {
      this.visible = true
      // 从后端数据获取单选题、多选题和判断题的列表
      getExamQuestionTypeList()
        .then(res => {
          console.log(res)
          if (res.code === 0) {
            console.log(res.data)
            this.radios = res.data.radios
            this.checks = res.data.checks
            this.judges = res.data.judges
          } else {
            this.$notification.error({
              message: '获取问题列表失败',
              description: res.msg
            })
          }
        })
        .catch(err => {
          // 失败就弹出警告消息
          this.$notification.error({
            message: '获取题库列表失败',
            description: err.message
          })
        })
      getExamQuestionBankList().then(res => {
        if (res.code === 0) {
          console.log(res.data)
          this.banks = res.data
        } else {
          this.$notification.error({
            message: '获取问题列表失败',
            description: res.msg
          })
        }
      })
    },
    popupScroll () {
      console.log('popupScroll')
    },
    handleNext (step) {
      // 处理下一步或者完成事件
      const {
        form: { validateFields }
      } = this

      // last step，最后一步，代表完成考试编写，需要点击"完成"创建考试
      this.confirmLoading = true
      validateFields((errors, values) => {
        console.log('提交数据到后端')
        this.confirmLoading = false
        if (!errors) {
          // 在这里把创建的考试的内容(存放在values中)提交给后端接口，需要的参数都已经封装成values这个json啦
          console.log('values:', values)
          // 把data中的question属性提交到后端，待写完后端接口.把前端的创建的题型提交到后端
          examRandomCreate(values)
            .then(res => {
              // 成功就跳转到结果页面
              console.log(res)
              if (res.code === 0) {
                this.$notification.success({
                  message: '创建成功',
                  description: '考试创建成功'
                })
                // 关闭弹出框
                this.visible = false
                this.$emit('ok')
              }
            })
            .catch(err => {
              // 失败就弹出警告消息
              this.$notification.error({
                message: '考试创建失败',
                description: err.message
              })
            })
        } else {
          this.confirmLoading = false
        }
      })
    },
    backward () {
      
    },
    handleCancel () {
      // clear form & currentStep
      this.visible = false
      this.currentStep = 0
    },
    // 改变选择的题目列表,这里需要分单选、多选和判断进行单独更新，下面的代码要针对radios、checks和judges分别适配
    handleRadioChange (values) {
      console.log(values)
      // 更新单选题的信息
      for (let i = 0; i < this.radios.length; i++) {
        // 遍历所有的题目的选项
        // 取出一个选项的id
        const name = this.radios[i].name
        // 当前问题是否被问题创建者选中
        let checked = false
        for (let j = 0; j < values.length; j++) {
          // 拿着
          const value = values[j]
          if (name === value) {
            // 说明这个问题被考试创建者选中
            checked = true
            this.radios[i].checked = true
            break // 跳出内部的for循环
          }
        }
        // 这个选项遍历到最后，发现还不是答案(不在答案数组中)，那么就把这个选项的answer属性设置为false
        if (checked === false) {
          this.radios[i].checked = false
        }
      }
    },

    // 更新多选题信息
    handleCheckChange (values) {
      console.log(values)
      // 更新单选题的信息
      for (let i = 0; i < this.checks.length; i++) {
        // 遍历所有的题目的选项
        // 取出一个选项的id
        const name = this.checks[i].name
        // 当前问题是否被问题创建者选中
        let checked = false
        for (let j = 0; j < values.length; j++) {
          // 拿着
          const value = values[j]
          if (name === value) {
            // 说明这个问题被考试创建者选中
            checked = true
            this.checks[i].checked = true
            break // 跳出内部的for循环
          }
        }
        // 这个选项遍历到最后，发现还不是答案(不在答案数组中)，那么就把这个选项的answer属性设置为false
        if (checked === false) {
          this.checks[i].checked = false
        }
      }
    },

    // 更新判断题信息
    handleJudgeChange (values) {
      console.log(values)
      // 更新单选题的信息
      for (let i = 0; i < this.judges.length; i++) {
        // 遍历所有的题目的选项
        // 取出一个选项的id
        const name = this.judges[i].name
        // 当前问题是否被问题创建者选中
        let checked = false
        for (let j = 0; j < values.length; j++) {
          // 拿着
          const value = values[j]
          if (name === value) {
            // 说明这个问题被考试创建者选中
            checked = true
            this.judges[i].checked = true
            break // 跳出内部的for循环
          }
        }
        // 这个选项遍历到最后，发现还不是答案(不在答案数组中)，那么就把这个选项的answer属性设置为false
        if (checked === false) {
          this.judges[i].checked = false
        }
      }
    }
  }
}
</script>
