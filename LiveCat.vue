<template>
  <div
    style="width: 100%; height: 100%"
    v-loading="loading"
    :element-loading-text="loadingText"
    :element-loading-spinner="loadingIcon"
    element-loading-background="black"
    v-if="renderVisible"
  >
  <div class="test"></div>
    <div class="player-container" ref="player"></div>
    <div class="btn-container">
      <SettingDrawer
        ref="SettingDrawer"
        :fullEnables="fullEnable"
        :from="from"
      />
      <InputDialog :visible="dialogVisible" @submit="submitText" />
    </div>
    <div
      v-show="!isPC"
      id="joyLeft"
      style=""
      :class="[isShow ? 'hidden' : '']"
    ></div>
    <div
      v-show="!isPC"
      id="joyRight"
      style=""
      :class="[isShow ? 'hidden' : '']"
    ></div>
    <LoadPhone v-show="!isPC" :isShow="isShow" chooseCss="container1" />
  </div>
</template>

<script>
import { Launcher } from 'live-cat'
import { mapState } from 'vuex'
import nipplejs from 'nipplejs'

import { MessageType } from '@/utils/message-type'
import { client, AgoraRTC } from '@/utils/agora'

import { isPC, isIOS } from '../../utils/index'
import SettingDrawer from '../../components/SettingDrawer'
import InputDialog from '../../components/InputDialog'
// 引入手机横屏
import LoadPhone from '../LoadPhone'

import 'vant/lib/dialog/style'
export default {
  name: 'LiveCat',
  data() {
    return {
      launch: null,
      connect: null,
      address: 'https://app.3dcat.live',
      appKey: 'dvi0vho8mqrUxMX5',
      micStatus: false,
      fullEnable: false,
      dialogVisible: false,
      volumesStr: '',
      from: { path: '/', query: {} },
      renderVisible: false,
      loading: true,
      loadingText: '正在启动云渲染...',
      loadingIcon: 'el-icon-loading',
      isShow: true
    }
  },
  components: { SettingDrawer, InputDialog, LoadPhone },
  props: {
    start: { type: Boolean }
  },

  computed: mapState({
    userInfo: (state) => state.user.info,
    apitoken: (state) => state.user.token,
    group: (state) => state.user.group,
    avatar: (state) => state.avatar.avatar,
    roomInfo: (state) => state.room.roomInfo
  }),
  watch: {
    start(val) {
      if (val) {
        this.renderVisible = true
        this.$nextTick(() => {
          this.liveStart()
        })
      } else {
        this.renderVisible = false
        this.connect && this.liveStop()
      }
    }
  },
  destroyed() {
    this.liveStop()
  },
  methods: {
    // 默认全屏弹窗
    open() {
      this.$message({
        type: 'success',
        message: '为了您更好的体验,即将进入全屏',
        onClose: () => {
          this.$refs['SettingDrawer'].fulled()
        }
      })
    },
    async send() {
      const data = {
        messageId: MessageType.SEND_JOIN_INFO,
        roomInfo: this.roomInfo,
        avatar: this.avatar,
        userInfo: this.userInfo,
        apitoken: this.apitoken,
        username: this.userInfo.phone,
        group: this.group,
        isPC: this.isPC,
        hideRightStick: true,
        debug: false
      }
      console.log(JSON.stringify(data))
      return await this.connect.emitUIInteraction(JSON.stringify(data))
    },
    async joinAgora(appid, channel, token, uid) {
      // console.log(this.roomInfo.vr_rooms_gmeappid)
      await client.leave()
      client.enableAudioVolumeIndicator()

      client.on('volume-indicator', async (volumes) => {
        const data = {
          messageId: MessageType.SEND_VOLUMES,
          volumes
        }
        // console.log('volume-indicator', JSON.stringify(data))
        this.volumesStr = JSON.stringify(data)
      })
      await client.join(appid, channel, token, uid)
      this.localAudioTrack = await AgoraRTC.createMicrophoneAudioTrack()

      await client.publish([this.localAudioTrack])
      // console.log('this.micStatus', this.micStatus)
      await this.localAudioTrack.setEnabled(this.micStatus)
    },
    async micOn() {
      await this.localAudioTrack.setEnabled(true)
      this.micStatus = true
    },
    async micOff() {
      await this.localAudioTrack.setEnabled(false)
      this.micStatus = false
    },
    async leaveAgora() {
      this.localAudioTrack.close()
      await client.leave()
    },
    async setAgoraRemoteVolumes(volumes) {
      client.remoteUsers.forEach((remoteUser) => {
        const volume = volumes.find(
          (v) => String(v.uid) === String(remoteUser.uid)
        )
        if (volume) {
          remoteUser.audioTrack.setVolume(volume.vol)
        }
      })
      if (this.volumesStr) {
        console.log(this.volumesStr)
        await this.connect.emitUIInteraction(this.volumesStr)
      }
    },
    async onAppReady() {
      await this.connect.emitUIInteraction(
        JSON.stringify({ messageId: MessageType.GOT_APP_READY })
      )
      await this.send()
      await this.joinAgora(
        this.roomInfo.vr_rooms_gmeappid,
        this.roomInfo.vr_rooms_gmeroomid,
        null,
        this.userInfo.vr_user_id
      )
    },

    async messageHandle(message) {
      switch (message.messageId) {
        case MessageType.MIC_ON:
          await this.micOn()
          break
        case MessageType.MIC_OFF:
          await this.micOff()
          break
        case MessageType.SET_REMOTE_VOLUME:
          await this.setAgoraRemoteVolumes(message.playersData)
          break
        case MessageType.APP_READY:
          await this.onAppReady()
          break
        case MessageType.CHANGE_AGORA_CHANEL:
          // console.log(message)
          await this.joinAgora(
            this.roomInfo.vr_rooms_gmeappid,
            message.channelName,
            null,
            this.userInfo.vr_user_id
          )
          break
        case MessageType.START_TYPING:
          this.dialogVisible = true
          break
        case MessageType.OPEN_NEW_URL:
          window.open(message.url, '_blank')
          break
        case MessageType.KICK_USER:
          if (message.jsonStr) {
            this.$refs['SettingDrawer'].exit(message.jsonStr)
            return
          }
          this.loadingText = '您的账号/邀请码已经在其他设备上登录!'
          this.loadingIcon = 'el-icon-warning'
          this.loading = true
          setTimeout(() => {
            this.$refs['SettingDrawer'].exit(message.jsonStr)
          }, 5000)
          break
        default:
          break
      }
    },
    async submitText(text) {
      console.log(text)
      await this.connect.emitUIInteraction(
        JSON.stringify({
          messageId: MessageType.END_TYPING,
          content: text
        })
      )
      this.dialogVisible = false
    },
    async liveStart() {
      this.isPC = isPC()
      this.isIOS = isIOS()
      const query = this.$route.query
      this.from = JSON.parse(query.from)
      this.initJoyStick()
      this.launch = new Launcher({
        baseOptions: {
          address: this.address,
          appKey: this.appKey,
          startType: 1
        },
        extendOptions: {
          onPlay: async () => {
            // console.log(this)
            this.connect = await this.launch.getConnection()
            this.connect.event.interaction.on((res) => {
              const str = new TextDecoder('utf-8').decode(new Uint8Array(res))
              console.log('接收到数据:', str)
              this.messageHandle(JSON.parse(str))
            })
            this.loading = false
            this.fitIOS()
          }
        }
      })
      await this.launch.automata(this.$refs.player).catch((res) => {})

      if (!this.isIOS) this.open()
      window.onbeforeunload = (e) => {
        e.preventDefault()
        this.channel.sendMessage(
          JSON.stringify({ messageId: MessageType.EXIT })
        )
        this.liveStop()
        return undefined
      }
    },
    async liveStop() {
      this.leaveAgora()
      this.connect?.emitUIInteraction(
        JSON.stringify({ messageId: MessageType.EXIT })
      )
      this.launch.destory()
    },
    // 添加虚拟摇杆
    async initJoyStick() {
      if (this.isPC) return
      const stickDom = document.getElementById('joyLeft')
      const joyStick = nipplejs.create({
        zone: stickDom,
        mode: 'static',
        position: { left: '50%', top: '50%' }
      })

      joyStick.on('move', (evt, data) => {
        if (this.leftSendTime && this.leftSendTime + 150 > Date.now()) {
          return
        }
        this.connect.emitUIInteraction(
          JSON.stringify({
            messageId: MessageType.MOVE_LEFT_STICK,
            x: data.vector.x,
            y: data.vector.y
          })
        )
        this.leftSendTime = Date.now()
      })
      joyStick.on('end', (evt, data) => {
        this.connect.emitUIInteraction(
          JSON.stringify({
            messageId: MessageType.MOVE_LEFT_STICK,
            x: 0,
            y: 0
          })
        )
      })
      const stickRightDom = document.getElementById('joyRight')
      const joyStickRight = nipplejs.create({
        zone: stickRightDom,
        mode: 'static',
        position: { left: '50%', top: '50%' }
      })
      joyStickRight.on('move', (evt, data) => {
        if (this.rightSendTime && this.rightSendTime + 100 > Date.now()) {
          return
        }
        this.connect.emitUIInteraction(
          JSON.stringify({
            messageId: MessageType.MOVE_RIGHT_STICK,
            x: data.vector.x,
            y: data.vector.y
          })
        )
        this.rightSendTime = Date.now()
      })
      joyStickRight.on('end', (evt, data) => {
        this.connect.emitUIInteraction(
          JSON.stringify({
            messageId: MessageType.MOVE_RIGHT_STICK,
            x: 0,
            y: 0
          })
        )
      })
    },
    // 适配横屏
    fitIOS() {
      this.orientationChangeHandle()

      window.addEventListener(
        'onorientationchange' in window ? 'orientationchange' : 'resize',
        this.orientationChangeHandle,
        false
      )
    },
    orientationChangeHandle() {
      if (window.orientation === 180 || window.orientation === 0) {
        // this.loadingText = '请解锁手机方向锁定，横屏体验！'
        // this.loadingIcon = 'el-icon-mobile-phone'
        // alert(this.loadingIcon)
        // this.loading = true
        // alert(111)
        this.isShow = true
        // Loading.service(this.options)
      }
      if (window.orientation === 90 || window.orientation === -90) {
        this.loading = false
        this.isShow = false
      }
    }
  }
}
</script>

<style scoped>
.player-container {
  background-color: black;
  width: 100%;
  height: 100%;
  display: block;
  z-index: -1;
}
.btn-container {
  position: absolute;
  display: flex;
  top: 80%;
  left: 0;
  justify-content: flex-end;
  flex-direction: row;
  z-index: 1;
}
</style>
<style>
video:focus {
  outline: -webkit-focus-ring-color auto 0;
}
#joyLeft {
  display: block;
  width: 100px;
  height: 100px;
  transform: translateZ(0);
  z-index: 1000;
  position: absolute;
  top: 62%;
  left: 6%;
}
#joyRight {
  display: block;
  width: 100px;
  height: 100px;
  transform: translateZ(0);
  z-index: 1000;
  position: absolute;
  top: 62%;
  left: 83%;
}
::v-deep .el-loading-mask {
  transform: translateZ(0);
}
::v-deep .back {
  background: url('../../assets/joy-background.png') !important;
  background-size: cover !important;
}
.hidden {
  opacity: 0;
}
</style>
