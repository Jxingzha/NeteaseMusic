<template>
  <div id="app" :class="`mode-${mode}`">
    <img src="./assets/img/bg-1.png" alt="" class="app-bg">
    <img id="play-music-bg" alt="">
    <canvas :width="pageWidth" :height="pageHeight" id="music-data-canvas" ></canvas>
    <div class="main-container">
      <div style="display: inline-block;width: 60%;" v-if="showCover">
        <Playing />
      </div>
      <router-view />
      <PageLeft />
      <Player />
      <Operation />
    </div>
  </div>
</template>

<script>
  import Storage from './assets/utils/Storage';
  import Player from './components/Player';
  import PageLeft from './components/PageLeft';
  import Playing from './components/Playing';
  import Operation from './components/Operation';
  import { loginStatus } from './assets/utils/request';
  import { messageHelp } from "./assets/utils/util";
  import { mapGetters } from 'vuex';

  export default {
    name: 'App',
    components: { Player, PageLeft, Playing, Operation },
    data() {
      return {
        defaultActive: '/',
        pageWidth: 0,
        pageHeight: 0,
      }
    },
    computed: {
      ...mapGetters({
        allSongs: 'getAllSongs',
        showCover: 'isShowCoverImg',
        mode: 'getMode',
      })
    },
    created() {
      window.VUE_APP = this;
      window.QUERY_QQ_TIMES = 1;
      loginStatus();

      if(/Android|webOS|iPhone|iPod|BlackBerry/i.test(navigator.userAgent)) {
        this.$store.dispatch('updateMode', 'mobile');
      }

      // 播放顺序，qq号的一些配置
      if (!Storage.get('orderType')) {
        Storage.set('orderType', 'liebiao');
      }
      this.defaultActive = window.location.hash.split('/')[1];

      Storage.setDefault({
        listen_size: 'size320',
        down_size: 'high',
        down_high: 'sizeflac',
        volume: 1,
        download_info: JSON.stringify({
          count: 0,
          list: [],
        }),
        key_code_map: JSON.stringify({
          PLAY_NEXT: '39',
          PLAY_PREV: '37',
          VOLUME_UP: '38',
          VOLUME_DOWN: '40',
          PLAY: '32',
          QUIT_SIMPLE: '27',
          TO_SIMPLE: ''
        })
      });

      // 初始化一下下载记录
      this.$store.dispatch('updateDownload', { status: 'abortAll'});

      messageHelp('newInfo');
    },
    mounted() {
      const canvas = document.getElementById('music-data-canvas');
      const ctx = canvas.getContext('2d');
      this.pageWidth = window.innerWidth;
      this.pageHeight = window.innerHeight;

      const initWindowOtherData = () => {
        window.musicOtherData = {
          musicRandomMap: {0: []},
          musicRandomMap2: {0: []},
          musicParticleList: [],
          musicBeforeData: {},
        };
      };

      initWindowOtherData();

      this.ctx = ctx;
      ctx.fillStyle = 'yellow';
      ctx.fillRect(100, 100, 100, 100);

      const pDom = document.getElementById('m-player');

      window.onresize = () => {
        this.pageWidth = window.innerWidth;
        this.pageHeight = window.innerHeight;
      };

      // 这个在定时器里运行，捕捉歌曲的音频数据和绘制canvas
      const draw = () => {
        const { musicDataMap = { 0: [] }, AnalyserNode, AudioBufferSourceNode, readNewMusic, trueStartTime = 0  } = window;
        const { musicRandomMap, musicRandomMap2, musicParticleList, musicBeforeData } = window.musicOtherData;
        const drawMusicType = Storage.get('drawMusicType');
        const num = (Storage.get('drawMusicNum') || 64);
        const { ctx, pageWidth, pageHeight } = this;

        if (!AnalyserNode || !AnalyserNode.getByteFrequencyData) {
          return;
        }
        let dataArr = new Uint8Array(AnalyserNode.frequencyBinCount);//用于存放音频数据的数组，其长度是fftsize的一半
        AnalyserNode.getByteFrequencyData(dataArr);// 将音频频域数据复制到传入的Uint8Array数组

        if (drawMusicType === '2') {
          dataArr = [ ...dataArr.reverse(), ...dataArr.reverse() ].filter((v, i) => i % 2);
        }
        // 发现解析有延迟，这里就当作解析到第一个有音频信号的时候再播放
        if (readNewMusic) {
          pDom.pause();
          let i = dataArr.findIndex((v) => v !== 0);
          if (i === -1) {
            const { ctx } = this;
            const { pageWidth, pageHeight } = this;
            ctx.clearRect(0, 0, pageWidth, pageHeight);
            initWindowOtherData();
            return;
          }
          if (i > -1) {
            window.readNewMusic = false;
            window.trueStartTime = AudioBufferSourceNode.context.currentTime;
            if (VUE_APP.$store.getters.isPlaying) {
              setTimeout(() => pDom.play(), 500);
            }
          }
        }
        const t = AudioBufferSourceNode.context.currentTime - (window.trueStartTime || 0);
        musicDataMap[t] = [...dataArr];
        switch (Storage.get('drawMusicStyle') || 'rect') {
          case 'line':
            musicRandomMap[t] = dataArr.map((v) =>  {
              if (v === 0)
                return 0;
              let n = v + Math.random() * 30 - 12;
              if (n < 0)
                return 0;
              if (n > 255)
                return 255;
              return n;
            });
            musicRandomMap2[t] = dataArr.map((v) =>  {
              if (v === 0)
                return 0;
              let n = v + Math.random() * 30 - 18;
              if (n < 0)
                return 0;
              if (n > 255)
                return 255;
              return n;
            });
            break;
          case 'particle':
            let arr = [];
            musicParticleList.forEach((o) => {
              if (typeof o !== 'object' && o.type !== 'particle') {
                return;
              }
              if (o.t < 1 || o.r < 2) {
                return;
              }
              o.x += o.vx;
              o.y -= o.vy;
              o.t = Math.min(o.t - 0.4, o.t * 0.99);
              o.r *= 0.99;
              if (Math.abs(o.vx) > 3) {
                o.tx = !o.tx;
              }
              o.vx += ( Number(o.tx) * 2 - 1) * 0.1;
              arr.push({...o});
            });
            dataArr.map((v, i) => {
              // 取 2% 的幸运儿
              if (v > 0 && Math.random() < 0.02) {
                arr.push({
                  x: pageWidth * i / num,
                  y: pageHeight,
                  r: Math.random() * 5 + (v / 256 * 7),
                  t: v / 256 * 100, // 透明度
                  vx: Math.random() * 5 - 3, // 横向速度
                  tx: Math.random() > 0.5, // 向左还是向右
                  vy: Math.random() * 2 + 2, // 垂直速度
                  type: 'particle',
                })
              }
            });
            window.musicOtherData.musicParticleList = arr;
            musicDataMap[t] = arr;
            break;
        }

        const keys = Object.keys(musicDataMap);
        let index = keys.findIndex((v) => v >= pDom.currentTime);

        if (index <= 0) {
          index = 0;
        }

        const arr = musicDataMap[keys[index]];

        if (index === 0) {
          arr.fill(0);
        }
        ctx.clearRect(0, 0, pageWidth, pageHeight);

        const drawMusicStyle = Storage.get('drawMusicStyle') || 'rect';
        if (arr) {
          let prevObj = {};
          const randomArr = musicRandomMap[keys[index]] || [];
          const randomArr2 = musicRandomMap2[keys[index]] || [];
          arr.forEach((v, i) => {
            ctx.lineCap = 'butt';
            // 泡泡特效
            if (drawMusicStyle === 'particle') {
              if (typeof v !== 'object' && v.type !== 'particle') {
                return;
              }
              ctx.beginPath();
              ctx.arc(v.x,v.y,v.r,0,2*Math.PI);
              ctx.fillStyle = `rgba(255,255,255,${v.t / 100})`;
              ctx.fill();
              return;
            }

            // 曲线、柱状、粒子
            if (typeof v !== 'number') {
              return;
            }
            let [x, y, w, h, y1, y2] = [
              pageWidth * i / num,
              pageHeight - 80 - v / 256 * pageHeight / 2,
              pageWidth * 0.9 / num,
              v / 256 * pageHeight / 2,
              pageHeight - 80 - (randomArr[i] || 0) / 256 * pageHeight / 2,
              pageHeight - 80 - (randomArr2[i] || 0) / 256 * pageHeight / 2,
            ];
            switch (drawMusicStyle) {
              case 'line':
                if (i === 0) {
                  prevObj = {
                    px: x,
                    py: y,
                    cx: x,
                    cy: y,
                    cy1: y1,
                    cy2: y2,
                    py1: y1,
                    py2: y2,
                  };
                } else {
                  const { px, py, py1, cx, cy, cy1, py2, cy2 } = prevObj;
                  ctx.strokeStyle = '#409EFF33';
                  ctx.beginPath();
                  ctx.lineWidth = 10;
                  ctx.moveTo(cx, cy);
                  ctx.quadraticCurveTo(px, py, (x + px) / 2, (y + py) / 2);
                  ctx.stroke();

                  ctx.strokeStyle = '#5cB87a33';
                  ctx.beginPath();
                  ctx.lineWidth = 3;
                  ctx.moveTo(cx, cy1);
                  ctx.quadraticCurveTo(px, py1, (x + px) / 2, (y1 + py1) / 2);
                  ctx.stroke();

                  ctx.strokeStyle = '#E6A23C33';
                  ctx.beginPath();
                  ctx.lineWidth = 5;
                  ctx.moveTo(cx, cy2);
                  ctx.quadraticCurveTo(px, py2, (x + px) / 2, (y2 + py2) / 2);
                  ctx.stroke();
                  prevObj = {
                    px: x,
                    py: y,
                    cx: (x + px) / 2,
                    cy: (y + py) / 2,
                    cy1: (y1 + py1) / 2,
                    cy2: (y2 + py2) / 2,
                    py1: y1,
                    py2: y2,
                  };
                }
                break;
              case 'line2':
                y1 = pageHeight * 0.5 + v / 256 * pageHeight * 0.1;
                y = pageHeight * 0.5 - v / 256 * pageHeight * 0.3;
                if (i === 0) {
                  prevObj = {
                    px: x,
                    py: y,
                    cx: x,
                    cy: y,
                    cy1: y1,
                    py1: y1,
                  };
                } else if (i === 1) {
                  let { px, py, py1 } = prevObj;
                  prevObj = {
                    px: x,
                    py: y,
                    cx: (x + px) / 2,
                    cy: (y + py) / 2,
                    cy1: (y1 + py1) / 2,
                    py1: y1,
                  };
                } else {
                  let color = '#409EFF22';
                  if (y === y1) {
                    color = '#409EFF10'
                  }
                  let { px, py, py1, cx, cy, cy1 } = prevObj;
                  ctx.strokeStyle = color;
                  ctx.beginPath();
                  ctx.lineWidth = 5;
                  ctx.moveTo(cx, cy);
                  ctx.quadraticCurveTo(px, py, (x + px) / 2, (y + py) / 2);
                  ctx.stroke();


                  ctx.beginPath();
                  ctx.strokeStyle = color;
                  ctx.lineWidth = 5;
                  ctx.moveTo(cx, cy1);
                  ctx.quadraticCurveTo(px, py1, (x + px) / 2, (y1 + py1) / 2);
                  ctx.stroke();

                  if (cy1 - cy > 5) {
                    const gradient = ctx.createLinearGradient(x,y,x,y1);
                    gradient.addColorStop(0,'#409EFF00');
                    gradient.addColorStop(0.5,'#409EFF44');
                    gradient.addColorStop(1,'#409EFF00');
                    ctx.strokeStyle = gradient;
                    ctx.beginPath();
                    ctx.lineWidth = 5;
                    ctx.moveTo(cx, cy + 2);
                    ctx.lineTo(cx, cy1 - 2);
                    ctx.stroke();
                  }
                  prevObj = {
                    px: x,
                    py: y,
                    cx: (x + px) / 2,
                    cy: (y + py) / 2,
                    cy1: (y1 + py1) / 2,
                    py1: y1,
                  };
                }
                break;
              case 'particle2':
                ctx.beginPath();
                ctx.arc(x, y, 5, 0, 2*Math.PI);
                ctx.fillStyle = '#409EFF66';
                ctx.fill();
                if (index > 2) {
                  let y2 = pageHeight - 80 - musicDataMap[keys[index-2]][i] / 256 * pageHeight / 2;
                  y2 = (y2 - y) * (y2 - y > 20 ? 1.5 : 2.5) + y;

                  if (y2 < 0)
                    y2 = 0;
                  if (y2 > pageHeight - 80)
                    y2 = pageHeight - 80;
                  ctx.beginPath();
                  ctx.moveTo(x, y2);
                  ctx.lineTo(x, y);
                  const gradient = ctx.createLinearGradient(x,y,x,y2);
                  gradient.addColorStop(0, '#409EFF66');
                  gradient.addColorStop(0.8,'#409EFF33');
                  gradient.addColorStop(1,'#409EFF00');
                  ctx.strokeStyle = gradient;
                  ctx.lineWidth = 8;
                  ctx.lineCap = 'round';
                  ctx.stroke();
                }
                break;
              default:
                const linearGradient= ctx.createLinearGradient(
                  x,
                  pageHeight,
                  x,
                  pageHeight / 2,
                );
                linearGradient.addColorStop(0,"#409EFF33");
                linearGradient.addColorStop(1,"#5cB87a33");
                ctx.fillStyle = linearGradient;
                ctx.fillRect(x, y, w, h);
                break;
            }
          });
        }
      };

      if (Storage.get('showDrawMusic') !== '0') {
        setInterval(draw, 1000 / 60);
      }
    },
    methods: {

    }
  }
</script>

<style lang="scss">
  @import "assets/style/common";
  body {
    overflow: hidden;
  }
  a {
    color: #fffc;
  }
  .hide-scroll {
    overflow-y: auto;

    &::-webkit-scrollbar {
      width: 0;
      height:8px;
      background-color:rgba(0,0,0,0);
    }
  }
  #app {
    height: 100vh;
    min-width: 1200px;

    #music-data-canvas {
      position: absolute;
      z-index: -1;
      top: 0;
      left: 0;
    }

    #play-music-bg {
      position: absolute;
      z-index: -5;
      left: -5vw;
      min-width: 110vw;
      min-height: 110vh;
      bottom: -30%;
      -webkit-filter: blur(50px) brightness(60%);
      -moz-filter: blur(50px) brightness(60%);
      -o-filter: blur(50px) brightness(60%);
      -ms-filter: blur(50px) brightness(60%);
      filter: blur(50px) brightness(60%);
    }

    .app-bg {
      position: relative;
      z-index: -10;
      top: 0;
      left: 0;
      min-width: 100vw;
      min-height: 100vh;
    }

    .main-container {
      position: absolute;
      overflow-y: auto;
      overflow-x: hidden;
      min-width: 1200px;
      height: calc(100vh - 80px);
      top: 0;
      left: 0;
      display: inline-block;
      vertical-align: top;
      padding: 20px 20px 0 20px;
      width: 100%;
      box-sizing: border-box;

      &::-webkit-scrollbar
      {
        width:8px;
        height:8px;
        background-color:rgba(0,0,0,0);
      }
      /*定义滚动条轨道
       内阴影+圆角*/
      &::-webkit-scrollbar-track
      {
        border-radius:10px;
        background-color: rgba(255,255,255,0.1);
      }
      /*定义滑块
       内阴影+圆角*/
      &::-webkit-scrollbar-thumb
      {
        border-radius:10px;
        background-color:rgba(255,255,255,0.5);
      }
    }

    .playing-bg {
      position: absolute;
      height: 76px;
      top: -3px;

      .wave-bg {
        width: 60vw;
        height: 60vw;
        border-radius: 35%;
        position: absolute;
        right: 0;
        top: -30vw;
        animation: waveBg 5s infinite linear;
        background: -webkit-linear-gradient(left, #409EFF33, #409EFF99);
      }
      .wave-bg2 {
        width: 80vw;
        height: 80vw;
        border-radius: 45%;
        position: absolute;
        right: 0;
        top: -40vw;
        animation: waveBg 8s infinite linear;
        background: -webkit-linear-gradient(top, #fff1, #fff2);
      }

      @keyframes waveBg {
        from {
          transform: rotate(0);
        }
        to {
          transform: rotate(360deg);
        }
      }
    }
  }

  /* 部分页面的右侧导航 */
  .right-select-tab-list {
    position: absolute;
    left: -120px;
    color: #fff;

    @for $i from 0 to 5 {
      .tab-item-#{$i} {
        position: absolute;
        white-space: nowrap;
        overflow: hidden;
        right: -120px;
        top: #{$i * 45 + 15}px;
        width: 40px;
        padding: 5px;
        float: right;
        transition: 0.3s linear;
        box-sizing: border-box;
        box-shadow: -5px 5px 5px #0003;

        .iconfont {
          margin-right: 10px;
          transition: 0.3s linear;
        }
      }
    }

    $color: (
      red: #F56C6C,
      blue: #409EFF,
      green: #67C23A,
      yellow: #E6A23C,
      gray: #666666,
    );

    .c-gray:hover {
      color: #fff8 !important;
      .iconfont {
        color: #fff8 !important;
      }
    }

    @each $c,$v in $color {
      .c-#{$c} {
        background: #0001;
        border-left: 5px solid #{$v}33;

        &:hover {
          background: #0001;
          width: 120px;
          cursor: pointer;
          color: #{$v}cc;

          &.selected {
            .iconfont {
              color: #{$v}cc;
            }
          }
          .iconfont {
            color: #{$v}cc;
          }
        }

        &.selected {
          background: #{$v}33;
        }
      }
    }
  }
</style>
