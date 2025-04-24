<template>
  <view class="container">
    <!-- 输入房间号和加入房间按钮 -->
    <view v-if="!isJoined">
      <input v-model="roomId" placeholder="请输入房间号" class="input-room" />
      <button @click="joinRoom" class="btn">加入房间</button>
    </view>

    <!-- 视频展示区域 -->
    <view v-if="isJoined" class="video-container">
      <view class="video-wrapper">
        <image v-if="localImage" :src="localImage" class="video" mode="aspectFill"></image>
        <view v-else class="video-placeholder">本地视频</view>
      </view>
      <canvas id="remoteVideo" class="video" type="2d" @ready="onCanvasReady" />
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
import { ref } from 'vue'
import { onShow, onHide } from "@dcloudio/uni-app"

const localVideoRef = ref<any>(null)
let animationFrameId: number | null = null
let ctx: any = null
let canvas: any = null
let photoInterval: any = null
let localImage = ref('')

// 房间号绑定
const roomId = ref('')

// 状态管理，判断是否加入房间
const isJoined = ref(false)

// WebSocket连接
let socket: any = null

// 初始化摄像头上下文
onShow(() => {

})

// 加入房间
function joinRoom() {
  if (!roomId.value) {
    uni.showToast({
      title: '请输入房间号',
      icon: 'error',
      duration: 1500
    })
    return
  }

  // 连接WebSocket
  socket = uni.connectSocket({
    url: `ws://192.168.0.60:8080/signal?roomId=${roomId.value}`,
    success: () => {
      console.log('WebSocket连接成功')
    }
  })

  socket.onOpen(() => {
    console.log('WebSocket连接已打开')
    isJoined.value = true
  })

  socket.onError((err: any) => {
    console.error('WebSocket连接错误：', err)
    uni.showToast({
      title: '连接失败：' + err.errMsg,
      icon: 'error',
      duration: 1500
    })
  })

  socket.onMessage((res: any) => {
    try {
      const data = JSON.parse(res.data)
      handleSignaling(data)
    } catch (err) {
      console.error('处理消息失败：', err)
    }
  })
}

function onCanvasReady() {
  const query = uni.createSelectorQuery()
  query.select('#remoteVideo')
    .fields({ node: true, size: true }, function (res: any) {
      // 这里不需要做任何事情，结果会在exec的回调中处理
    })
    .exec((res) => {
      if (res && res[0]) {
        canvas = res[0].node
        ctx = canvas.getContext('2d')

        // 设置canvas尺寸
        const dpr = uni.getSystemInfoSync().pixelRatio
        canvas.width = res[0].width * dpr
        canvas.height = res[0].height * dpr
        ctx.scale(dpr, dpr)
      }
    })
}

function handleSignaling(data: any) {
  switch (data.type) {
    case 'video_frame':
      // 绘制接收到的视频帧
      if (ctx && canvas) {
        const image = new Image()
        image.onload = () => {
          ctx.clearRect(0, 0, canvas.width, canvas.height)
          ctx.drawImage(image, 0, 0, canvas.width, canvas.height)
        }
        image.src = 'data:image/jpeg;base64,' + data.data
      }
      break
    // 可以添加其他信令处理
  }
}

// 开始通话
async function startCall() {
  if (!socket) {
    uni.showToast({
      title: '请先加入房间',
      icon: 'error',
      duration: 1500
    })
    return
  }

  try {
    // 获取平台信息
    const platform = uni.getSystemInfoSync().platform

    // 开始定时获取视频帧
    photoInterval = setInterval(() => {
      if (platform === 'android' || platform === 'ios') {
        // APP平台，使用摄像头
        uni.chooseImage({
          count: 1,
          sizeType: ['compressed'],
          sourceType: ['camera'],
          success: (res) => {
            const tempFilePath = res.tempFilePaths[0]
            localImage.value = tempFilePath

            // 读取图片并转换为base64
            uni.getFileSystemManager().readFile({
              filePath: tempFilePath,
              encoding: 'base64',
              success: (result: any) => {
                if (socket) {
                  socket.send({
                    data: JSON.stringify({
                      type: 'video_frame',
                      data: result.data
                    })
                  })
                }
              },
              fail: (error: any) => {
                console.error('读取图片失败：', error)
              }
            })
          },
          fail: (error: any) => {
            console.error('获取图片失败：', error)
          }
        })
      } else {
        // H5平台，使用getUserMedia
        if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
          navigator.mediaDevices.getUserMedia({ video: true })
            .then(stream => {
              const video = document.createElement('video')
              video.srcObject = stream
              video.play()

              const canvas = document.createElement('canvas')
              const ctx = canvas.getContext('2d')

              video.onloadedmetadata = () => {
                canvas.width = video.videoWidth
                canvas.height = video.videoHeight

                ctx?.drawImage(video, 0, 0)
                const base64 = canvas.toDataURL('image/jpeg', 0.8)

                if (socket) {
                  socket.send({
                    data: JSON.stringify({
                      type: 'video_frame',
                      data: base64.split(',')[1]
                    })
                  })
                }

                // 停止视频流
                stream.getTracks().forEach(track => track.stop())
              }
            })
            .catch(error => {
              console.error('获取视频流失败：', error)
            })
        }
      }
    }, 2000) // 每2秒获取一帧，减少调用频率
  } catch (err) {
    console.error('开始通话失败：', err)
    uni.showToast({
      title: '开始通话失败',
      icon: 'error',
      duration: 1500
    })
  }
}

// 离开房间
function leaveRoom() {
  if (socket) {
    socket.close()
    socket = null
  }

  // 停止视频预览和定时器
  if (photoInterval) {
    clearInterval(photoInterval)
    photoInterval = null
  }

  localImage.value = ''
  isJoined.value = false
  roomId.value = ''
}

// 组件卸载时清理
onHide(() => {
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
  }
  if (socket) {
    socket.close()
  }
  if (photoInterval) {
    clearInterval(photoInterval)
  }
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
}

.video-placeholder {
  width: 100%;
  height: 100%;
  background: #000;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
}
</style>
