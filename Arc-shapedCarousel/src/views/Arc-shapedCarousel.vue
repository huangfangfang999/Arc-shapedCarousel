<template>
  <div
    class="arc-wrapper"
    @mousedown="onStart"
    @mousemove="onMove"
    @mouseup="onEnd"
    @mouseleave="onEnd"
    @touchstart="onStart"
    @touchmove="onMove"
    @touchend="onEnd"
  >
    <div
      v-for="cell in visibleList"
      :key="cell.vueKey"
      class="avatar"
      :style="getStyle(cell.slotIndex)"
    >
      <img :src="cell.src" />
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";

import img1 from "@/assets/img/img1.png";
import img2 from "@/assets/img/img2.png";
import img3 from "@/assets/img/img3.png";
import img4 from "@/assets/img/img4.png";

const props = defineProps({
  list: {
    type: Array,
    default: () => [img1, img2, img3, img4],
  },
});

/**
 * 交互与弧形布局参数
 */
const CONFIG = {
  /** 拖动 / 惯性：像素位移换算成「当前项索引」变化的比例，越大滑动越灵敏 */
  MOVE_RATIO: 0.006,
  /** 惯性：每帧速度衰减系数，越接近 1 滑行越远，越小越快停下 */
  FRICTION: 0.92,
  /**
   * 吸附：每帧向目标索引逼近的比例（0~1）。
   * 宜取约 0.08~0.15：过小像裹糖浆，过大则一两帧就「闪」到下一项。
   */
  SNAP_SPEED: 0.11,
  /** 画面：物理 current 与展示 displayCurrent 的跟随系数，越小弧线跟手越「柔」、越慢 */
  DISPLAY_LERP: 0.22,

  /** 弧形：头像沿圆弧的水平展开半径（px），越大左右两侧分得越开 */
  RADIUS: 200,
  /**
   * 弧形：纵向拉伸系数，与 y = -cos(angle) * RADIUS * ARC_HEIGHT 相乘。
   * 越大两端相对中心「往下坠」越明显（弯得更狠）；过大可能与容器高度打架导致裁剪。
   */
  ARC_HEIGHT: 0.42,
  /** 弧形：相邻槽位之间的圆心角（弧度）。越大同一半径下弧越「拱」、弯折越狠 */
  ANGLE_STEP: 0.52,
  /** 弧形：相对中心的偏移每增加 1，缩小多少（与 scale = 1 - |offset| * SCALE 配合） */
  SCALE: 0.18,
  /** 弧形：相对中心的偏移每增加 1，透明度降低多少（与 opacity = 1 - |offset| * OPACITY 配合） */
  OPACITY: 0.35,
};

/**
 * 状态
 */
/** 物理滚动位置（整数索引可为小数），驱动数据与惯性 */
const current = ref(0);
/** 展示用位置，向 current 插值，避免弧形上一帧跳太大 */
const displayCurrent = ref(0);
const velocity = ref(0);

let lastX = 0;
let lastT = 0;
let dragging = false;
let raf = null;
let smoothRaf = null;

const kickDisplaySmooth = () => {
  if (smoothRaf != null) return;

  const tick = () => {
    const diff = current.value - displayCurrent.value;
    displayCurrent.value += diff * CONFIG.DISPLAY_LERP;

    const keepGoing =
      dragging || Math.abs(current.value - displayCurrent.value) > 0.0005;

    if (keepGoing) {
      smoothRaf = requestAnimationFrame(tick);
    } else {
      displayCurrent.value = current.value;
      smoothRaf = null;
    }
  };

  smoothRaf = requestAnimationFrame(tick);
};

/**
 * 无限循环
 */
const loopIndex = (i) => {
  const len = props.list.length;
  return ((i % len) + len) % len;
};

/**
 * 可视窗口与弧形都用 displayCurrent 对齐，避免「图已经换了、弧还没跟上」时整屏闪一下。
 * key 用列表项索引（重复时用 `${index}~${槽位}`），让 Vue 复用同一张图的 DOM，避免槽位 key 不变却批量换 src 导致解码闪白。
 */
const visibleList = computed(() => {
  const center = Math.round(displayCurrent.value);
  const arr = [];
  const seen = new Set();
  for (let i = -3; i <= 3; i++) {
    const index = loopIndex(center + i);
    const vueKey = seen.has(index) ? `${index}~${i}` : index;
    seen.add(index);
    arr.push({
      src: props.list[index],
      slotIndex: i + 3,
      vueKey,
    });
  }
  return arr;
});

/**
 * 获取坐标
 */
const getX = (e) => (e.touches ? e.touches[0].clientX : e.clientX);

/**
 * 开始拖
 */
const onStart = (e) => {
  dragging = true;
  cancelAnimationFrame(raf);

  lastX = getX(e);
  lastT = Date.now();
  velocity.value = 0;
  kickDisplaySmooth();
};

/**
 * 拖动
 */
const onMove = (e) => {
  if (!dragging) return;

  const x = getX(e);
  const now = Date.now();

  const dx = x - lastX;
  const dt = now - lastT || 16;

  velocity.value = dx / dt;
  velocity.value = Math.max(-1.5, Math.min(1.5, velocity.value));

  // ⭐ 关键：方向统一（不会反向）
  current.value -= dx * CONFIG.MOVE_RATIO;

  lastX = x;
  lastT = now;
  kickDisplaySmooth();
};

/**
 * 松手
 */
const onEnd = () => {
  if (!dragging) return;
  dragging = false;
  inertia();
};

/**
 * 惯性
 */
let lastFrame = 0;

const inertia = () => {
  lastFrame = Date.now();

  const step = () => {
    const now = Date.now();
    const dt = now - lastFrame;
    lastFrame = now;

    velocity.value *= CONFIG.FRICTION;

    current.value -= velocity.value * dt * CONFIG.MOVE_RATIO;
    kickDisplaySmooth();

    if (Math.abs(velocity.value) < 0.02) {
      snap();
      return;
    }

    raf = requestAnimationFrame(step);
  };

  step();
};

/**
 * 吸附
 */
const snap = () => {
  const target = Math.round(current.value);

  const step = () => {
    const diff = target - current.value;

    if (Math.abs(diff) < 0.001) {
      current.value = target;
      kickDisplaySmooth();
      return;
    }

    current.value += diff * CONFIG.SNAP_SPEED;
    kickDisplaySmooth();
    raf = requestAnimationFrame(step);
  };

  step();
};

/**
 * 弧形布局（关键修复点）
 */
const getStyle = (i) => {
  const offset =
    i - 3 - (displayCurrent.value - Math.round(displayCurrent.value));

  const angle = offset * CONFIG.ANGLE_STEP;

  const x = Math.sin(angle) * CONFIG.RADIUS;
  const y = -Math.cos(angle) * CONFIG.RADIUS * CONFIG.ARC_HEIGHT;

  const scale = 1 - Math.abs(offset) * CONFIG.SCALE;
  const opacity = 1 - Math.abs(offset) * CONFIG.OPACITY;

  return {
    transform: `translate(${x}px, ${y}px) scale(${scale})`,
    zIndex: 100 - Math.abs(offset),
    opacity,
  };
};
</script>
<style scoped>
.arc-wrapper {
  position: relative;
  height: 220px;
  overflow: hidden;
  touch-action: none;
}

.avatar {
  position: absolute;
  left: 50%;
  top: 50%;
  transform-origin: center;
  /* 位置由 displayCurrent 每帧插值驱动，不再用 transition，避免与跟手逻辑打架 */
  transition: none;
}

.avatar img {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  user-select: none;
  pointer-events: none;
}
</style>
