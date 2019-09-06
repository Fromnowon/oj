<template>
  <div class="container">
    <div class="avatar-container">
      <img class="avatar" :src="profile.avatar" />
    </div>
    <Card :padding="100">
      <div v-if="profile.user">
        <p style="margin-top: -10px">
          <span v-if="profile.user" class="emphasis">{{profile.user.username}}</span>
          <span v-if="profile.school">@{{profile.school}}</span>
        </p>
        <p v-if="profile.mood">{{profile.mood}}</p>
        <hr id="split" />

        <div class="flex-container">
          <div class="left">
            <p>{{$t('m.UserHomeSolved')}}</p>
            <p class="emphasis">{{profile.accepted_number}}</p>
          </div>
          <div class="middle">
            <p>{{$t('m.UserHomeserSubmissions')}}</p>
            <p class="emphasis">{{profile.submission_number}}</p>
          </div>
          <div class="right">
            <p>{{$t('m.UserHomeScore')}}</p>
            <p class="emphasis">{{profile.total_score}}</p>
          </div>
        </div>
        <div id="problems">
          <div v-if="problems.length">
            {{$t('m.List_Solved_Problems')}}
            <Poptip v-if="refreshVisible" trigger="hover" placement="right-start">
              <Icon type="ios-help-outline"></Icon>
              <div slot="content">
                <p>
                  If you find the following problem id does not exist,
                  <br />try to click the button.
                </p>
                <Button type="info" @click="freshProblemDisplayID">regenerate</Button>
              </div>
            </Poptip>
          </div>
          <p v-else>{{$t('m.UserHomeIntro')}}</p>
          <div class="btns">
            <div class="problem-btn" v-for="problemID of problems" :key="problemID">
              <Button type="ghost" @click="goProblem(problemID)">{{problemID}}</Button>
            </div>
          </div>
        </div>
        <!-- 最近提交记录 -->
        <div class="recent_submission">
          <h3>最近提交记录</h3>
          <Table :columns="recent_columns" :data="recent_data"></Table>
        </div>
        <div id="icons">
          <a :href="profile.github">
            <Icon type="social-github-outline" size="30"></Icon>
          </a>
          <a :href="'mailto:'+ profile.user.email">
            <Icon class="icon" type="ios-email-outline" size="30"></Icon>
          </a>
          <a :href="profile.blog">
            <Icon class="icon" type="ios-world-outline" size="30"></Icon>
          </a>
        </div>
      </div>
    </Card>
  </div>
</template>
<script>
import { mapActions } from "vuex";
import time from "@/utils/time";
import api from "@oj/api";
import { JUDGE_STATUS, USER_TYPE } from "@/utils/constants";

export default {
  data() {
    return {
      username: "",
      profile: {},
      problems: [],
      recent_columns: [
        {
          title: "问题ID",
          key: "problem",
          align: "center",
          render: (h, params) => {
            return h(
              "a",
              {
                attrs: {
                  href: "problem/" + params.row.problem
                }
              },
              params.row.problem
            );
          }
        },
        {
          title: "提交时间",
          key: "create_time",
          align: "center"
        },
        {
          title: "结果",
          key: "result",
          align: "center",
          render: (h, params) => {
            return h(
              "Tag",
              {
                props: {
                  color: JUDGE_STATUS[params.row.result].color
                }
              },
              JUDGE_STATUS[params.row.result].name
            );
          }
        }
      ],
      recent_data: []
    };
  },
  mounted() {
    this.init();
  },
  methods: {
    ...mapActions(["changeDomTitle"]),
    init() {
      this.username = this.$route.query.username;
      api.getUserInfo(this.username).then(res => {
        this.changeDomTitle({ title: res.data.data.user.username });
        this.profile = res.data.data;
        this.getSolvedProblems();
        let registerTime = time.utcToLocal(
          this.profile.user.create_time,
          "YYYY-MM-D"
        );
        console.log("The guy registered at " + registerTime + ".");
      });

      //处理最近提交记录，直接指向当前用户名
      const limit = 10;
      api
        .getSubmissionList(0, limit, {
          limit: limit,
          username: this.$route.query.username
        })
        .then(res => {
          let results = res.data.data.results;
          for (
            let index = 0, length = results.length;
            index < length;
            index++
          ) {
            results[index].create_time = time.utcToLocal(
              results[index].create_time,
              "YYYY-MM-D HH:MM:ss"
            );
          }
          this.recent_data = results;
        });
    },
    getSolvedProblems() {
      let ACMProblems = this.profile.acm_problems_status.problems || {};
      let OIProblems = this.profile.oi_problems_status.problems || {};
      // todo oi problems
      let ACProblems = [];
      for (let problems of [ACMProblems, OIProblems]) {
        Object.keys(problems).forEach(problemID => {
          if (problems[problemID]["status"] === 0) {
            ACProblems.push(problems[problemID]["_id"]);
          }
        });
      }
      ACProblems.sort();
      this.problems = ACProblems;
    },
    goProblem(problemID) {
      this.$router.push({
        name: "problem-details",
        params: { problemID: problemID }
      });
    },
    freshProblemDisplayID() {
      api.freshDisplayID().then(res => {
        this.$success("Update successfully");
        this.init();
      });
    }
  },
  computed: {
    refreshVisible() {
      if (!this.username) return true;
      if (this.username && this.username === this.$store.getters.user.username)
        return true;
      return false;
    }
  },
  watch: {
    $route(newVal, oldVal) {
      if (newVal !== oldVal) {
        this.init();
      }
    }
  }
};
</script>

<style lang="less" scoped>
.container {
  position: relative;
  width: 75%;
  margin: 170px auto;
  text-align: center;
  p {
    margin-top: 8px;
    margin-bottom: 8px;
  }
  .avatar-container {
    position: absolute;
    left: 50%;
    transform: translate(-50%);
    z-index: 1;
    top: -90px;
    .avatar {
      width: 140px;
      height: 140px;
      border-radius: 50%;
      box-shadow: 0 1px 1px 0;
    }
  }
  .emphasis {
    font-size: 20px;
    font-weight: 600;
  }
  #split {
    margin: 20px auto;
    width: 90%;
  }
  .flex-container {
    margin-top: 30px;
    padding: auto 20px;
    .left {
      flex: 1 1;
    }
    .middle {
      flex: 1 1;
      border-left: 1px solid #999;
      border-right: 1px solid #999;
    }
    .right {
      flex: 1 1;
    }
  }
  #problems {
    margin-top: 40px;
    padding-left: 30px;
    padding-right: 30px;
    font-size: 18px;
    .btns {
      margin-top: 15px;
      .problem-btn {
        display: inline-block;
        margin: 5px;
      }
    }
  }
  #icons {
    position: absolute;
    bottom: 20px;
    left: 50%;
    transform: translate(-50%);
    .icon {
      padding-left: 20px;
    }
  }
  .recent_submission {
    margin-top: 50px;
    h3 {
      margin-bottom: 20px;
    }
  }
}
</style>
