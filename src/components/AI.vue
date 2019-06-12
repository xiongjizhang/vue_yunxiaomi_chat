<template>
	<transition name="slideToRight">
		<div class="container">
			<header class="chat-header">
				<i class="iconfont icon-xiaoxi" @click="$router.push({path: '/Chat'})"></i>
				<h3>智能客服</h3>
				<i class="iconfont icon-logout" @click="$router.push('/')"></i>
			</header>
			<div class="chat-content" ref="chatContent">
				<ul class="message-list">
					<li class="clearfix"
					    v-for="msg of messages"
					    :class="{'others': msg.from === 'ai', 'mine': msg.from !== 'ai'}"
          >
  					<p class="date">{{ msg.date }}</p>
  					<p class="info">
  						<span class="portrait">
  							<img :src="msg.portrait">
  						</span>
  						<span class="nickname">{{ msg.nickname }}</span>
  					</p>
  					<p class="content" v-html="msg.content"></p>
  				</li>
  			</ul>
  		</div>
  		<footer class="chat-footer">
  			<i></i>
  			<input v-model="inputText" @keyup.enter="sendMsg" autofocus>
        <i class="sendBtn btn iconfont icon-icon_send_fill"
           :class="{'clickable': clickable}"
           @click="sendMsg"
        ></i>
      </footer>
    </div>
  </transition>
</template>



<script>
  import CryptoJS from 'crypto-js'

  function addPreZero(num){
    if(num<10){
      return '0'+num
    }else{
      return num
    }
  }

  export default {
  name: 'Chat',
  data() {
    return {
      messages: [],
      inputText: '',
      nickname: '',
      portrait: '',
      location: ''
    }
  },

  beforeRouterEnter(to, from, next) {
    if (!localStorage.nickname) {
      next('/')
    } else {
      next()
    }
  },

  created() {
    this.nickname = this.$store.state.nickname
    this.portrait = this.$store.state.portrait
    this.location = this.$store.state.location

    if (localStorage.record_ai) {
      this.messages = JSON.parse(localStorage.record_ai)
      return
    }

    setTimeout(() => {
      this.messages.push({
        from: 'ai',
        date: this.getTime(),
        nickname: '智能助手',
        portrait: 'static/portrait/robot.svg',
        content: '有什么可以帮您的吗？'
      })
    }, 1000)
  },

  activated() {
    this.$nextTick(() => {
      this.$refs.chatContent.scrollTop = this.$refs.chatContent.scrollHeight
    })
  },

  methods: {

    async sendMsg() {
      if (!this.inputText) {
        return
      }

      let chatbot = "chatbot-cn-0pp14yt01000av"  // chatbot-cn-v0h15cemp0006p  chatbot-cn-0pp14yt01000av

      let date = new Date()
      // 当前时间转为utc时间
      let utc_time = date.getUTCFullYear() + '-' + addPreZero(date.getUTCMonth()+1) + '-' + addPreZero(date.getUTCDate()) + 'T' +
        addPreZero(date.getUTCHours()) + ':' + addPreZero(date.getUTCMinutes()) + ':' + addPreZero(date.getUTCSeconds()) + 'Z'

      let sign_content = 'GET&%2F&AccessKeyId%3DLTAIUzHm83TGI0Gz%26Action%3DChat%26Format%3Djson' +
        '%26InstanceId%3D' + chatbot + '%26SignatureMethod%3DHMAC-SHA1' +
        '%26SignatureNonce%3D' + date.valueOf() +
        '%26SignatureVersion%3D1.0' +
        '%26Timestamp%3D' + encodeURIComponent(encodeURIComponent(utc_time)) + // .replace(':','%253A').replace(':','%253A') +
        '%26Utterance%3D' + encodeURIComponent(encodeURIComponent(this.inputText)) +
        '%26Version%3D2017-10-11'

      // 签名计算
      let signature = Buffer.from(CryptoJS.HmacSHA1(sign_content, 's1HdCHuGJGORPbZ09rNbc8nMOdIQxQ&').toString(), 'hex').toString('base64')

      let url = 'https://chatbot.cn-shanghai.aliyuncs.com/?Format=json&Version=2017-10-11' +
        '&AccessKeyId=LTAIUzHm83TGI0Gz' +
        '&Signature=' + encodeURIComponent(signature) +
        '&SignatureMethod=HMAC-SHA1' +
        '&Timestamp=' + encodeURIComponent(utc_time) +
        '&SignatureVersion=1.0' +
        '&SignatureNonce=' + date.valueOf() +
        '&Action=Chat&InstanceId=' + chatbot +
        '&Utterance=' + encodeURIComponent(this.inputText)

      // console.log(url)
      // console.log(signature)
      var result_str = new Array()
      this.pushMine()
      var message_type
      // this.pushMine()
      /*this.axios.get(url).then(function (response){
        console.log(response.data.Messages[0]) // JSON.parse
        message_type = response.data.Messages[0].Type
        if (message_type === 'Recommend') {
          console.log(response.data.Messages[0].Recommends)
          let recommends = response.data.Messages[0].Recommends
          recommends.forEach(recommend => {
            console.log(recommend.Title)
            result_str.push(recommend.Title)
          })
        }
      }).catch(function(error){
        console.log(error)
      })*/
      /*this.axios.get(url).then(res => {
        console.log(res)
        this.pushAINew('您是想问下面哪个问题: ')
        message_type = res.data.Messages[0].Type
      })*/

      let yunXiaoMiData = await this.axios.get(url)

      console.log(yunXiaoMiData.data)

      var yunXiaoMiStr = ''
      message_type = yunXiaoMiData.data.Messages[0].Type

      if (message_type === 'Recommend') {
        result_str.push('猜您是不是想问: ')
        yunXiaoMiData.data.Messages[0].Recommends.forEach(recommend => {
          result_str.push('<b>' + recommend.Title + '</b>')
        })
      } else if (message_type === 'Knowledge') {
        result_str.push(yunXiaoMiData.data.Messages[0].Knowledge.Content)
      } else {
        result_str.push(yunXiaoMiData.data.Messages[0].Text.Content)
      }
      this.pushAINew(result_str.join('<br>'))
      // 智能机器人应答的接口
      /*let url = 'http://www.niceming.cn/api/chat/AI'
      let data = {
        city: this.location,
        userId: this.nickname,
        inputText: this.inputText
      }

      this.pushMine()
      this.axios.get(url, { params: data }).then(res => {
        this.pushAI(res.data.results)
      })*/
      this.inputText = ''
    },

    pushMine() {
      this.messages.push({
        from: 'mine',
        date: this.getTime(),
        nickname: this.nickname,
        portrait: this.portrait,
        content: this.inputText
      })
    },

    async getYunXiaoMiData (url) {
      try {
        // let res = await this.axios.get(url)
        this.yunXiaoMiData = await this.axios.get(url).data
      } catch (err) {
        console.log(err)
        alert('请求出错！')
      }
    },

    pushAINew(result) {
      let message = {
        from: 'ai',
        date: this.getTime(),
        nickname: '智能助手',
        portrait: 'static/portrait/robot.svg'
      }
      message.content = result
      this.messages.push(message)
    },

    pushAI(results) {
      let message = {
        from: 'ai',
        date: this.getTime(),
        nickname: '智能助手',
        portrait: 'static/portrait/robot.svg'
      }

      if (!results.length) {
        message.content = '这个问题可难倒我了'
        this.messages.push(message)
        return
      }

      results.forEach(item => {
        console.log(item.resultType)
        if (item.resultType === 'text') {
          message.content = item.values.text
        } else if (item.resultType === 'image') {
          message.content = `<img width="250" src="${item.values.image}">`
        }
        message.content = item
        this.messages.push(message)
      })
    },

    getTime() {
      return this.moment().format('YYYY-MM-DD HH:mm:ss')
    },

    fixedBottom() {
      clearTimeout(this.timer)
      this.timer = setTimeout(() => {
        this.$refs.chatContent.scrollTop = this.$refs.chatContent.scrollHeight
      }, 20)
    }
  },

  computed: {
    clickable() {
      return this.inputText.length > 0
    }
  },

  watch: {
    messages: {
      handler() {
        localStorage.record_ai = JSON.stringify(this.messages)
        this.fixedBottom()
      },
      deep: true
    }
  }
}
</script>

<style lang="scss" scoped>
@import '../style/public.scss';
@import '../style/_Chat.scss';
</style>
