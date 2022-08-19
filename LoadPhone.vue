<template>
  <div :class="[isShow ? 'show' : 'hidden', 'cover']">
    <div :class="chooseCss">
      <div class="icon loading">
        <span class="iconfont icon-pingmusuoding"></span>
      </div>
      <div class="title">请解锁手机方向锁定，横屏体验！</div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'LoadPhone',
  props: {
    // 是否显示遮罩层
    isShow: {
      type: Boolean,
      default: false
    },
    // 显示哪一种css样式
    chooseCss: {
      type: String,
      default: 'container'
    }
  },
  data() {
    return {}
  }
}
</script>

<style lang="scss" scoped>
.cover {
  position: absolute;
  z-index: 999;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgb(0, 0, 0);
  .container {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-60%, -50%) rotate(-90deg);
    width: 300px;
    height: 120px;
    transform-origin: center;
    text-align: center;
    .title {
      color: #409eff;
      margin-top: 20px;
    }
    .icon {
      font-size: 50px;
      color: #409eff;
      transform: rotate(90deg);
      span {
        font-size: 50px;
        // width: 50px !important;
        // height: 50px !important;
        color: #409eff;
      }
    }
  }
  .container1 {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%) rotate(0);
    width: 300px;
    height: 120px;
    transform-origin: center;
    text-align: center;
    .title {
      color: #409eff;
      margin-top: 20px;
    }
    .icon {
      font-size: 50px;
      color: #409eff;
      // transform: rotate(90deg);
      span {
        // display: block;
        font-size: 50px;
        // width: 50px;
        // height: 50px;
        color: #409eff;
      }
    }
  }
}
.show {
  display: block;
}
.hidden {
  display: none;
}
.loading {
  animation: loading 3s linear infinite;
}
@-webkit-keyframes loading {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

@keyframes loading {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
