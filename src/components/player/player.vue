<template>
    <div class="player" v-show="playList.length > 0">
      <transition name="normal"
                  @enter="enter"
                  @after-enter="afterEnter"
                  @leave="leave"
                  @after-leave="afterLeave">
        <div class="normal-player" v-show="fullScreen">
          <div class="background">
            <img width="100%" height="100%" :src="currentSong.image" />
          </div>
          <div class="top">
            <div class="back" @click="back">
              <i class="icon-back"></i>
            </div>
            <h2 class="title" v-html="currentSong.name"></h2>
            <h3 class="subtitle" v-html="currentSong.singer"></h3>
          </div>
          <div class="middle" @touchstart.prevent="middleTouchStart" @touchmove.prevent="middleTouchMove" @touchend.prevent="middleTouchEnd">
            <div class="middle-l" ref="middleL">
              <div class="cd-wrapper" ref="cdWrapper">
                <div class="cd" :class="cdClass">
                  <img class="image" :src="currentSong.image" />
                </div>
              </div>
              <div class="playing-lyric-wrapper">
                <div class="playing-lyric">{{playingLyric}}</div>
              </div>
            </div>
            <scroll class="middle-r" ref="lyricList" :data="currentLyric && currentLyric.lines">
              <div class="lyric-wrapper">
                <div v-if="currentLyric">
                  <p ref="lyricLine"
                     class="text"
                     :class="{ 'current': index === currentLineNum }"
                     v-for="(line, index) in currentLyric.lines"
                     :key="line.time">{{line.txt}}</p>
                </div>
              </div>
            </scroll>
          </div>
          <div class="bottom">
            <div class="dot-wrapper">
              <span class="dot" :class="{ 'active': currentShow === 'cd' }"></span>
              <span class="dot" :class="{ 'active': currentShow === 'lyric' }"></span>
            </div>
            <div class="progress-wrapper">
              <span class="time time-l">{{format(currentTime)}}</span>
              <div class="progress-bar-wrapper">
                <progress-bar :percent="percent" @percentChange="onProgressBarChange"></progress-bar>
              </div>
              <span class="time time-r">{{format(currentSong.duration)}}</span>
            </div>
            <div class="operators">
              <div class="icon i-left">
                <i :class="iconMode" @click="changeMode"></i>
              </div>
              <div class="icon i-left" :class="disable">
                <i class="icon-prev" @click="prev"></i>
              </div>
              <div class="icon i-center" :class="disable">
                <i :class="playIcon" @click="togglePlaying"></i>
              </div>
              <div class="icon i-right" :class="disable">
                <i class="icon-next" @click="next"></i>
              </div>
              <div class="icon i-right">
                <i class="icon" :class="getFavoriteIcon(currentSong)" @click="toggleFavorite(currentSong)"></i>
              </div>
            </div>
          </div>
        </div>
      </transition>
      <transition name="mini">
        <div class="mini-player" v-show="!fullScreen" @click="open">
          <div class="icon">
            <img :class="cdClass" width="40" height="40"  :src="currentSong.image" />
          </div>
          <div class="text">
            <h2 class="name" v-html="currentSong.name"></h2>
            <p class="desc" v-html="currentSong.singer"></p>
          </div>
          <div class="control">
            <progress-circle :radius="radius" :percent="percent">
              <i class="icon-mini" :class="miniIcon" @click.stop="togglePlaying"></i>
            </progress-circle>
          </div>
          <div class="control" @click.stop="showPlayList">
            <i class="icon-playlist"></i>
          </div>
        </div>
      </transition>
      <play-list ref="playList"></play-list>
      <audio ref="audio" :src="currentSong.url" @canplay="ready" @error="error" @timeupdate="timeUpdate" @ended="end"></audio>
    </div>
</template>

<script type="text/ecmascript-6">
import ProgressBar from 'base/progress-bar/progress-bar';
import Scroll from 'base/scroll/scroll';
import ProgressCircle from 'base/progress-circle/progress-circle';
import PlayList from 'components/play-list/play-list';
import { mapGetters, mapMutations, mapActions } from 'vuex';
import animations from 'create-keyframe-animation';
import { prefixStyle } from 'common/js/dom';
import { playMode } from 'common/js/config';
import { playerMixin } from 'common/js/mixin';
import Lyric from 'lyric-parser';

const transform = prefixStyle('transform');
const transformDuration = prefixStyle('transitionDuration');
export default {
  mixins: [playerMixin],
  name: 'Player',
  data() {
    return {
      songReady: false,
      currentTime: 0,
      radius: 32,
      currentLyric: null,
      currentLineNum: 0,
      currentShow: 'cd',
      playingLyric: ''
    };
  },
  computed: {
    ...mapGetters([
      'fullScreen',
      'playing',
      'currentIndex'
    ]),
    playIcon() {
      return this.playing ? 'icon-pause' : 'icon-play';
    },
    miniIcon() {
      return this.playing ? 'icon-pause-mini' : 'icon-play-mini';
    },
    cdClass() {
      return this.playing ? 'play' : 'pause';
    },
    disable() {
      return this.songReady ? '' : 'disable';
    },
    percent() {
      return this.currentTime / this.currentSong.duration;
    }
  },
  created() {
    this.touch = {};
  },
  methods: {
    ...mapMutations({
      setFullScreen: 'SET_FULL_SCREEN'
    }),
    ...mapActions([
      'savePlayHistory'
    ]),
    middleTouchStart(e) {
      this.touch.initiated = true;
      // 用来判断是否是一次移动
      this.touch.moved = false;
      const touches = e.touches[0];
      this.touch.startX = touches.pageX;
      this.touch.startY = touches.pageY;
    },
    middleTouchMove(e) {
      if (!this.touch.initiated) {
        return;
      }
      const touch = e.touches[0];
      const deltaX = touch.pageX - this.touch.startX;
      const deltaY = touch.pageY - this.touch.startY;

      if (Math.abs(deltaY) > Math.abs(deltaX)) {
        return;
      }
      if (!this.touch.moved) {
        this.touch.moved = true;
      }
      const left = this.currentShow === 'cd' ? 0 : -window.innerWidth;
      const offsetWidth = Math.min(0, Math.max(-window.innerWidth, left + deltaX));
      this.touch.percent = Math.abs(offsetWidth / window.innerWidth);
      this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px, 0, 0)`;
      this.$refs.lyricList.$el.style[transformDuration] = 0;
      this.$refs.middleL.style.opacity = 1 - this.touch.percent;
      this.$refs.middleL.style[transformDuration] = 0;
    },
    middleTouchEnd() {
      if (!this.touch.moved) {
        return;
      }
      let offsetWidth;
      let opacity;
      if (this.currentShow === 'cd') {
        if (this.touch.percent > 0.1) {
          offsetWidth = -window.innerWidth;
          opacity = 0;
          this.currentShow = 'lyric';
        } else {
          offsetWidth = 0;
          opacity = 1;
        }
      } else {
        if (this.touch.percent < 0.9) {
          offsetWidth = 0;
          this.currentShow = 'cd';
          opacity = 1;
        } else {
          offsetWidth = -window.innerWidth;
          opacity = 0;
        }
      }
      const time = 300;
      this.$refs.lyricList.$el.style[transform] = `translate3d(${offsetWidth}px, 0, 0)`;
      this.$refs.lyricList.$el.style[transformDuration] = `${time}ms`;
      this.$refs.middleL.style.opacity = opacity;
      this.$refs.middleL.style[transformDuration] = `${time}ms`;
      this.touch.initiated = false;
    },
    showPlayList() {
      this.$refs.playList.show();
    },
    back() {
      this.setFullScreen(false);
    },
    open() {
      this.setFullScreen(true);
    },
    togglePlaying() {
      if (!this.songReady) {
        return;
      }
      this.setPlayingState(!this.playing);
      if (this.currentLyric) {
        this.currentLyric.togglePlay();
      }
    },
    prev() {
      if (!this.songReady) {
        return;
      }
      let index = this.currentIndex - 1;
      if (index === -1) {
        index = this.playList.length - 1;
      }
      this.setCurrentIndex(index);
      if (!this.playing) {
        this.togglePlaying();
      }
      this.songReady = false;
    },
    next() {
      if (!this.songReady) {
        return;
      }
      if (this.playList.length === 1) {
        this.loop();
      } else {
        let index = this.currentIndex + 1;
        if (index === this.playList.length) {
          index = 0;
        }
        this.setCurrentIndex(index);
        if (!this.playing) {
          this.togglePlaying();
        }
      }
      this.songReady = false;
    },
    end() {
      if (!this.mode === playMode.loop) {
        this.loop();
      } else {
        this.next();
      }
    },
    loop() {
      this.$refs.audio.currentTime = 0;
      this.$refs.audio.play();
      if (this.currentLyric) {
        this.currentLyric.seek(0);
      }
    },
    ready() {
      this.songReady = true;
      this.savePlayHistory(this.currentSong);
    },
    error() {
      this.songReady = true;
    },
    timeUpdate(e) {
      this.currentTime = e.target.currentTime;
    },
    format(interval) {
      interval = interval | 0;
      const minute = interval / 60 | 0;
      const second = this._pad(interval % 60);
      return `${minute}:${second}`;
    },
    _pad(num, n = 2) {
      let len = num.toString().length;
      while (len < n) {
        num = '0' + num;
        len++;
      }
      return num;
    },
    onProgressBarChange(percent) {
      const currentTime = this.currentSong.duration * percent;
      this.$refs.audio.currentTime = currentTime;
      if (!this.playing) {
        this.togglePlaying();
      }
      if (this.currentLyric) {
        this.currentLyric.seek(currentTime * 1000);
      }
    },
    enter(el, done) {
      const { x, y, scale } = this._getPosAndScale();

      let animation = {
        0: {
          transform: `translate3d(${x}px, ${y}px, 0) scale(${scale})`
        },
        60: {
          transform: `translate3d(0, 0, 0) scale(1.1)`
        },
        100: {
          transform: `translate3d(0, 0, 0) scale(1)`
        }
      };

      animations.registerAnimation({
        name: 'move',
        animation,
        presets: {
          duration: 400,
          easing: 'linear'
        }
      });

      animations.runAnimation(this.$refs.cdWrapper, 'move', done);
    },
    afterEnter() {
      animations.unregisterAnimation('move');
      this.$refs.cdWrapper.style.animation = '';
    },
    leave(el, done) {
      this.$refs.cdWrapper.style.transition = 'all 0.4s';
      const { x, y, scale } = this._getPosAndScale();
      this.$refs.cdWrapper.style[transform] = `translate3d(${x}px, ${y}px, 0) scale(${scale})`;
      this.$refs.cdWrapper.addEventListener('transitionend', done);
    },
    afterLeave() {
      this.$refs.cdWrapper.style.transition = '';
      this.$refs.cdWrapper.style.transform = '';
    },
    _getPosAndScale() {
      const targetWidth = 40;
      const paddingLeft = 40;
      const paddingBottom = 30;
      const paddingTop = 80;
      const width = window.innerWidth * 0.8;
      const scale = targetWidth / width;
      const x = -(window.innerWidth / 2 - paddingLeft);
      const y = window.innerHeight - paddingTop - (width / 2) - paddingBottom;
      return {x, y, scale};
    },
    getLyric() {
      this.currentSong.getLyric().then((lyric) => {
        if (this.currentSong.lyric !== lyric) {
          return;
        }
        this.currentLyric = new Lyric(lyric, this.handleLyric);
        if (this.playing) {
          this.currentLyric.play();
        }
      }).catch(() => {
        this.currentLyric = null;
        this.playingLyric = '';
        this.currentLineNum = 0;
      });
    },
    handleLyric({lineNum, txt}) {
      this.currentLineNum = lineNum;
      if (lineNum > 5) {
        let lineEle = this.$refs.lyricLine[lineNum - 5];
        this.$refs.lyricList.scrollToElement(lineEle, 1000);
      } else {
        this.$refs.lyricList.scrollToElement(0, 0, 1000);
      }
      this.playingLyric = txt;
    }
  },
  watch: {
    currentSong(newSong, oldSong) {
      if (!newSong.id) {
        return;
      }
      if (newSong.id === oldSong.id) {
        return;
      }
      if (this.currentLyric) {
        this.currentLyric.stop();
      }
      this.$nextTick(() => {
        this.$refs.audio.play();
        this.getLyric();
      });
    },
    playing(newPlaying) {
      this.$nextTick(() => {
        const audio = this.$refs.audio;
        newPlaying ? audio.play() : audio.pause();
      });
    }
  },
  components: {
    ProgressBar, ProgressCircle, PlayList, Scroll
  }
};
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
@import "~common/stylus/variable";
@import "~common/stylus/mixin";
.player{
  .normal-player{
    position: fixed;
    left: 0;
    right: 0;
    top: 0;
    bottom: 0;
    z-index: 150;
    background: $color-background;
    &.normal-enter-active, &.normal-leave-active{
      transition: all 0.4s;
      .top, .bottom{
        transition: all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32);
      }
    }
    &.normal-enter, &.normal-leave-to{
      opacity: 0;
      .top{
        transform: translate3d(0, -100px, 0);
      }
      .bottom{
        transform: translate3d(0, 100px, 0);
      }
    }
    .background{
      position: absolute;
      left: 0;
      top: 0;
      width: 100%;
      height: 100%;
      z-index: -1;
      opacity: 0.6;
      filter: blur(20px);
    }
    .top{
      position: absolute;
      width: 100%;
      margin-bottom: 25px;
      .back{
        position: absolute;
        top: 0px;
        left: 6px;
        z-index: 50;
        .icon-back{
          display: block;
          padding: 9px;
          font-size: $font-size-large-x;
          color: $color-theme;
          transform: rotate(-90deg);
        }
      }
      .title{
        width: 70%;
        margin: 0 auto;
        line-height: 40px;
        text-align: center;
        font-size: $font-size-large;
        color: $color-text;
      }
      .subtitle{
        line-height: 20px;
        text-align: center;
        font-size: $font-size-medium;
        color: $color-text;
      }
    }
    .middle{
      position: fixed;
      width: 100%;
      top: 80px;
      bottom: 170px;
      white-space: nowrap;
      font-size: 0;
      .middle-l{
        display: inline-block;
        vertical-align: top;
        position: relative;
        width: 100%;
        height: 0;
        padding-top: 80%;
        .cd-wrapper{
          position: absolute;
          left: 10%;
          top: 0;
          width: 80%;
          height: 100%;
          .cd{
            position: relative;
            width: 100%;
            height: 100%;
            box-sizing: border-box;
            border: 10px solid rgba(255, 255, 255, 0.1);
            border-radius: 50%;
            &.play{
              animation: rotate 20s linear infinite;
            }
            &.pause{
              animation-play-state: paused;
            }
            .image{
              position: absolute;
              left: 0;
              top: 0;
              width: 100%;
              height: 100%;
              border-radius: 50%;
            }
          }
        }
        .playing-lyric-wrapper{
          width: 80%;
          margin: 30px auto 0 auto;
          overflow: hidden;
          .playing-lyric{
            height: 20px;
            line-height: 20px;
            text-align: center;
            font-size: $font-size-medium;
            color: $color-text-l;
          }
        }
      }
      .middle-r{
        display: inline-block;
        vertical-align: top;
        width: 100%;
        height: 100%;
        overflow: hidden;
        .lyric-wrapper{
          width: 80%;
          margin: 0 auto;
          overflow: hidden;
          text-align: center;
          .text{
            line-height: 32px;
            color: $color-text-l;
            font-size: $font-size-medium;
            &.current{
              color: $color-text;
            }
          }
        }
      }
    }
    .bottom{
      position: absolute;
      bottom: 50px;
      width: 100%;
      .dot-wrapper{
        text-align: center;
        font-size: 0;
        .dot{
          display: inline-block;
          vertical-align: middle;
          margin: 0 4px;
          width: 8px;
          height: 8px;
          border-radius: 50%;
          background: $color-text-l;
          &.active{
            width: 20px;
            border-radius: 5px;
            background: $color-text-ll;
          }
        }
      }
      .progress-wrapper{
        display: flex;
        align-items: center;
        width: 80%;
        margin: 0 auto;
        padding: 10px 0;
        .progress-bar-wrapper{
          flex: 1;
        }
        .time{
          flex: 0 0 30px;
          width: 30px;
          height: 30px;
          line-height: 29px;
          font-size: $font-size-small;
          color: $color-text;
          &.time-l{
            text-align: left;
          }
          &.time-r{
            text-align: right;
          }
        }
      }
      .operators{
        display: flex;
        align-items: center;
        .icon{
          flex: 1;
          color: $color-theme;
          &.disable{
            color: $color-theme-d;
          }
          i{
            font-size: 30px;
          }
        }
        .i-left{
          text-align: right;
        }
        .i-center{
          padding: 0 20px;
          text-align: center;
          i{
            fong-size: 40px;
          }
        }
        .i-right{
          text-align: left;
        }
        .icon-favorite{
          color: $color-sub-theme;
        }
      }
    }
  }
  .mini-player{
    display: flex;
    align-items: center;
    position: fixed;
    left: 0;
    bottom: 0;
    z-index: 180;
    width: 100%;
    height: 60px;
    background: $color-highlight-background;
    &.mini-enter-active, &.mini-leave-active{
      transition: all 0.4s;
    }
    &.mini-enter, &.mini-leave-to{
      opacity: 0;
    }
    .icon{
      flex: 0 0 40px;
      width: 40px;
      padding: 0 10px 0 20px;
      img{
        border-radius: 50%;
        &.play {
          animation: rotate 10s linear infinite;
        }
        &.pause {
          animation-play-state: paused;
        }
      }
    }
    .text{
      /* display: flex;
      flex-directin: column;
      justify-content: center; */
      flex: 1;
      line-height: 20px;
      overflow: hidden;
      .name{
        margin-bottom: 2px;
        no-wrap();
        font-size: $font-size-medium;
        color: $color-text;
      }
      .desc{
        no-wrap();
        font-size: $font-size-small;
        color: $color-text-d;
      }
    }
    .control{
      flex: 0 0 30px;
      width: 30px;
      padding: 0 10px;
      .icon-play-mini, .icon-pause-mini, .icon-playlist{
        font-size: 30px;
        color: $color-theme-d;
      }
      .icon-mini{
        font-size: 32px;
        position: absolute;
        left: 0;
        top: 0;
      }
    }
  }
}
@keyframes rotate{
  0% {
    transform: rotate(0);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
