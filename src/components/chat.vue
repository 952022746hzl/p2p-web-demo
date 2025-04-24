<template>
  <view class="container">
    <!-- 输入房间号和加入房间按钮 -->
    <view v-if="!isJoined">
      <input v-model="roomId" placeholder="请输入房间号" class="input-room" />
      <button @click="joinRoom" class="btn">加入房间</button>
    </view>

    <!-- 视频展示区域 -->
    <view class="video-container" v-show="isJoined">
      <view class="video-wrapper">
        <video :ref="(el: HTMLVideoElement | null) => { if (el) localVideo = el }" class="video" autoplay playsinline
          muted @loadedmetadata="handleVideoLoaded"></video>
      </view>
      <view class="video-wrapper">
        <video :ref="(el: HTMLVideoElement | null) => { if (el) remoteVideo = el }" class="video" autoplay
          playsinline></video>
      </view>
    </view>

    <!-- 开始通话按钮 -->
    <view v-if="isJoined">
      <button @click="startCall" class="btn-call">开始通话</button>
    </view>

    <!-- 离开房间按钮 -->
    <view v-if="isJoined">
      <button @click="leaveRoom" class="btn-leave">离开房间</button>
    </view>
  </view>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, nextTick } from 'vue'

const localVideo = ref<HTMLVideoElement | null>(null)
const remoteVideo = ref<HTMLVideoElement | null>(null)
const roomId = ref('')
const isJoined = ref(false)
const isComponentMounted = ref(false)

// WebRTC 相关变量
let peerConnection: RTCPeerConnection | null = null
let localStream: MediaStream | null = null
let socket: WebSocket | null = null

// 处理视频加载完成事件
function handleVideoLoaded() {
  console.log('视频元素加载完成，localVideo:', localVideo.value)
}

// 组件挂载后设置标志
onMounted(() => {
  isComponentMounted.value = true
  console.log('组件已挂载，localVideo:', localVideo.value)
  // 检查视频元素是否存在
  const videoElement = document.querySelector('video')
  console.log('DOM中的视频元素:', videoElement)
})

// 创建 RTCPeerConnection
function createPeerConnection() {
  const configuration: RTCConfiguration = {
    iceServers: [
      { urls: 'stun:123.123.123.123:123' },
      {
        urls: 'turn:123.123.123.123:123',
        username: 'user',
        credential: 'password'
      }
    ],
    iceCandidatePoolSize: 10,
    iceTransportPolicy: 'all',
    bundlePolicy: 'max-bundle' as RTCBundlePolicy,
    rtcpMuxPolicy: 'require' as RTCRtcpMuxPolicy
  }

  console.log('创建RTCPeerConnection，配置:', configuration)
  peerConnection = new RTCPeerConnection(configuration)

  // 添加本地流到连接
  if (localStream) {
    console.log('添加本地媒体轨道到连接')
    localStream.getTracks().forEach(track => {
      console.log(`添加轨道: ${track.kind}`)
      peerConnection?.addTrack(track, localStream!)
    })
  }

  // 处理远程流
  peerConnection.ontrack = (event) => {
    console.log('收到远程媒体流:', event.streams[0])
    console.log('远程轨道数量:', event.streams[0].getTracks().length)
    if (remoteVideo.value) {
      remoteVideo.value.srcObject = event.streams[0]
      console.log('远程视频元素已设置媒体流')
    } else {
      console.error('远程视频元素未找到')
    }
  }

  // ICE 候选处理
  peerConnection.onicecandidate = (event) => {
    console.log('ICE候选事件:', event)
    if (event.candidate) {
      console.log('新的ICE候选:', event.candidate)
      console.log('候选类型:', event.candidate.type)
      console.log('候选协议:', event.candidate.protocol)
      console.log('候选地址:', event.candidate.address)
      if (socket) {
        socket.send(JSON.stringify({
          type: 'candidate',
          candidate: event.candidate
        }))
      }
    } else {
      console.log('ICE候选收集完成')
    }
  }

  // ICE连接状态变化
  peerConnection.oniceconnectionstatechange = () => {
    console.log('ICE连接状态变化:', peerConnection?.iceConnectionState)
    if (peerConnection?.iceConnectionState === 'disconnected' ||
      peerConnection?.iceConnectionState === 'failed') {
      console.log('ICE连接失败，尝试重新协商')
      // 重新创建连接
      if (peerConnection) {
        peerConnection.close()
        peerConnection = null
      }
      setTimeout(() => {
        createPeerConnection()
        startCall()
      }, 1000) // 延迟1秒后重试
    }
  }

  // ICE收集状态变化
  peerConnection.onicegatheringstatechange = () => {
    console.log('ICE收集状态变化:', peerConnection?.iceGatheringState)
  }

  // 信令状态变化
  peerConnection.onsignalingstatechange = () => {
    console.log('信令状态变化:', peerConnection?.signalingState)
  }

  // 连接状态变化
  peerConnection.onconnectionstatechange = () => {
    console.log('连接状态变化:', peerConnection?.connectionState)
    if (peerConnection?.connectionState === 'failed') {
      console.log('连接失败，尝试重新连接')
      if (peerConnection) {
        peerConnection.close()
        peerConnection = null
      }
      setTimeout(() => {
        createPeerConnection()
        startCall()
      }, 1000)
    }
  }
}

// 加入房间
async function joinRoom() {
  if (!roomId.value) {
    alert('请输入房间号')
    return
  }

  if (!isComponentMounted.value) {
    console.error('组件尚未完全挂载')
    return
  }

  try {
    // 获取本地媒体流
    console.log('开始获取本地媒体流...')
    localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true
    })
    console.log('本地媒体流获取成功:', localStream)
    console.log('视频轨道数量:', localStream.getVideoTracks().length)
    console.log('音频轨道数量:', localStream.getAudioTracks().length)

    // 设置加入房间状态
    isJoined.value = true

    // 等待 DOM 更新
    await nextTick()

    if (localVideo.value) {
      console.log('找到本地视频元素')
      localVideo.value.srcObject = localStream
      console.log('本地视频元素已设置媒体流')

      // 添加视频元素事件监听
      localVideo.value.onloadedmetadata = () => {
        console.log('本地视频元数据已加载')
      }
      localVideo.value.onplay = () => {
        console.log('本地视频开始播放')
      }
      localVideo.value.onerror = (error) => {
        console.error('本地视频播放错误:', error)
      }
    } else {
      console.error('本地视频元素未找到，当前localVideo:', localVideo.value)
    }

    // 连接信令服务器
    socket = new WebSocket(`ws://localhost:8080/signal?roomId=${roomId.value}`)

    socket.onopen = () => {
      console.log('WebSocket连接已打开')
    }

    socket.onmessage = async (event) => {
      const message = JSON.parse(event.data)
      console.log('收到信令消息:', message)

      switch (message.type) {
        case 'offer':
          if (!peerConnection) {
            createPeerConnection()
          }
          try {
            const offer = new RTCSessionDescription({
              type: 'offer',
              sdp: message.offer.sdp
            })
            await peerConnection?.setRemoteDescription(offer)
            const answer = await peerConnection?.createAnswer()
            await peerConnection?.setLocalDescription(answer)
            socket?.send(JSON.stringify({
              type: 'answer',
              answer: answer
            }))
          } catch (error) {
            console.error('处理offer时出错:', error)
          }
          break

        case 'answer':
          try {
            if (peerConnection?.signalingState === 'have-local-offer') {
              const answer = new RTCSessionDescription({
                type: 'answer',
                sdp: message.answer.sdp
              })
              await peerConnection?.setRemoteDescription(answer)
            } else {
              console.log('忽略answer，当前信令状态:', peerConnection?.signalingState)
            }
          } catch (error) {
            console.error('处理answer时出错:', error)
          }
          break

        case 'candidate':
          try {
            if (peerConnection?.remoteDescription) {
              const candidate = new RTCIceCandidate(message.candidate)
              await peerConnection?.addIceCandidate(candidate)
            } else {
              console.log('等待远程描述设置完成后再添加候选')
            }
          } catch (error) {
            console.error('处理candidate时出错:', error)
          }
          break
      }
    }

    socket.onerror = (error) => {
      console.error('WebSocket错误:', error)
    }

  } catch (error) {
    console.error('加入房间失败:', error)
    alert('加入房间失败: ' + error)
  }
}

// 开始通话
async function startCall() {
  if (!socket) {
    alert('请先加入房间')
    return
  }

  try {
    console.log('开始创建通话...')
    if (!peerConnection) {
      createPeerConnection()
    }
    console.log('创建offer...')
    const offer = await peerConnection?.createOffer({
      offerToReceiveAudio: true,
      offerToReceiveVideo: true,
      iceRestart: true
    })
    console.log('offer创建成功:', offer)
    await peerConnection?.setLocalDescription(offer)
    console.log('本地描述设置完成')
    socket.send(JSON.stringify({
      type: 'offer',
      offer: offer
    }))
    console.log('offer已发送')
  } catch (error) {
    console.error('开始通话失败:', error)
    alert('开始通话失败: ' + error)
  }
}

// 离开房间
function leaveRoom() {
  if (localStream) {
    localStream.getTracks().forEach(track => track.stop())
    localStream = null
  }

  if (peerConnection) {
    peerConnection.close()
    peerConnection = null
  }

  if (socket) {
    socket.close()
    socket = null
  }

  if (localVideo.value) {
    localVideo.value.srcObject = null
  }

  if (remoteVideo.value) {
    remoteVideo.value.srcObject = null
  }

  isJoined.value = false
  roomId.value = ''
}

// 组件卸载时清理
onUnmounted(() => {
  leaveRoom()
})
</script>

<style scoped>
.container {
  padding: 20px;
}

.input-room {
  width: 100%;
  margin-bottom: 10px;
  padding: 10px;
  font-size: 16px;
}

.btn,
.btn-call,
.btn-leave {
  width: 100%;
  padding: 12px;
  margin-top: 10px;
  font-size: 16px;
  background-color: #007aff;
  color: white;
  border: none;
  border-radius: 6px;
}

.video-container {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
}

.video-wrapper {
  width: 48%;
  height: 240px;
  position: relative;
}

.video {
  width: 100%;
  height: 100%;
  background: #000;
  object-fit: cover;
}
</style>