# vue-project

基于vue3+vite+pinia<br>
弧形轮播路径：src/views/Arc-shapedCarousel.vue<br>
不是vue项目，也可以参考其中的js，未使用任何插件，原生写法<br>
可以直接复制<br>

![动图示例](https://github.com/huangfangfang999/Arc-shapedCarousel/blob/main/Arc-shapedCarousel/src/assets/arc-gif.gif)<br>

其中，交互与弧形布局参数都可以调整<br>

```JavaScript
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


### 项目运行

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```
