<template>
  <a-layout style="min-height: 100vh">
    <a-layout-header class="header" style="color: #fff">
      <!--   v-if="examDetail.exam" 是为了防止 异步请求时页面渲染的时候还没有拿到这个值而报错， 下面多处这个判断都是这个道理 -->
      <span style="font-size: 15px; margin-left: 0px" v-if="examDetail.exam">
        <a-avatar slot="avatar" size="large" shape="circle" :src="examDetail.exam.examAvatar | imgSrcFilter" />
        开始时间：{{ examDetail.exam.examStartDate }}
      </span>
    </a-layout-header>
    <a-layout>
      <a-layout>
        <a-layout-content class="main-ctn">
          <div style="display: flex; justify-content: center">
            <div class="question-ids-panel" :class="questionIdsPanel">
              <div
                v-for="(q, index) in questionsIds"
                :key="index"
                @click="getQuestionDetail(index)"
                :class="questionIdsPanelClass(index)"
              >
                <a-button
                  type="primary"
                  v-if="resultsMap !== '' && resultsMap.get(q) === 'True'"
                  icon="check"
                  shape="circle"
                />
                <a-button type="info" style="background:red;color:white" v-else icon="close" shape="circle" />
              </div>
            </div>
            <div class="question-panel">
              <span v-if="currentQuestion !== ''">
                <strong>{{ currentQuestion.type }} </strong>
                <p v-html="currentQuestion.name"></p>
                <strong style="color: green" v-if="questionRight">本题您答对啦！</strong>
                <strong style="color: red" v-if="!questionRight">本题您答错啦！</strong>
              </span>
              <br /><br />
              <!-- 单选题和判断题 -->
              <!-- key不重复只需要在一个for循环中保证即可 -->
              <a-radio-group
                v-model="radioValue"
                v-if="currentQuestion.type === '单选题' || currentQuestion.type === '判断题'"
              >
                <a-radio
                  disabled
                  v-for="option in currentQuestion.options"
                  :key="option.questionOptionId"
                  :style="optionStyle"
                  :value="option.questionOptionId"
                >
                  <span>
                    <span v-if="currentQuestion.type === '单选题'">{{ option.questionOptionKey + '.' }}</span>
                    {{ option.questionOptionContent }}
                  </span>
                </a-radio>
              </a-radio-group>

              <!-- 题目出错的时候才显示这块 -->
              <div
                v-if="
                  !questionRight &&
                  currentQuestion !== '' &&
                  (currentQuestion.type === '单选题' || currentQuestion.type === '判断题')
                "
              >
                <span style="color: red"><br />正确答案是：{{ answerRightOptions }}<br /></span>
              </div>

              <!-- 多选题 -->
              <a-checkbox-group v-model="checkValues" v-if="currentQuestion.type === '多选题'">
                <a-checkbox
                  disabled
                  v-for="option in currentQuestion.options"
                  :key="option.questionOptionId"
                  :style="optionStyle"
                  :value="option.questionOptionId"
                >
                  <span>
                    <span>{{ option.questionOptionKey + '.' }}</span>
                    {{ option.questionOptionContent }}
                  </span>
                </a-checkbox>
              </a-checkbox-group>

              <!-- 题目出错的时候才显示这块 -->
              <div v-if="!questionRight && currentQuestion !== '' && currentQuestion.type === '多选题'">
                <span style="color: red"><br />正确答案是：{{ answerRightOptions }}<br /></span>
              </div>

              <span style="color: red"><br />答案解析：<br /></span>
              <p v-html="currentQuestion.description"></p>
            </div>
          </div>
        </a-layout-content>
        <a-layout-footer :style="{ textAlign: 'end' }">
          <a-button type="primary" @click="pre">上一题</a-button>
          &nbsp;
          <a-button type="primary" @click="next">下一题</a-button>
          &nbsp;

          <a-button type="primary" @click="toggleQuesIdsPanel" class="qustion-ids-panel-togg-btn" icon="appstore" />
        </a-layout-footer>
      </a-layout>
    </a-layout>
  </a-layout>
</template>

<script>
import { getExamDetail, getQuestionDetail, getExamRecordDetail } from '../../api/exam'
import UserMenu from '../../components/tools/UserMenu'
import { mapGetters } from 'vuex'

export default {
  name: 'ExamRecordDetail',
  components: {
    UserMenu,
  },
  data() {
    return {
      // 考试详情对象
      examDetail: {},
      // 考试记录详情对象
      recordDetail: {},
      // 用户做过的问题都放到这个数组中，键为问题id, 值为currentQuestion(其属性answers属性用于存放答案选项地id或ids),，用于存放用户勾选的答案
      answersMap: {},
      // 题目的正确答案
      answersRightMap: {},
      // 题目的作答结果(正确或错误)
      resultsMap: new Map(),
      // 当前用户的问题
      currentQuestion: '',
      // 单选或判断题的绑定值，用于从answersMap中初始化做题状态
      radioValue: '',
      // 单选题的正确答案，用于从answersRightMap中初始化做题状态
      radioRightValue: '',
      // 多选题的绑定值，用于从answersMap中初始化做题状态
      checkValues: [],
      // 多选题的绑定值，用于从answersRightMap中初始化做题状态
      checkRightValues: [],
      optionStyle: {
        display: 'block',
        height: '30px',
        lineHeight: '30px',
        marginLeft: '0px',
      },
      questionsIds: [],
      currIndex: 0,
      quesIdsPanelVisible: false,
    }
  },
  computed: {
    /**
     * 当前题目用户是否作答正确
     * */
    questionRight() {
      return this.resultsMap !== '' && this.resultsMap.get(this.currentQuestion.id) === 'True'
    },
    questionIdsPanel() {
      if (!this.quesIdsPanelVisible) {
        return 'hidden'
      }
      return ''
    },
    answerRightOptions() {
      const optIds = this.answersRightMap.get(this.currentQuestion.id)
      const optMap = {}
      for (const opt of this.currentQuestion.options) {
        optMap[opt.questionOptionId] = opt
      }
      const res = []
      if (optIds) {
        for (const optId of optIds) {
          const opt = optMap[optId]

          if (this.currentQuestion.type === '判断题') {
            res.push(opt.questionOptionContent)
          } else {
            res.push(opt.questionOptionKey)
          }
        }
      }
      return res
    },
  },
  mounted() {
    this.answersMap = new Map()
    this.answersRightMap = new Map()
    this.resultsMap = new Map()
    const that = this
    // 从后端获取考试的详细信息，渲染到考试详情里,需要加个延时，要不拿不到参数
    getExamDetail(this.$route.params.exam_id).then((res) => {
      if (res.code === 0) {
        // 赋值考试对象
        that.examDetail = res.data
        const { radioIds = [], checkIds = [], judgeIds = [] } = this.examDetail
        this.questionsIds = [...radioIds, ...checkIds, ...judgeIds]
        if (this.questionsIds.length > 0) {
          this.getQuestionDetail(this.currIndex)
        }
        return res.data
      } else {
        this.$notification.error({
          message: '获取考试详情失败',
          description: res.msg,
        })
      }
    })
    // 查看考试记录详情，渲染到前端界面
    getExamRecordDetail(this.$route.params.record_id).then((res) => {
      if (res.code === 0) {
        console.log(res.data)
        // 赋值考试对象
        that.recordDetail = res.data
        // 赋值用户的作答答案
        that.objToMap()
        return res.data
      } else {
        this.$notification.error({
          message: '获取考试记录详情失败',
          description: res.msg,
        })
      }
    })
  },
  methods: {
    // 从全局变量中获取用户昵称和头像,
    ...mapGetters(['nickname', 'avatar']),
    /**
     * 把后端传过来的对象Object转换成Map
     **/
    objToMap() {
      for (const item in this.recordDetail.answersMap) {
        this.answersMap.set(item, this.recordDetail.answersMap[item])
      }

      for (const item in this.recordDetail.answersRightMap) {
        this.answersRightMap.set(item, this.recordDetail.answersRightMap[item])
      }

      for (const item in this.recordDetail.resultsMap) {
        this.resultsMap.set(item, this.recordDetail.resultsMap[item])
      }
    },
    getQuestionDetail(index) {
      this.currIndex = index
      const questionId = this.questionsIds[this.currIndex]
      // 问题切换时从后端拿到问题详情，渲染到前端content中
      const that = this
      // 清空问题绑定的值
      this.radioValue = ''
      this.radioRightValue = ''
      this.checkValues = []
      this.checkRightValues = []
      getQuestionDetail(questionId).then((res) => {
        if (res.code === 0) {
          // 赋值当前考试对象
          that.currentQuestion = res.data
          // 查看用户是不是已经做过这道题又切换回来的，answersMap中查找，能找到这个题目id对应的值数组不为空说明用户做过这道题
          if (that.answersMap.get(that.currentQuestion.id)) {
            // 说明之前做过这道题了
            if (that.currentQuestion.type === '单选题' || that.currentQuestion.type === '判断题') {
              that.radioValue = that.answersMap.get(that.currentQuestion.id)[0]
              that.radioRightValue = that.answersRightMap.get(that.currentQuestion.id)[0]
            } else if (that.currentQuestion.type === '多选题') {
              // 数组是引用类型，因此需要进行拷贝，千万不要直接赋值
              Object.assign(that.checkValues, that.answersMap.get(that.currentQuestion.id))
              Object.assign(that.checkRightValues, that.answersRightMap.get(that.currentQuestion.id))
            }
          }
          return res.data
        } else {
          this.$notification.error({
            message: '获取问题详情失败',
            description: res.msg,
          })
        }
      })
    },
    questionIdsPanelClass(index) {
      let res = ''
      if (this.answersMap.get(this.questionsIds[index])) {
        res += 'done '
      } else {
        res += 'undo '
      }
      if (index === this.currIndex) {
        res += 'current'
      }
      return res
    },
    pre() {
      if (this.currIndex > 0) {
        this.currIndex--
      } else {
        this.$message.info('已是第一题')
        return
      }
      this.getQuestionDetail(this.currIndex)
    },
    next() {
      if (this.currIndex < this.questionsIds.length - 1) {
        this.currIndex++
      } else {
        this.$message.info('已是最后一题')
        return
      }
      this.getQuestionDetail(this.currIndex)
    },
    toggleQuesIdsPanel() {
      this.quesIdsPanelVisible = !this.quesIdsPanelVisible
    },
  },
}
</script>

<style scoped lang="less">
@media screen and (min-width: 800px) {
  .question-ids-panel {
    height: 84vh;
    width: 255px;
    margin-right: 10px;
  }

  .main-ctn {
    margin: 10px 10px 0px;
    height: 84vh;
  }

  .question-panel {
    height: 84vh;
  }

  .qustion-ids-panel-togg-btn {
    display: none;
  }
}

@media screen and (max-width: 800px) {
  .question-ids-panel {
    position: absolute;
    bottom: 0;
    z-index: 99;
    border: 1px solid;
    width: 90%;
    border-radius: 10px;
    max-height: 60vh;

  }
  .question-ids-panel.hidden {
    display: none;
  }

  .question-panel {
    flex-grow: 1;

    /deep/ .ant-checkbox-wrapper {
      height: auto !important;
    }

    /deep/ .ant-radio-wrapper {
      height: auto !important;
      white-space: normal;
    }
  }

  .main-ctn {
    // height: 100vh;
    position: relative;
  }

  /deep/ .ant-layout-header {
    padding: 0 10px;
  }
}

.main-ctn {
  overflow: initial;
}

.question-panel {
  padding: 24px;
  background: rgb(255, 255, 255);
  flex-grow: 1;
}

.question-ids-panel {
  display: flex;
  flex-wrap: wrap;
  align-content: flex-start;
  justify-content: space-around;
  background: #fff;
  padding: 10px;
  overflow-y: scroll;
}

.question-ids-ele {
  width: 40px;
  text-align: center;
  vertical-align: center;
  line-height: 40px;
  height: 40px;
  margin-bottom: 10px;
  margin-right: 10px;
}

.question-ids-ele:hover {
  cursor: pointer;
}

.question-ids-ele::after {
  content: '';
  width: 40px;
  border: 1px solid transparent;
}
.question-ids-ele.undo {
  border: 1px solid;
}

.question-ids-ele.done {
  border: 1px solid;
  background-color: lightblue;
}
.question-ids-ele.current {
  box-shadow: 1px 1px 5px #999;
}
</style>
