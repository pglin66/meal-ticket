<template>
  <div class="page-apply">
    <div class="page-wrap" :style="{'min-height': $root.clientHeight}">
      <v-header :title="teamName">
        <a :href="locals.api + '/team/export?teamId=' + teamId" slot="right" class="export">Export</a>
      </v-header>

      <div class="search" v-show="showInput">
        <input type="search"
          v-model="value"
          :placeholder="placeholder"
          @keyup.enter="confirmSubmit"
          @keydown.tab.prevent="useLastApplyUserName"
        >
        <span class="submit" :class="{'active': value.trim().length}" @click="confirmSubmit">提交</span>
      </div>

      <div class="total"><span>{{today}} 已报名{{member.length}}人</span></span></div>

      <div class="list" v-if="member && member.length">
        <v-panel v-for="(index, item) in member"
          v-link="'/' + teamId + '/' + item.name"
          :title="item.name"
          :summary="item.intro"
          >
          <span slot="left">
            <img :src="'/static/img/avatar/' + item.avatar + '.jpg'">
          </span>
        </v-panel>
      </div>
    </div>

    <div class="page-sticky-footer">
      数据将会每日统计上报，请使用真实姓名
    </div>

    <toast :show.sync="already" :type="'text'">该成员已存在</toast>

    <confirm :show.sync="confirm" title="确认删除？" @on-confirm="remove(true)" @on-cancel="remove(false)"></confirm>
    <confirm :show.sync="showConfirmSubmitModal" title="每天只有一次提交，确认提交？" @on-confirm="submit(true)" @on-cancel="submit(false)"></confirm>
  </div>
</template>

<script>
  import moment from 'moment'
  import store from 'store'
  import _ from 'lodash'

  import toast from 'vux/src/components/toast'
  import confirm from 'vux/src/components/confirm'

  import vHeader from 'components/v-header.vue'
  import vPanel from 'components/v-panel.vue'

  export default {
    components: {
      toast,
      confirm,
      vHeader,
      vPanel
    },
    data () {
      return {
        member: [],
        value: '',
        already: false,
        confirm: false,
        today: moment().format('YYYY-MM-DD').toString(),
        timer: -1,
        showInput: true,
        showConfirmSubmitModal: false,
        teamId: '',
        teamName: '',
        lastApplyUserName: ''
      }
    },
    computed: {
      placeholder: function () {
        var vm = this
        if (vm.lastApplyUserName) {
          return vm.lastApplyUserName + '  [ tab ]'
        }
        return '请输入你的姓名，一天只能提交一次'
      }
    },
    methods: {
      confirmSubmit: function () {
        var vm = this
        var value = vm.value.trim()
        if (!value) {
          return
        }
        vm.showConfirmSubmitModal = true
      },
      submit: function (ok) {
        var vm = this
        var value = vm.value.trim()
        if (!ok) {
          return
        }
        if (!value) {
          return
        }
        if (vm.member.indexOf(value) > -1) {
          vm.already = true
          return
        }

        vm.$http({
          method: 'post',
          url: locals.api + '/team/apply',
          body: {
            teamId: vm.teamId,
            userName: value
          }
        }).then(function ({body}) {
          if (body.err) {
            return
          }

          var date = moment().format('YYYY-MM-DD').toString()
          store.set('lastApplyDate', date)
          vm.showInput = false

          store.set('lastApplyUserName', value)
          store.set('whoami', {
            date: date,
            teamId: vm.teamId,
            userName: value
          })

          // keydown后同步修改vm.value，不会生效
          // 可能是插件监听keyup事件后重新赋值
          // 改成keyup解决了
          vm.value = ''
          vm.update()
        })
      },
      remove: function (ok) {
        var vm = this
        if (ok) {
          vm.member.$remove(vm.confirmTarget)
          vm.$http({
            method: 'delete',
            url: locals.api + '/team/apply',
            body: {
              teamId: vm.teamId,
              userName: vm.confirmTarget
            }
          }).then(function () {
            vm.update()
          })
        }
        vm.confirmTarget = null
      },
      confirmRemove: function (item) {
        var vm = this
        vm.confirmTarget = item
        vm.confirm = true
      },
      update: function () {
        var vm = this
        return vm.$http({
          method: 'get',
          url: locals.api + '/team/apply',
          params: {
            teamId: vm.teamId
          }
        }).then(function ({body}) {
          if (!_.isEqual(vm.member, body.data)) {
            vm.member = body.data
          }
        })
      },
      useLastApplyUserName: function () {
        var vm = this
        if (vm.lastApplyUserName) {
          vm.value = vm.lastApplyUserName
        }
      }
    },
    ready: function () {
      var vm = this
      var poll = async function () {
        await vm.update()
        vm.timer = setTimeout(poll, 2000)
      }
      vm.timer = setTimeout(poll, 1000)
    },
    destroyed: function () {
      var vm = this
      clearTimeout(vm.timer)
    },
    route: {
      activate: function (transition) {
        var vm = this

        vm.teamId = vm.$route.params.team
        vm.$root.loadingVisible = true

        var lastApplyDate = moment().format('YYYY-MM-DD').toString()
        if (store.get('lastApplyDate') === lastApplyDate) {
          vm.showInput = false
        }

        if (store.get('lastApplyUserName')) {
          vm.lastApplyUserName = store.get('lastApplyUserName')
        }

        vm.$http({
          method: 'get',
          url: locals.api + '/team/name',
          params: {
            teamId: vm.teamId
          }
        }).then(function ({body}) {
          vm.teamName = body.data.teamName
          // vm.$root.$refs.header.title = vm.teamName
        }).then(function () {
          return vm.update()
        }).then(function () {
          vm.$root.loadingVisible = false
          // 不知道什么原因，loading的动画，会被下一页的入场动画打断，导致loading的遮罩层永远不隐藏
          // 猜测可能是入场动画的transform导致loading的opacity的事件被打断
          transition.next()
        })
      }
    }
  }
</script>

<style lang='less' scoped>
  @page-sticky-footer-height: 50px;
  .page-wrap{
    margin-bottom: -@page-sticky-footer-height;
    &:after{
      content: "";
      display: block;
      height: @page-sticky-footer-height;
    }
  }
  .page-sticky-footer{
    font-size: 12px;
    line-height: @page-sticky-footer-height;
    text-align: center;
    color: #999;
  }
  .export{
    line-height: 40px;
    color: #333;
  }
  .total{
    margin: 12px 0;
    font-size: 12px;
    text-align: center;
  }
  .list{
    margin: 0 12px;
    /*margin-bottom: 20px;*/
    border-radius: 8px;
    box-shadow: 0 0 5px 1px rgba(0, 0, 0, .1);
    img{
      display: inline-block;
      width: 40px;
      height: 40px;
      margin-right: 10px;
      border-radius: 100px;
      object-fit: cover;
    }
  }
  .page-apply{
    .weui_cell{
      cursor: pointer;
    }
  }
  .search{
    position: relative;
    margin: 12px 12px;
    input{
      width: 100%;
      height: 36px;
      padding: 0 72px 0 12px;
      border: none;
      border-radius: 8px;
      font-size: 14px;
      background: #e5e5e5;
      box-shadow: 0 0 10px 0 rgba(0, 0, 0, .02) inset;
      outline: none;
      -webkit-appearance: none;
    }
    .submit{
      position: absolute;
      right: 0;
      top: 0;
      width: 60px;
      height: 100%;
      background: #b1b1b1;
      text-align: center;
      line-height: 36px;
      border-radius: 0 8px 8px 0;
      color: #fff;
      cursor: pointer;
      &.active{
        background: #1bc769;
      }
    }
  }
</style>

<style lang="less">
  .page-apply{
    .list{
      .v-panel{
        .center{
          min-height: 40px;
        }
      }
    }
  }
</style>
