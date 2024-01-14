<template>
  <div id="root">
    <header>
      <Publicity v-show="!running" />
      <el-button class="res" type="text" @click="showResult = true">
        抽奖结果
      </el-button>
      <el-button class="con" type="text" @click="showConfig = true">
        抽奖配置
      </el-button>
    </header>
    <div id="main" :class="{ mask: showRes }"></div>
    <div id="tags">
      <ul v-for="item in datas" :key="item.key">
        <li>
          <a
            href="javascript:void(0);"
            :style="{
              color: '#fff',
            }"
          >
            {{ item.name ? item.name : item.key }}
            <img v-if="item.photo" :src="item.photo" :width="50" :height="50" />
          </a>
        </li>
      </ul>
    </div>
    <transition name="bounce">
      <div id="resbox" v-show="showRes">
        <p @click="showRes = false">{{ categoryName }}抽奖结果：</p>
        <div class="container">
          <span
            v-for="item in resArr"
            :key="item"
            class="itemres"
            :style="resCardStyle"
            :data-id="item"
            @click="showRes = false"
            :class="{
              numberOver:
                !!photos.find((d) => d.id === item) ||
                !!list.find((d) => d.key === item),
            }"
          >
            <span class="cont" v-if="!photos.find((d) => d.id === item)">
              <span
                v-if="!!list.find((d) => d.key === item)"
                :style="{
                  fontSize: '40px',
                }"
              >
                {{ list.find((d) => d.key === item).name }}
              </span>
              <span v-else>
                {{ item }}
              </span>
            </span>
            <img
              v-if="photos.find((d) => d.id === item)"
              :src="photos.find((d) => d.id === item).value"
              alt="photo"
              :width="160"
              :height="160"
            />
          </span>
        </div>
      </div>
    </transition>

    <el-button
      class="audio"
      type="text"
      @click="
        () => {
          playAudio(!audioPlaying);
        }
      "
    >
      <i
        class="iconfont"
        :class="[audioPlaying ? 'iconstop' : 'iconplay1']"
      ></i>
    </el-button>

    <LotteryConfig :visible.sync="showConfig" @resetconfig="reloadTagCanvas" />
    <Tool
      @toggle="toggle"
      @resetConfig="reloadTagCanvas"
      @getPhoto="getPhoto"
      :running="running"
      :closeRes="closeRes"
    />
    <Result :visible.sync="showResult"></Result>

    <span class="copy-right">
      Copyright©zhangyongfeng5350@gmail.com
    </span>

    <audio
      id="audiobg"
      preload="auto"
      controls
      autoplay
      loop
      @play="playHandler"
      @pause="pauseHandler"
    >
      <source :src="audioSrc" />
      你的浏览器不支持audio标签
    </audio>
  </div>
</template>
<script>
import LotteryConfig from '@/components/LotteryConfig';
import Publicity from '@/components/Publicity';
import Tool from '@/components/Tool';
import bgaudio from '@/assets/bg.mp3';
import beginaudio from '@/assets/begin.mp3';
import {
  getData,
  configField,
  resultField,
  newLotteryField,
  conversionCategoryName,
  listField,
} from '@/helper/index';
import { luckydrawHandler } from '@/helper/algorithm';
import Result from '@/components/Result';
import { database, DB_STORE_NAME } from '@/helper/db';
export default {
  name: 'App',

  components: { LotteryConfig, Publicity, Tool, Result },

  computed: {
    resCardStyle() {
      const style = { fontSize: '30px' };
      const { number } = this.config;
      if (number < 100) {
        style.fontSize = '100px';
      } else if (number < 1000) {
        style.fontSize = '80px';
      } else if (number < 10000) {
        style.fontSize = '60px';
      }
      return style;
    },
    config: {
      get() {
        return this.$store.state.config;
      },
    },
    result: {
      get() {
        return this.$store.state.result;
      },
      set(val) {
        this.$store.commit('setResult', val);
      },
    },
    list() {
      return this.$store.state.list;
    },
    allresult() {
      let allresult = [];
      for (const key in this.result) {
        if (this.result.hasOwnProperty(key)) {
          const element = this.result[key];
          allresult = allresult.concat(element);
        }
      }
      return allresult;
    },
    datas() {
      const { number } = this.config;
      const nums = number >= 1500 ? 500 : this.config.number;
      const configNum = number > 1500 ? Math.floor(number / 3) : number;
      const randomShowNums = luckydrawHandler(configNum, [], nums);
      const randomShowDatas = randomShowNums.map((item) => {
        const listData = this.list.find((d) => d.key === item);
        const photo = this.photos.find((d) => d.id === item);
        return {
          key: item * (number > 1500 ? 3 : 1),
          name: listData ? listData.name : '',
          photo: photo ? photo.value : '',
        };
      });
      return randomShowDatas;
    },
    categoryName() {
      return conversionCategoryName(this.category);
    },
    photos() {
      return this.$store.state.photos;
    },
  },
  created() {
    const data = getData(configField);
    if (data) {
      this.$store.commit('setConfig', Object.assign({}, data));
    }
    const result = getData(resultField);
    if (result) {
      this.$store.commit('setResult', result);
    }

    const newLottery = getData(newLotteryField);
    if (newLottery) {
      const config = this.config;
      newLottery.forEach((item) => {
        this.$store.commit('setNewLottery', item);
        if (!config[item.key]) {
          this.$set(config, item.key, 0);
        }
      });
      this.$store.commit('setConfig', config);
    }

    const list = getData(listField);
    if (list) {
      this.$store.commit('setList', list);
    }
  },

  data() {
    return {
      running: false,
      showRes: false,
      showConfig: false,
      showResult: false,
      resArr: [],
      category: '',
      audioPlaying: false,
      audioSrc: bgaudio,
    };
  },
  watch: {
    photos: {
      deep: true,
      handler() {
        this.$nextTick(() => {
          this.reloadTagCanvas();
        });
      },
    },
  },
  mounted() {
    this.startTagCanvas();
    setTimeout(() => {
      this.getPhoto();
    }, 1000);
    window.addEventListener('resize', this.reportWindowSize);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.reportWindowSize);
  },
  methods: {
    reportWindowSize() {
      const AppCanvas = this.$el.querySelector('#rootcanvas');
      if (AppCanvas.parentElement) {
        AppCanvas.parentElement.removeChild(AppCanvas);
      }
      this.startTagCanvas();
    },
    playHandler() {
      this.audioPlaying = true;
    },
    pauseHandler() {
      this.audioPlaying = false;
    },
    playAudio(type) {
      if (type) {
        this.$el.querySelector('#audiobg').play();
      } else {
        this.$el.querySelector('#audiobg').pause();
      }
    },
    loadAudio() {
      this.$el.querySelector('#audiobg').load();
      this.$nextTick(() => {
        this.$el.querySelector('#audiobg').play();
      });
    },
    getPhoto() {
      database.getAll(DB_STORE_NAME).then((res) => {
        if (res && res.length > 0) {
          this.$store.commit('setPhotos', res);
        }
      });
    },
    speed() {
      return [0.1 * Math.random() + 0.01, -(0.1 * Math.random() + 0.01)];
    },
    createCanvas() {
      const canvas = document.createElement('canvas');
      canvas.width = document.body.offsetWidth;
      canvas.height = document.body.offsetHeight;
      canvas.id = 'rootcanvas';
      this.$el.querySelector('#main').appendChild(canvas);
    },
    startTagCanvas() {
      this.createCanvas();
      const { speed } = this;
      window.TagCanvas.Start('rootcanvas', 'tags', {
        textColour: null,
        initial: speed(),
        dragControl: 1,
        textHeight: 20,
        noSelect: true,
        lock: 'xy',
      });
    },
    reloadTagCanvas() {
      window.TagCanvas.Reload('rootcanvas');
    },
    closeRes() {
      this.showRes = false;
    },
    toggle(form) {
      const { speed, config } = this;
      if (this.running) {
        this.audioSrc = bgaudio;
        this.loadAudio();

        window.TagCanvas.SetSpeed('rootcanvas', speed());
        this.showRes = true;
        this.running = !this.running;
        this.$nextTick(() => {
          this.reloadTagCanvas();
        });
      } else {
        this.showRes = false;
        if (!form) {
          return;
        }

        this.audioSrc = beginaudio;
        this.loadAudio();

        const { number } = config;
        const { category, mode, qty, remain, allin } = form;
        let num = 1;
        if (mode === 1 || mode === 5) {
          num = mode;
        } else if (mode === 0) {
          num = remain;
        } else if (mode === 99) {
          num = qty;
        }
        const resArr = luckydrawHandler(
          number,
          allin ? [] : this.allresult,
          num
        );
        this.resArr = resArr;

        this.category = category;
        if (!this.result[category]) {
          this.$set(this.result, category, []);
        }
        const oldRes = this.result[category] || [];
        const data = Object.assign({}, this.result, {
          [category]: oldRes.concat(resArr),
        });
        this.result = data;
        window.TagCanvas.SetSpeed('rootcanvas', [5, 1]);
        this.running = !this.running;
      }
    },
  },
};
</script>
<style lang="scss">
#root {
  // 主体样式
  height: 100%;                   // 设置为父元素的100%
  position: relative;             // 相对定位
  background-image: url('./assets/bg01.jpg');  // 背景图片
  background-size: cover;         // 背景图片覆盖整个元素，保持宽高比
  background-position: center center;  // 背景图片居中显示
  background-repeat: no-repeat;   // 背景图片不重复
  .mask {
    -webkit-filter: blur(5px);    // 为webkit浏览器添加模糊效果
    filter: blur(5px);            // 为元素添加5像素的模糊效果
  }
  header {
    height: 50px;                 // 元素高度
    line-height: 50px;            // 文本垂直居中
    position: relative;           // 相对定位
    .el-button {
      position: absolute;         // 绝对定位
      top: 17px;                  // 距离顶部17像素
      padding: 0;                 // 内边距设置为0
      z-index: 9999;              // 顶层显示
      &.con {
        color: #fff;            // 白色
        font-weight: bold;        // 加粗
        right: 20px;              // 当el-button同时有class con时，距离右边20像素
      }
      &.res {
        color: #fff;            // 白色
        font-weight: bold;        // 加粗
        right: 100px;             // 当el-button同时有class res时，距离右边100像素
      }
    }
  }
  .audio {
    position: absolute;         // 绝对定位
    top: 100px;                 // 距离顶部100像素
    right: 30px;                // 距离右边30像素
    width: 40px;                // 宽度
    height: 40px;               // 高度
    line-height: 40px;          // 文本垂直居中
    border: 1px solid #fff;   // 白色边框
    border-radius: 50%;         // 圆形
    padding: 0;                 // 内边距为0
    text-align: center;         // 文本居中对齐

    .iconfont {
      position: relative;       // iconfont类元素定位为相对定位
      left: 1px;                // iconfont类元素向左偏移1像素
    }
  }
  .copy-right {
    position: absolute;         // 绝对定位
    right: 0;                   // 紧贴右边
    bottom: 0;                  // 紧贴底部
    color: #ccc;              // 文字颜色
    font-size: 12px;            // 字体大小
  }
  .bounce-enter-active {
    animation: bounce-in 1.5s;  // 动画，持续1.5秒
  }
  .bounce-leave-active {
    animation: bounce-in 0s reverse; // 反向动画，持续0秒
  }
}

#main {
  height: 100%;
}

#resbox {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 1280px;
  transform: translateX(-50%) translateY(-50%);
  text-align: center;
  p {
    color: red;
    font-size: 50px;
    line-height: 120px;
  }
  .container {
    display: flex;
    justify-content: center;
    flex-wrap: wrap;
  }
  .itemres {
    background: #fff;
    width: 160px;
    height: 160px;
    border-radius: 4px;
    border: 1px solid #ccc;
    line-height: 160px;
    font-weight: bold;
    margin-right: 20px;
    margin-bottom: 20px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    .cont {
      display: flex;
      justify-content: center;
      align-items: center;
    }
    &.numberOver::before {
      content: attr(data-id);
      width: 30px;
      height: 22px;
      line-height: 22px;
      background-color: #fff;
      position: absolute;
      bottom: 0;
      left: 0;
      font-size: 14px;
      // border-radius: 50%;
      z-index: 1;
    }
  }
}
</style>
