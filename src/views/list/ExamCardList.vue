<template>
  <div class="card-list" ref="content">
    <a-collapse :activeKey="activeKey">
      <a-collapse-panel key="1" header="自定义考试">
        <a-form :form="form" name="basic">
          <a-row :gutter="48">
            <a-col :md="8" :sm="24">
              <a-form-item>
                <a-select v-model="form.questionBankId" placeholder="请选择题库">
                  <a-select-option v-for="bank in banks" :key="bank.id" :value="bank.id">
                    {{ bank.name }}
                  </a-select-option>
                </a-select>
              </a-form-item>
            </a-col>
            <a-col :md="4" :sm="12">
              <a-form-item> 随机选题&nbsp;<a-switch v-model="form.random"></a-switch> </a-form-item>
            </a-col>
            <template v-if="advanced">
              <a-col :md="8" :sm="24" v-if="form.random">
                <a-form-item>
                  <a-input-number
                    v-model="form.radioCount"
                    :defaultValue="5"
                    style="width: 100%"
                    placeholder="单选题数量"
                  />
                </a-form-item>
              </a-col>
              <a-col :md="8" :sm="24" v-if="form.random">
                <a-form-item>
                  <a-input-number
                    v-model="form.checkCount"
                    :defaultValue="5"
                    style="width: 100%"
                    placeholder="多选题数量"
                  />
                </a-form-item>
              </a-col>
              <a-col :md="8" :sm="24" v-if="form.random">
                <a-form-item>
                  <a-input-number
                    v-model="form.judgeCount"
                    :defaultValue="5"
                    style="width: 100%"
                    placeholder="判断题数量"
                  />
                </a-form-item>
              </a-col>
            </template>
            <a-col :md="(!advanced && 8) || 24" :sm="24">
              <span
                class="table-page-search-submitButtons"
                :style="(advanced && { float: 'right', overflow: 'hidden' }) || {}"
              >
                <a-button type="primary" @click="randExam">考试</a-button>
                <a-button style="margin-left: 8px" @click="() => (this.form = {})">重置</a-button>
                <a @click="toggleAdvanced" style="margin-left: 8px">
                  {{ advanced ? '收起' : '展开' }}
                  <a-icon :type="advanced ? 'up' : 'down'" />
                </a>
              </span>
            </a-col>
          </a-row>
        </a-form>
      </a-collapse-panel>
      <a-collapse-panel key="2" header="考试列表">
        <a-list :grid="{ gutter: 24, lg: 3, md: 2, sm: 1, xs: 1 }" :dataSource="dataSource">
          <a-list-item slot="renderItem" slot-scope="item">
            <a-card :hoverable="true" @click="joinExam(item.id)">
              <a-card-meta>
                <div style="margin-bottom: 3px" slot="title">{{ item.title }}</div>
                <a-avatar class="card-avatar" slot="avatar" :src="item.avatar | imgSrcFilter" size="large" />
                <div class="meta-content" slot="description">{{ item.content }}</div>
              </a-card-meta>
              <template class="ant-card-actions" slot="actions">
                <a>满分：{{ item.score }}分</a>
                <a>限时：{{ item.elapse }}分钟</a>
              </template>
            </a-card>
          </a-list-item>
        </a-list>
      </a-collapse-panel>
    </a-collapse>
  </div>
</template>

<script>
import { getExamQuestionBankList, getExamCardList, examRandomCreate } from '@api/exam'
export default {
  name: 'ExamCardList',

  data () {
    return {
      dataSource: [],
      banks: [],
      activeKey: ['1', '2'],
      form: {},
      advanced: false,
      queryParam: {}
    }
  },
  methods: {
    joinExam (id) {
      const routeUrl = this.$router.resolve({
        path: `/exam/${id}`
      })
      window.open(routeUrl.href, '_blank')
    },
    onChnage () {
      console.log('1')
    },
    toggleAdvanced () {
      this.advanced = !this.advanced
    },
    randExam () {
      examRandomCreate(this.form).then(res => {
        if (res.code === 0) {
          this.joinExam(res.data.examId)
        }
      })
    }
  },
  mounted () {
    // 从后端数据获取考试列表，适配前端卡片
    getExamCardList()
      .then(res => {
        console.log(res)
        if (res.code === 0) {
          this.dataSource = res.data
        } else {
          this.$notification.error({
            message: '获取考试列表失败',
            description: res.msg
          })
        }
      })
      .catch(err => {
        // 失败就弹出警告消息
        this.$notification.error({
          message: '获取考试列表失败',
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
  }
}
</script>

<style lang="less" scoped>
.card-avatar {
  width: 48px;
  height: 48px;
  border-radius: 48px;
}

.ant-card-actions {
  background: #f7f9fa;

  li {
    float: left;
    text-align: center;
    margin: 12px 0;
    color: rgba(0, 0, 0, 0.45);
    width: 50%;

    &:not(:last-child) {
      border-right: 1px solid #e8e8e8;
    }

    a {
      color: rgba(0, 0, 0, 0.45);
      line-height: 22px;
      display: inline-block;
      width: 100%;

      &:hover {
        color: #1890ff;
      }
    }
  }
}

.new-btn {
  background-color: #fff;
  border-radius: 2px;
  width: 100%;
  height: 188px;
}

.meta-content {
  position: relative;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  height: 64px;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
}
</style>
