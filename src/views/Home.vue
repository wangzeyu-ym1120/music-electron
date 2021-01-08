<template>
  <div class="mixin-components-container">
    <div class="title-container">
      <p class="song-name" v-text="songName"></p>
    </div>

    <div ref="waveform"></div>

    <div class="player-controls">
      <button class="btn controls-btn-prev" @click="playPre">
        <i class="fa fa-backward"></i>
      </button>
      <button class="btn controls-btn--play-pause" @click="playMusic">
        <i v-if="playing" class="fa fa-fw fa-pause"></i>
        <i v-else class="fa fa-fw fa-play"></i>
      </button>
      <button class="btn controls-btn-next" @click="playNext">
        <i class="fa fa-forward"></i>
      </button>
      <button class="btn controls-btn-volume">
        <div class="volum-control">
          <i class="fa fa-volume-up"></i>
          <input type="range" v-model="vol" class="volume" :min="min" :max="max" :style="
            `background-image:linear-gradient( to right, ${fillColor}, ${fillColor} ${percent}, ${emptyColor} ${percent})`
          ">
        </div>
      </button>
    </div>
  </div>
</template>

<script>
  // @ is an alias to /src
  // 可视化音频插件
  import WaveSurfer from 'wavesurfer.js'

  /**
   * node.js 中的 path(路径) 模块 
   * 中文文档 http://nodejs.cn/api/path.html
   * path.basename(path[, ext]) 返回 path 的最后一部分
   * path.dirname(path) 返回 path 的目录名
   * path.resolve([...paths]) 将路径或路径片段的序列解析为绝对路径 默认返回当前工作目录的绝对路径
   */ 
  const path = require('path')
  /**
   * node.js 中的 fs(文件系统) 模块 
   * 中文文档 http://nodejs.cn/api/fs.html
   * fs.readdir(path[, options], callback) 读取目录的内容
   * fs.readFileSync(path[, options]) 同步读取文件
   */
  const fs = window.require('fs')
  /**
   * Stream(流) 是一个抽象接口，Node 中有很多对象实现了这个接口
   * Readable - 可读操作
   */
  const { Readable } = require('stream')
  // 流转blob
  const streamToBlob = require('stream-to-blob')
  // 可以从渲染进程向主进程发送同步或异步消息. 也可以收到主进程的响应
  const { ipcRenderer } = window.require('electron')
  /**
   * 在渲染进程中使用主进程模块
   * remote模块为渲染进程（web页面）和主进程通信（IPC）提供了一种简单方法 
   */
  const remote = window.require('electron').remote


  export default {
    name: 'Music',
    mounted() {
      this.init()
    },
    watch: {
      vol: {
        handler() {
          let rate = (this.vol - this.min) / (this.max - this.min)
          this.percent = 100 * rate + '%'
          this.wavesurfer.setVolume(Number(rate))
        }
      }
    },
    methods: {
      init() {
        // 创建一个wavesurfer实例
        this.createWaveSurfer()
        // 首次加载
        let originPath = remote.process.argv[1]
        originPath = '/Users/ebiz21/Music/网易云音乐/华晨宇 - 好想爱这个世界啊.mp3'
        this.loadMusic(originPath)

        /**
         * 播放其他歌曲
         * 监听主进程senond-instance传过来的第二首歌曲的本地路径
         */
        ipcRenderer.on('path', (event, arg) => {
          const newOriginPath = arg
          this.loadMusic(newOriginPath)
        })
      },
      createWaveSurfer() {
        this.wavesurfer = WaveSurfer.create({
          container: this.$refs.waveform, // 绑定容器 或者使用ID
          barWidth: 2, // 波形宽度
          barHeight: 1, // 波形初始高度
          barGap: 1,
          barRadius: 2,
          waveColor: '#fff', // 波形底色
          progressColor: '#00BFFF', // 执行过后的颜色
          backend: 'MediaElement',
          mediaControls: false, // 播放音频时间轴
          audioRate: '1' // 播放速度
        })
      },
      playPre() {
        this.playFileList(this.filePath, 'pre')
      },
      playMusic() {
        this.playing = !this.playing
        this.wavesurfer.playPause.bind(this.wavesurfer)()
      },
      playNext() {
        this.playFileList(this.filePath, 'next')
      },
      playFileList(diskPath, pos) {
        let isInFiles
        let fileIndex
        let preIndex
        let nextIndex
        let fullPath
        let dirPath = path.dirname(diskPath)
        let basename = path.basename(diskPath)

        fs.readdir(dirPath, (err, files) => {
          let length = files.length
          isInFiles = files.includes(basename)
          fileIndex = files.indexOf(basename)
          if (!isInFiles) {
            return
          }
          if (pos == 'pre') {
            preIndex = fileIndex - 1 < 0 ? length - 1 : fileIndex - 1
          }
          if (pos == 'next') {
            preIndex = fileIndex + 1 >= length ? 0 : fileIndex + 1
          }
          fullPath = path.resolve(dirPath, files[preIndex])
          this.loadMusic(fullPath)
        })
      },
      loadMusic(diskPath) {
        this.filePath = diskPath
        this.songName = path.basename(diskPath)

        let buffer = fs.readFileSync(diskPath)
        let stream = this.bufferToStream(buffer)
        let fileUrl
        streamToBlob(stream).then(res => {
          fileUrl = res
          let filePath = window.URL.createObjectURL(fileUrl)
          this.wavesurfer.load(filePath)
          this.wavesurfer.play()
          this.playing = true
        }).catch(err => {
          console.log(err)
        })
      },
      bufferToStream(binary) {
        const readableInstanceStream = new Readable({
          read() {
            this.push(binary)
            this.push(null)
          }
        })
        return readableInstanceStream
      }
    },
    data() {
      return {
        wavesurfer: null,
        filePath: '',
        songName: '',
        playing: false,
        vol: 30,
        max: 100,
        min: 0,
        fillColor: 'rgba(48, 113, 169, 0.8)',
        emptyColor: 'rgba(48, 113, 169, 0.4)',
        percent: '30%',
      }
    }
  }
</script>

<style lang="less" scoped>
  .mixin-components-container {
    .title-container {
      width: 200px;
      margin: 0 auto;
      overflow: hidden;

      .song-name {
        text-align: center;
        padding-left: 20px;
        font-size: 16px;
        color: #fff;
        display: inline-block;
        white-space: nowrap;
        animation: wordsLoop 10s linear infinite normal;
      }

      @keyframes wordsLoop {
        0% {
          transform: translateX(200px);
          -webkit-transform: translateX(200px);
        }

        100% {
          transform: translateX(-100%);
          -webkit-transform: translateX(-100%);
        }
      }

      @-webkit-keyframes wordsLoop {
        0% {
          transform: translateX(200px);
          -webkit-transform: translateX(200px);
        }

        100% {
          transform: translateX(-100%);
          -webkit-transform: translateX(-100%);
        }
      }
    }

    .player-controls {
      width: 50%;
      margin: 30px auto 0;
      display: flex;
      flex-direction: row;
      justify-content: space-between;

      .btn {
        width: 100px;
        height: 100px;
        padding: 30px;
        color: #fff;
        font-size: 25px;
        background: transparent;
        outline: none;
        border: none;

        &:hover {
          cursor: url(../assets/img/pointer.png), pointer;
        }
      }

      .volum-control {
        display: flex;
        flex-direction: row;
        flex-wrap: nowrap;
        justify-content: flex-start;
        align-items: center;

        &:hover {
          .volume {
            display: block;
          }
        }
      }

      .volume {
        display: none;
        -webkit-app-regino: no-drag;
        width: 100px;
        height: 8px;
        margin-left: 5px;
        appearance: none;
        outline: none;
        border-radius: 8px;

        &::-webkit-slider-thumb {
          appearance: none;
          width: 8px;
          background-color: #3071a9cc;
          border: 8px solid #3071a9cc;
          border-radius: 100%;
          cursor: pointer;
        }
      }
    }
  }
</style>