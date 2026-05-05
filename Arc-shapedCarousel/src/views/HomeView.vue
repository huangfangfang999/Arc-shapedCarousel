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
      v-for="(item, i) in visibleList"
      :key="i"
      class="avatar"
      :style="getStyle(item.offset + 3)"
    >
      <img :src="item.value" />
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import img1 from "@/assets/img/img1.png";
import img2 from "@/assets/img/img2.png";
import img3 from "@/assets/img/img3.png";
import img4 from "@/assets/img/img4.png";
import img5 from "@/assets/img/img5.png";
import img6 from "@/assets/img/img6.png";
import img7 from "@/assets/img/img7.png";

const props = defineProps({
  list: {
    type: Array,
    default: () => [img1, img2, img3, img4, img5, img6, img7],
  },
});

// =========================
// 核心状态
// =========================
const current = ref(0); // 当前“虚拟索引”（可以是小数）
const velocity = ref(0); // 速度（用于惯性）
let startX = 0;
let lastX = 0;
let isDragging = false;
let rafId = null;

// =========================
// 无限循环处理
// =========================
const visibleCount = 7;

const visibleList = computed(() => {
  const result = [];

  for (let i = -3; i <= 3; i++) {
    const index = Math.round(current.value) + i;

    // ❗关键：边界判断
    if (index >= 0 && index < props.list.length) {
      result.push({
        value: props.list[index],
        offset: i,
      });
    }
  }

  return result;
});

const getLoopIndex = (i) => {
  const len = props.list.length;
  return ((i % len) + len) % len;
};

// =========================
// 拖拽逻辑
// =========================
const onStart = (e) => {
  isDragging = true;
  cancelAnimationFrame(rafId);

  startX = getX(e);
  lastX = startX;
  velocity.value = 0;
};

const onMove = (e) => {
  if (!isDragging) return;

  const x = getX(e);
  const delta = x - lastX;

  current.value -= delta * 0.003;
  velocity.value = delta * 0.2;

  lastX = x;
  const min = 0;
  const max = props.list.length - 1;

  current.value = Math.max(min, Math.min(max, current.value));
};

const onEnd = () => {
  if (!isDragging) return;
  isDragging = false;

  startInertia();
};

// =========================
// 惯性滑动
// =========================
const startInertia = () => {
  const friction = 0.92;

  const step = () => {
    velocity.value *= friction;

    if (Math.abs(velocity.value) < 0.1) {
      snapToNearest();
      return;
    }

    current.value -= velocity.value * 0.01;
    rafId = requestAnimationFrame(step);
  };

  step();
};

// =========================
// 吸附到最近
// =========================
const snapToNearest = () => {
  const target = Math.round(current.value);

  const step = () => {
    const diff = target - current.value;

    if (Math.abs(diff) < 0.001) {
      current.value = target;
      return;
    }

    current.value += diff * 0.1;
    requestAnimationFrame(step);
  };

  step();
};

// =========================
// 工具函数
// =========================
const getX = (e) => {
  return e.touches ? e.touches[0].clientX : e.clientX;
};

// =========================
// 核心：弧形计算
// =========================
const getStyle = (i) => {
  const offset = i - 3 + (Math.round(current.value) - current.value);

  const radius = 120;
  const angle = offset * 0.5;

  const x = Math.sin(angle) * radius;
  const y = -Math.cos(angle) * radius * 0.3;

  const scale = 1 - Math.abs(offset) * 0.2;
  const opacity = 1 - Math.abs(offset) * 0.3;

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
  transition:
    transform 0.2s,
    opacity 0.2s;
}

.avatar img {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  user-select: none;
  pointer-events: none;
}
</style>
