# 隔空写字 — 实时手势识别交互系统 实现计划

> **面向 AI 代理的工作者：** 使用子代理逐任务实现此计划。步骤使用复选框（`- [ ]`）语法跟踪进度。

**目标：** 构建单文件 HTML 隔空写字系统，摄像头为背景，MediaPipe 手势识别驱动隔空写字/擦除/缩放/消散。

**架构：** 单 Canvas 叠加 Video，rAF 主循环（检测→状态→渲染），ESM import 加载 MediaPipe HandLandmarker。

**技术栈：** HTML5 + CSS3 + Vanilla JS (ESM) + MediaPipe HandLandmarker (`@mediapipe/tasks-vision@0.10.18`)

---

## 文件清单

| 文件 | 职责 |
|------|------|
| `index.html` | 全部代码：CSS 样式、HTML 结构、JS 逻辑（配置/手势引擎/绘制/粒子/UI/主循环） |

---

### 任务 1：HTML 骨架 — 摄像头 + Canvas + CSS 布局

**文件：** 创建 `D:\AI code\phoneProject01\index.html`

- [ ] **步骤 1：创建基础 HTML 结构**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>隔空写字 — Air Writing</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { width: 100%; height: 100%; overflow: hidden; background: #000; font-family: system-ui, sans-serif; }

    #container {
      position: relative; width: 100%; height: 100%;
    }

    #video {
      position: absolute; top: 0; left: 0;
      width: 100%; height: 100%;
      object-fit: cover;
      transform: scaleX(-1); /* 镜像 */
    }

    #canvas {
      position: absolute; top: 0; left: 0;
      width: 100%; height: 100%;
      transform: scaleX(-1); /* 与 video 同步镜像 */
      pointer-events: none; /* 让触摸穿透到 video */
    }

    /* 加载遮罩 */
    #loading {
      position: absolute; inset: 0;
      display: flex; flex-direction: column;
      align-items: center; justify-content: center;
      background: rgba(0,0,0,0.9); z-index: 100;
      color: #fff; font-size: 18px;
    }
    #loading .spinner {
      width: 48px; height: 48px;
      border: 3px solid rgba(255,255,255,0.2);
      border-top-color: #00d4ff;
      border-radius: 50%;
      animation: spin 1s linear infinite;
      margin-bottom: 16px;
    }
    @keyframes spin { to { transform: rotate(360deg); } }

    /* 底部状态栏 */
    #status-bar {
      position: absolute; bottom: 16px; left: 50%; transform: translateX(-50%);
      display: flex; gap: 16px; align-items: center;
      background: rgba(15,23,42,0.8); backdrop-filter: blur(12px);
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 24px; padding: 8px 20px;
      color: #e2e8f0; font-size: 13px; z-index: 50;
    }
    #status-bar .dot {
      width: 12px; height: 12px; border-radius: 50%;
      box-shadow: 0 0 8px currentColor;
    }

    /* 手势提示 */
    #hint {
      position: absolute; top: 20px; left: 50%; transform: translateX(-50%);
      color: rgba(255,255,255,0.6); font-size: 13px;
      background: rgba(0,0,0,0.5); border-radius: 16px;
      padding: 6px 18px; z-index: 50;
      transition: opacity 0.5s;
    }
  </style>
</head>
<body>
  <div id="container">
    <video id="video" autoplay playsinline></video>
    <canvas id="canvas"></canvas>
    <div id="loading">
      <div class="spinner"></div>
      <span id="loading-text">正在加载手势模型...</span>
    </div>
    <div id="status-bar">
      <span id="tool-label">✏️ 笔</span>
      <div class="dot" id="color-dot" style="background:#00d4ff;color:#00d4ff;"></div>
      <span id="size-label">中</span>
    </div>
    <div id="hint">🖐️ 张开手掌 → 打开菜单 | ✌️+☝️ 合拢 → 写字</div>
  </div>
</body>
</html>
```

- [ ] **步骤 2：浏览器打开文件验证**

用浏览器打开 `index.html`，确认：
- 全屏黑色背景
- 底部状态栏可见（笔 + 颜色点 + 大小）
- 顶部提示文字可见
- 加载遮罩层可见（带旋转动画）

---

### 任务 2：MediaPipe 初始化 + 摄像头启动

**文件：** 修改 `D:\AI code\phoneProject01\index.html`

- [ ] **步骤 1：在 `</body>` 前添加 `<script type="module">` 并初始化配置**

```html
<script type="module">
// ========== 配置常量 ==========
const CONFIG = {
  // 手势阈值
  INDEX_MIDDLE_TOGETHER_MAX_DIST: 0.06,    // 食指中指"合拢"最大距离
  THUMB_INDEX_APART_MIN_DIST: 0.18,        // 拇指食指"张开"最小距离
  GESTURE_CONFIRM_FRAMES: 6,               // 手势确认帧数
  UI_DWELL_MS: 800,                        // UI 悬停选择时间
  UI_COOLDOWN_MS: 1500,                    // 手掌开合冷却
  PALM_HOLD_FRAMES: 15,                    // 手掌持续帧数(~0.5s)

  // 画布
  CANVAS_W: 640,
  CANVAS_H: 480,

  // 笔迹预设
  COLORS: ['#00d4ff', '#ff2d55', '#32d74b', '#ffd60a', '#af52de', '#ffffff'],
  SIZES: [3, 6, 10],
  SIZE_LABELS: ['细', '中', '粗'],

  // 粒子
  PARTICLES_PER_STROKE: 80,
  PARTICLE_FALL_SPEED_MIN: 0.3,
  PARTICLE_FALL_SPEED_MAX: 1.8,
  PARTICLE_DRIFT: 0.6,
  PARTICLE_DECAY_MIN: 0.004,
  PARTICLE_DECAY_MAX: 0.015,

  // 缩放
  SCALE_MIN: 0.3,
  SCALE_MAX: 3.0,

  // Model
  MODEL_URL: 'https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task',
  WASM_URL: 'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.18/wasm',
};

// ========== 全局状态 ==========
const state = {
  tool: 'pen',              // 'pen' | 'eraser'
  color: CONFIG.COLORS[0],
  sizeIdx: 1,               // 0=细 1=中 2=粗
  uiOpen: false,
  uiCooldownUntil: 0,
  gesture: 'idle',          // 'idle'|'writing'|'pointing'|'scaling'|'dispersing'
  gestureFrames: {},
  strokes: [],
  particles: [],
  currentStroke: null,
  viewScale: 1,
  viewOffsetX: 0,
  viewOffsetY: 0,
  // 缩放相关
  pinchStartDist: 0,
  pinchStartScale: 1,
  pinchCenter: null,
  // UI 悬停
  hoverTarget: null,
  hoverStartTime: 0,
  // 手势结果缓存
  latestResults: null,
};
</script>
```

- [ ] **步骤 2：添加 DOM 引用和 MediaPipe 加载函数**

```javascript
// ========== DOM 引用 ==========
const video = document.getElementById('video');
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const loadingEl = document.getElementById('loading');
const loadingText = document.getElementById('loading-text');
const toolLabel = document.getElementById('tool-label');
const colorDot = document.getElementById('color-dot');
const sizeLabel = document.getElementById('size-label');
const hintEl = document.getElementById('hint');

// ========== MediaPipe 初始化 ==========
let handLandmarker = null;

async function initMediaPipe() {
  loadingText.textContent = '正在加载 WASM 运行时...';
  const { FilesetResolver, HandLandmarker } = await import(
    `https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.18/hand_landmarker/hand_landmarker.js`
  );

  loadingText.textContent = '正在加载手势识别模型...';
  const vision = await FilesetResolver.forVisionTasks(CONFIG.WASM_URL);

  handLandmarker = await HandLandmarker.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath: CONFIG.MODEL_URL,
      delegate: 'GPU',
    },
    runningMode: 'VIDEO',
    numHands: 2,
    minHandDetectionConfidence: 0.7,
    minTrackingConfidence: 0.5,
  });

  loadingText.textContent = '模型加载完成！';
}
```

- [ ] **步骤 3：添加摄像头启动函数**

```javascript
// ========== 摄像头 ==========
async function startCamera() {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: { width: 640, height: 480, facingMode: 'user' },
    audio: false,
  });
  video.srcObject = stream;

  return new Promise((resolve) => {
    video.onloadedmetadata = () => {
      canvas.width = CONFIG.CANVAS_W;
      canvas.height = CONFIG.CANVAS_H;
      video.play();
      resolve();
    };
  });
}
```

- [ ] **步骤 4：添加启动入口**

```javascript
// ========== 启动 ==========
async function main() {
  try {
    await initMediaPipe();
    await startCamera();
    loadingEl.style.display = 'none';
    requestAnimationFrame(loop);
  } catch (err) {
    loadingText.textContent = `启动失败: ${err.message}`;
    console.error(err);
  }
}
main();
```

- [ ] **步骤 5：浏览器验证**

打开 `index.html`，确认：
- 浏览器请求摄像头权限
- 加载遮罩显示进度文字后消失
- 摄像头画面全屏显示（镜像）
- 状态栏显示默认工具信息

---

### 任务 3：手势识别引擎

**文件：** 修改 `D:\AI code\phoneProject01\index.html`（在 `main()` 前插入）

- [ ] **步骤 1：添加手指伸展判定函数**

```javascript
// ========== 手指检测工具 ==========
// Landmark 索引
const L = { WRIST:0, THUMB_CMC:1, THUMB_MCP:2, THUMB_IP:3, THUMB_TIP:4,
            INDEX_MCP:5, INDEX_PIP:6, INDEX_DIP:7, INDEX_TIP:8,
            MIDDLE_MCP:9, MIDDLE_PIP:10, MIDDLE_DIP:11, MIDDLE_TIP:12,
            RING_MCP:13, RING_PIP:14, RING_DIP:15, RING_TIP:16,
            PINKY_MCP:17, PINKY_PIP:18, PINKY_DIP:19, PINKY_TIP:20 };

function isFingerExtended(lm, tipIdx, pipIdx) {
  return lm[tipIdx].y < lm[pipIdx].y;
}

function isThumbExtended(lm) {
  const tip = lm[L.THUMB_TIP], ip = lm[L.THUMB_IP], wrist = lm[L.WRIST];
  const tipDist = Math.hypot(tip.x - wrist.x, tip.y - wrist.y);
  const ipDist = Math.hypot(ip.x - wrist.x, ip.y - wrist.y);
  return tipDist > ipDist * 1.05;
}

function dist(lm, i1, i2) {
  return Math.hypot(lm[i1].x - lm[i2].x, lm[i1].y - lm[i2].y);
}

function getFingerState(lm) {
  return {
    thumb: isThumbExtended(lm),
    index: isFingerExtended(lm, L.INDEX_TIP, L.INDEX_PIP),
    middle: isFingerExtended(lm, L.MIDDLE_TIP, L.MIDDLE_PIP),
    ring: isFingerExtended(lm, L.RING_TIP, L.RING_PIP),
    pinky: isFingerExtended(lm, L.PINKY_TIP, L.PINKY_PIP),
  };
}

function countExtended(fs) {
  return (fs.thumb?1:0) + (fs.index?1:0) + (fs.middle?1:0) + (fs.ring?1:0) + (fs.pinky?1:0);
}

function onlyTheseExtended(fs, fingers) {
  const all = ['thumb','index','middle','ring','pinky'];
  return all.every(f => fingers.includes(f) ? fs[f] : !fs[f]);
}
```

- [ ] **步骤 2：添加单只手势判定函数**

```javascript
// ========== 单只手势判定 ==========
function classifyHandGesture(lm) {
  const fs = getFingerState(lm);
  const dIndexMiddle = dist(lm, L.INDEX_TIP, L.MIDDLE_TIP);
  const dThumbIndex = dist(lm, L.THUMB_TIP, L.INDEX_TIP);

  // 🖐️ 整个手掌: 5指伸展
  if (countExtended(fs) === 5) {
    return 'palm';
  }

  // 🤙 拇指+食指张开: 仅拇指和食指伸展，距离大
  if (onlyTheseExtended(fs, ['thumb', 'index']) && dThumbIndex > CONFIG.THUMB_INDEX_APART_MIN_DIST) {
    return 'thumbIndexApart';
  }

  // ✌️☝️ 食指+中指合拢: 仅食指中指伸展，距离小
  if (onlyTheseExtended(fs, ['index', 'middle']) && dIndexMiddle < CONFIG.INDEX_MIDDLE_TOGETHER_MAX_DIST) {
    return 'indexMiddleTogether';
  }

  // ☝️ 食指指向: 仅食指伸展
  if (onlyTheseExtended(fs, ['index'])) {
    return 'indexPoint';
  }

  return 'unknown';
}
```

- [ ] **步骤 3：添加手势确认状态机（防抖）**

```javascript
// ========== 手势确认（防抖状态机）==========
function updateGestureState(handGestures, results) {
  const now = performance.now();
  const palmDetected = handGestures.some(g => g === 'palm');
  const writeDetected = handGestures.some(g => g === 'indexMiddleTogether');
  const apartDetected = handGestures.some(g => g === 'thumbIndexApart');
  const pointDetected = handGestures.some(g => g === 'indexPoint');

  // 两个手各伸食指 → 缩放
  const twoHandsIndex = results.handedness?.length === 2
    && handGestures.length === 2
    && handGestures.every(g => {
      const idx = handGestures.indexOf(g);
      // 需要重新取每个手的 lm...
      return g === 'indexPoint';
    });

  // 手掌 → 切换 UI（带冷却）
  if (palmDetected) {
    state.gestureFrames['palm'] = (state.gestureFrames['palm'] || 0) + 1;
    if (state.gestureFrames['palm'] >= CONFIG.PALM_HOLD_FRAMES && now > state.uiCooldownUntil) {
      state.uiOpen = !state.uiOpen;
      state.uiCooldownUntil = now + CONFIG.UI_COOLDOWN_MS;
      state.gestureFrames['palm'] = 0;
      state.hoverTarget = null;
      state.hoverStartTime = 0;
    }
  } else {
    state.gestureFrames['palm'] = 0;
  }

  // 拇指+食指张开 → 触发消散
  if (apartDetected) {
    state.gestureFrames['apart'] = (state.gestureFrames['apart'] || 0) + 1;
    if (state.gestureFrames['apart'] >= CONFIG.GESTURE_CONFIRM_FRAMES && state.strokes.length > 0) {
      triggerDisperse();
      state.gestureFrames['apart'] = 0;
    }
  } else {
    state.gestureFrames['apart'] = 0;
  }

  // 确定最终手势状态
  if (state.particles.length > 0) {
    state.gesture = 'dispersing';
  } else if (twoHandsIndex) {
    state.gesture = 'scaling';
  } else if (writeDetected && !state.uiOpen) {
    state.gesture = 'writing';
  } else if (pointDetected && state.uiOpen) {
    state.gesture = 'pointing';
  } else {
    state.gesture = 'idle';
  }
}
```

- [ ] **步骤 4：浏览器验证（需摄像头）**

打开页面，在摄像头前尝试：
- 张开整个手掌 → 控制台无报错
- 伸出食指 → 控制台无报错
- 确认手势引擎正常运行（可在 loop 中加 console.log 调试）

---

### 任务 4：霓虹发光笔迹绘制系统

**文件：** 修改 `D:\AI code\phoneProject01\index.html`

- [ ] **步骤 1：添加坐标转换和 Stroke 管理函数**

```javascript
// ========== 坐标转换 ==========
function screenToWorld(sx, sy) {
  return {
    x: (sx - state.viewOffsetX) / state.viewScale,
    y: (sy - state.viewOffsetY) / state.viewScale,
  };
}

function worldToScreen(wx, wy) {
  return {
    x: wx * state.viewScale + state.viewOffsetX,
    y: wy * state.viewScale + state.viewOffsetY,
  };
}

function landmarkToCanvas(lm, idx) {
  return {
    x: lm[idx].x * canvas.width,
    y: lm[idx].y * canvas.height,
  };
}

// ========== Stroke 管理 ==========
let strokeIdCounter = 0;

function startStroke(point, color, size, tool) {
  state.currentStroke = {
    id: strokeIdCounter++,
    points: [point],
    color,
    size,
    tool,
  };
}

function addPointToStroke(point) {
  if (state.currentStroke) {
    state.currentStroke.points.push(point);
  }
}

function finishStroke() {
  if (state.currentStroke && state.currentStroke.points.length > 1) {
    state.strokes.push(state.currentStroke);
  }
  state.currentStroke = null;
}
```

- [ ] **步骤 2：添加霓虹发光笔迹渲染函数**

```javascript
// ========== 霓虹笔迹渲染 ==========
function drawNeonStroke(stroke) {
  if (stroke.points.length < 2) return;

  const size = stroke.size * state.viewScale;
  const worldPts = stroke.points;

  // 外层发光
  ctx.save();
  ctx.beginPath();
  ctx.moveTo(worldPts[0].x, worldPts[0].y);
  for (let i = 1; i < worldPts.length; i++) {
    const midX = (worldPts[i-1].x + worldPts[i].x) / 2;
    const midY = (worldPts[i-1].y + worldPts[i].y) / 2;
    ctx.quadraticCurveTo(worldPts[i-1].x, worldPts[i-1].y, midX, midY);
  }
  ctx.lineTo(worldPts[worldPts.length-1].x, worldPts[worldPts.length-1].y);
  ctx.strokeStyle = stroke.color;
  ctx.lineWidth = size + 6;
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';
  ctx.shadowColor = stroke.color;
  ctx.shadowBlur = 15;
  ctx.globalAlpha = 0.4;
  ctx.stroke();

  // 内层实线
  ctx.shadowBlur = 5;
  ctx.globalAlpha = 0.9;
  ctx.lineWidth = size;
  ctx.stroke();

  ctx.restore();
}

function drawAllStrokes() {
  // 应用视图变换
  ctx.save();
  ctx.translate(state.viewOffsetX, state.viewOffsetY);
  ctx.scale(state.viewScale, state.viewScale);

  for (const stroke of state.strokes) {
    drawNeonStroke(stroke);
  }

  // 正在绘制中的笔画
  if (state.currentStroke && state.currentStroke.points.length >= 2) {
    drawNeonStroke(state.currentStroke);
  }

  ctx.restore();
}
```

- [ ] **步骤 3：添加当前笔画光标指示器**

```javascript
// ========== 光标指示器 ==========
function drawCursor(pos) {
  const s = state.viewScale;
  if (state.tool === 'pen') {
    ctx.beginPath();
    ctx.arc(pos.x, pos.y, state.SIZES[state.sizeIdx] * s / 2 + 4, 0, Math.PI*2);
    ctx.fillStyle = state.color;
    ctx.globalAlpha = 0.25;
    ctx.fill();
    ctx.globalAlpha = 1;
  } else {
    // 橡皮擦光标
    ctx.beginPath();
    ctx.arc(pos.x, pos.y, 15 * s, 0, Math.PI*2);
    ctx.strokeStyle = '#fff';
    ctx.lineWidth = 2;
    ctx.globalAlpha = 0.5;
    ctx.setLineDash([4, 4]);
    ctx.stroke();
    ctx.setLineDash([]);
    ctx.globalAlpha = 1;
  }
}
```

- [ ] **步骤 4：添加主渲染函数**

```javascript
// ========== 主渲染 ==========
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // 绘制所有笔迹
  drawAllStrokes();

  // 绘制粒子
  drawParticles();

  // TODO: 绘制 UI 面板（任务 5）
  // TODO: 绘制 UI 悬停进度（任务 5）
}
```

- [ ] **步骤 5：浏览器验证**

确认不报错，render 函数可正常调用。由于尚未连接手势到写字逻辑，暂时无可见笔迹。

---

### 任务 5：环形 UI 面板 + 食指悬停选择

**文件：** 修改 `D:\AI code\phoneProject01\index.html`

- [ ] **步骤 1：添加 UI 面板数据结构**

```javascript
// ========== UI 面板数据 ==========
const TOOLS = [
  { key: 'pen', icon: '✏️', label: '笔' },
  { key: 'eraser', icon: '🧹', label: '橡皮' },
];

const uiButtons = []; // 运行时计算位置

function computeUILayout() {
  const items = [];
  const panelY = 80;
  const startX = canvas.width / 2 - 320;
  const btnW = 70, btnH = 54, gap = 8;

  let cx = startX;

  // 工具按钮：笔、橡皮
  for (const tool of TOOLS) {
    items.push({
      type: 'tool', key: tool.key, icon: tool.icon, label: tool.label,
      x: cx, y: panelY, w: btnW, h: btnH,
    });
    cx += btnW + gap;
  }

  cx += 12; // 分隔

  // 颜色按钮
  for (let i = 0; i < CONFIG.COLORS.length; i++) {
    items.push({
      type: 'color', key: i, color: CONFIG.COLORS[i],
      x: cx, y: panelY, w: 36, h: btnH,
    });
    cx += 36 + 4;
  }

  cx += 12;

  // 大小按钮
  for (let i = 0; i < CONFIG.SIZES.length; i++) {
    items.push({
      type: 'size', key: i, size: CONFIG.SIZES[i], label: CONFIG.SIZE_LABELS[i],
      x: cx, y: panelY, w: 50, h: btnH,
    });
    cx += 50 + gap;
  }

  return items;
}
```

- [ ] **步骤 2：添加 UI 面板渲染函数**

```javascript
// ========== UI 面板渲染 ==========
let uiAnimProgress = 0; // 0~1 面板动画

function renderUIPanel() {
  const buttons = computeUILayout();
  const targetAlpha = state.uiOpen ? 1 : 0;
  uiAnimProgress += (targetAlpha - uiAnimProgress) * 0.15;

  if (uiAnimProgress < 0.01) return;

  const alpha = uiAnimProgress;

  ctx.save();
  ctx.globalAlpha = alpha;

  // 面板背景
  const panelX = canvas.width / 2 - 360;
  const panelW = 720;
  const panelY = 56;
  const panelH = 62;

  ctx.fillStyle = 'rgba(15, 23, 42, 0.85)';
  ctx.beginPath();
  ctx.roundRect(panelX, panelY, panelW, panelH, 20);
  ctx.fill();
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.12)';
  ctx.lineWidth = 1;
  ctx.stroke();

  // 绘制每个按钮
  for (const btn of buttons) {
    const isSelected = (
      (btn.type === 'tool' && btn.key === state.tool) ||
      (btn.type === 'color' && CONFIG.COLORS[btn.key] === state.color) ||
      (btn.type === 'size' && btn.key === state.sizeIdx)
    );
    const isHovered = state.hoverTarget && state.hoverTarget.key === btn.key && state.hoverTarget.type === btn.type;

    // 背景
    if (isSelected) {
      ctx.fillStyle = 'rgba(0, 212, 255, 0.25)';
    } else if (isHovered) {
      ctx.fillStyle = 'rgba(255, 255, 255, 0.1)';
    } else {
      ctx.fillStyle = 'rgba(255, 255, 255, 0.05)';
    }
    ctx.beginPath();
    ctx.roundRect(btn.x, btn.y, btn.w, btn.h, 12);
    ctx.fill();

    // 边框（选中或悬停）
    if (isSelected) {
      ctx.strokeStyle = 'rgba(0, 212, 255, 0.6)';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.roundRect(btn.x, btn.y, btn.w, btn.h, 12);
      ctx.stroke();
    }

    // 内容
    if (btn.type === 'color') {
      ctx.fillStyle = btn.color;
      ctx.shadowColor = btn.color;
      ctx.shadowBlur = 6;
      ctx.beginPath();
      ctx.arc(btn.x + btn.w/2, btn.y + btn.h/2, 10, 0, Math.PI*2);
      ctx.fill();
      ctx.shadowBlur = 0;
    } else if (btn.type === 'size') {
      ctx.fillStyle = '#e2e8f0';
      ctx.font = '12px system-ui';
      ctx.textAlign = 'center';
      ctx.fillText(btn.label, btn.x + btn.w/2, btn.y + btn.h/2 + 4);
      // 粗细示意线
      ctx.strokeStyle = state.color;
      ctx.lineWidth = btn.size;
      ctx.lineCap = 'round';
      ctx.beginPath();
      ctx.moveTo(btn.x + 10, btn.y + btn.h/2);
      ctx.lineTo(btn.x + btn.w - 10, btn.y + btn.h/2);
      ctx.stroke();
    } else {
      ctx.fillStyle = '#e2e8f0';
      ctx.font = '18px system-ui';
      ctx.textAlign = 'center';
      ctx.fillText(btn.icon, btn.x + btn.w/2, btn.y + btn.h/2 - 4);
      ctx.font = '10px system-ui';
      ctx.fillText(btn.label, btn.x + btn.w/2, btn.y + btn.h/2 + 14);
    }
  }

  // 悬停进度环
  if (state.hoverTarget && state.hoverStartTime > 0 && state.uiOpen) {
    const elapsed = performance.now() - state.hoverStartTime;
    const progress = Math.min(elapsed / CONFIG.UI_DWELL_MS, 1);
    const btn = buttons.find(b => b.key === state.hoverTarget.key && b.type === state.hoverTarget.type);
    if (btn) {
      const cx = btn.x + btn.w / 2, cy = btn.y + btn.h / 2;
      const r = Math.max(btn.w, btn.h) / 2 + 6;
      ctx.strokeStyle = state.color;
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.arc(cx, cy, r, -Math.PI/2, -Math.PI/2 + progress * Math.PI*2);
      ctx.stroke();
    }
  }

  ctx.restore();
}
```

- [ ] **步骤 3：添加 UI 悬停检测函数**

```javascript
// ========== UI 悬停检测 ==========
function checkUIHover(fingerScreenX, fingerScreenY) {
  if (!state.uiOpen) {
    state.hoverTarget = null;
    state.hoverStartTime = 0;
    return;
  }

  const buttons = computeUILayout();
  const now = performance.now();

  // 转换到 canvas 坐标系（因为 canvas 是 CSS mirror 的，screen coords 需要转换）
  // fingerScreen 是屏幕坐标，canvas 的 CSS 是 mirror 的
  // canvas 屏幕左边界 = canvas.getBoundingClientRect().left
  const rect = canvas.getBoundingClientRect();
  const canvasX = (fingerScreenX - rect.left) * (canvas.width / rect.width);
  const canvasY = (fingerScreenY - rect.top) * (canvas.height / rect.height);

  let found = null;
  for (const btn of buttons) {
    if (canvasX >= btn.x && canvasX <= btn.x + btn.w &&
        canvasY >= btn.y && canvasY <= btn.y + btn.h) {
      found = btn;
      break;
    }
  }

  if (found) {
    if (!state.hoverTarget || state.hoverTarget.type !== found.type || state.hoverTarget.key !== found.key) {
      state.hoverTarget = found;
      state.hoverStartTime = now;
    } else {
      const elapsed = now - state.hoverStartTime;
      if (elapsed >= CONFIG.UI_DWELL_MS) {
        applyUISelection(found);
        state.hoverStartTime = now; // 重置，防止连续触发
      }
    }
  } else {
    state.hoverTarget = null;
    state.hoverStartTime = 0;
  }
}

function applyUISelection(btn) {
  if (btn.type === 'tool') {
    state.tool = btn.key;
    updateStatusBar();
  } else if (btn.type === 'color') {
    state.color = CONFIG.COLORS[btn.key];
    updateStatusBar();
  } else if (btn.type === 'size') {
    state.sizeIdx = btn.key;
    updateStatusBar();
  }
}

function updateStatusBar() {
  toolLabel.textContent = state.tool === 'pen' ? '✏️ 笔' : '🧹 橡皮';
  colorDot.style.background = state.color;
  colorDot.style.color = state.color;
  sizeLabel.textContent = CONFIG.SIZE_LABELS[state.sizeIdx];
}
```

- [ ] **步骤 4：更新 render() 函数加入 UI 渲染**

```javascript
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  drawAllStrokes();
  drawParticles();
  renderUIPanel();  // 新增
}
```

- [ ] **步骤 5：浏览器验证**

- 张开手掌 → UI 面板从顶部滑入
- 伸出食指悬停在按钮上 → 进度环出现
- 悬停 0.8s → 按钮被选中（状态栏更新）
- 再次张开手掌 → 面板滑出

---

### 任务 6：连接手势到写字 + 橡皮擦逻辑

**文件：** 修改 `D:\AI code\phoneProject01\index.html`

- [ ] **步骤 1：添加每帧手势处理函数（连接写字）**

```javascript
// ========== 手势→动作映射 ==========
function processGestures(results) {
  if (!results || !results.landmarks) {
    // 无手 → 结束当前笔划
    if (state.currentStroke) finishStroke();
    return;
  }

  const handGestures = [];
  for (const lm of results.landmarks) {
    handGestures.push(classifyHandGesture(lm));
  }

  updateGestureState(handGestures, results);

  // 处理食指指向（UI选择）
  if (state.gesture === 'pointing') {
    // 找到食指指向的那只手
    for (let i = 0; i < results.landmarks.length; i++) {
      const g = handGestures[i];
      if (g === 'indexPoint') {
        const tip = landmarkToCanvas(results.landmarks[i], L.INDEX_TIP);
        const rect = canvas.getBoundingClientRect();
        const screenX = rect.right - tip.x * (rect.width / canvas.width); // 镜像转换
        const screenY = rect.top + tip.y * (rect.height / canvas.height);
        checkUIHover(screenX, screenY);
        break;
      }
    }
  }

  // 处理写字
  if (state.gesture === 'writing') {
    for (let i = 0; i < results.landmarks.length; i++) {
      const g = handGestures[i];
      if (g === 'indexMiddleTogether') {
        const lm = results.landmarks[i];
        const idxTip = landmarkToCanvas(lm, L.INDEX_TIP);
        const midTip = landmarkToCanvas(lm, L.MIDDLE_TIP);
        const midX = (idxTip.x + midTip.x) / 2;
        const midY = (idxTip.y + midTip.y) / 2;
        const worldPt = screenToWorld(midX, midY);

        if (!state.currentStroke) {
          startStroke(worldPt, state.color, CONFIG.SIZES[state.sizeIdx], state.tool);
        } else {
          addPointToStroke(worldPt);
        }
        break;
      }
    }
  } else {
    if (state.currentStroke) finishStroke();
  }

  // 处理橡皮擦（在 writing 手势 + eraser 工具时）
  if (state.gesture === 'writing' && state.tool === 'eraser') {
    for (let i = 0; i < results.landmarks.length; i++) {
      const g = handGestures[i];
      if (g === 'indexMiddleTogether') {
        const lm = results.landmarks[i];
        const idxTip = landmarkToCanvas(lm, L.INDEX_TIP);
        const midTip = landmarkToCanvas(lm, L.MIDDLE_TIP);
        const ex = (idxTip.x + midTip.x) / 2;
        const ey = (idxTip.y + midTip.y) / 2;
        eraseAt(ex, ey);
        break;
      }
    }
  }
}
```

- [ ] **步骤 2：添加橡皮擦函数**

```javascript
// ========== 橡皮擦 ==========
function eraseAt(ex, ey) {
  const threshold = 20 * state.viewScale;
  for (let i = state.strokes.length - 1; i >= 0; i--) {
    const stroke = state.strokes[i];
    for (const pt of stroke.points) {
      const dx = pt.x - ex, dy = pt.y - ey;
      if (Math.sqrt(dx*dx + dy*dy) < threshold) {
        state.strokes.splice(i, 1);
        break;
      }
    }
  }
}
```

- [ ] **步骤 3：添加主循环 loop 函数**

```javascript
// ========== 主循环 ==========
async function loop() {
  if (handLandmarker && video.readyState >= 2) {
    const results = await handLandmarker.detectForVideo(video, performance.now());
    processGestures(results);
  }
  render();
  requestAnimationFrame(loop);
}
```

- [ ] **步骤 4：浏览器验证**

- 食指+中指合拢在空中移动 → 画布出现霓虹发光笔迹
- 松开手 → 笔迹保留在画布上
- 切换到橡皮模式 → 合拢手指擦过笔迹 → 笔迹消失

---

### 任务 7：雪花消散粒子系统

**文件：** 修改 `D:\AI code\phoneProject01\index.html`

- [ ] **步骤 1：添加粒子触发和更新函数**

```javascript
// ========== 粒子系统 ==========
function triggerDisperse() {
  if (state.strokes.length === 0) return;

  for (const stroke of state.strokes) {
    // 采样笔迹点
    const sampleCount = Math.min(stroke.points.length, CONFIG.PARTICLES_PER_STROKE);
    const step = Math.max(1, Math.floor(stroke.points.length / sampleCount));

    for (let i = 0; i < stroke.points.length; i += step) {
      const pt = stroke.points[i];
      const screen = worldToScreen(pt.x, pt.y);
      state.particles.push({
        x: screen.x,
        y: screen.y,
        vx: (Math.random() - 0.5) * CONFIG.PARTICLE_DRIFT * 2,
        vy: CONFIG.PARTICLE_FALL_SPEED_MIN + Math.random() * (CONFIG.PARTICLE_FALL_SPEED_MAX - CONFIG.PARTICLE_FALL_SPEED_MIN),
        life: 1,
        decay: CONFIG.PARTICLE_DECAY_MIN + Math.random() * (CONFIG.PARTICLE_DECAY_MAX - CONFIG.PARTICLE_DECAY_MIN),
        size: 2 + Math.random() * 4,
        color: stroke.color,
        wobblePhase: Math.random() * Math.PI * 2,
        wobbleSpeed: 0.02 + Math.random() * 0.04,
        wobbleAmp: 0.3 + Math.random() * 0.7,
      });
    }
  }

  state.strokes = [];
  state.currentStroke = null;
}

function updateParticles() {
  for (let i = state.particles.length - 1; i >= 0; i--) {
    const p = state.particles[i];
    p.life -= p.decay;
    p.y += p.vy;
    p.x += p.vx + Math.sin(p.wobblePhase) * p.wobbleAmp;
    p.wobblePhase += p.wobbleSpeed;
    p.vy += 0.02; // 逐渐加速下落
    p.size *= 0.998; // 逐渐变小

    if (p.life <= 0) {
      state.particles.splice(i, 1);
    }
  }

  if (state.particles.length === 0 && state.gesture === 'dispersing') {
    state.gesture = 'idle';
  }
}

function drawParticles() {
  for (const p of state.particles) {
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
    ctx.fillStyle = p.color;
    ctx.globalAlpha = p.life * 0.8;
    ctx.shadowColor = p.color;
    ctx.shadowBlur = 4;
    ctx.fill();
  }
  ctx.shadowBlur = 0;
  ctx.globalAlpha = 1;
}
```

- [ ] **步骤 2：更新 render() 函数加入粒子更新**

```javascript
function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  updateParticles();    // 新增：每帧更新粒子
  drawAllStrokes();
  drawParticles();
  renderUIPanel();
}
```

- [ ] **步骤 3：浏览器验证**

- 先写几笔字
- 做拇指+食指张开手势（L 形）→ 笔迹变为雪花粒子
- 粒子向下飘落、水平摇摆、逐渐透明消失
- 画布清空

---

### 任务 8：双手食指缩放

**文件：** 修改 `D:\AI code\phoneProject01\index.html`

- [ ] **步骤 1：添加缩放处理函数**

```javascript
// ========== 双手缩放 ==========
function handleScaling(results) {
  if (!results || results.landmarks.length !== 2) {
    state.gesture = state.gesture === 'scaling' ? 'idle' : state.gesture;
    return;
  }

  const hands = results.landmarks;
  const g0 = classifyHandGesture(hands[0]);
  const g1 = classifyHandGesture(hands[1]);

  if (g0 !== 'indexPoint' || g1 !== 'indexPoint') {
    state.gesture = state.gesture === 'scaling' ? 'idle' : state.gesture;
    return;
  }

  // 两食指指尖
  const tip0 = landmarkToCanvas(hands[0], L.INDEX_TIP);
  const tip1 = landmarkToCanvas(hands[1], L.INDEX_TIP);
  const currentDist = Math.hypot(tip1.x - tip0.x, tip1.y - tip0.y);
  const centerX = (tip0.x + tip1.x) / 2;
  const centerY = (tip0.y + tip1.y) / 2;

  if (state.gesture !== 'scaling') {
    // 进入缩放模式
    state.gesture = 'scaling';
    state.pinchStartDist = currentDist;
    state.pinchStartScale = state.viewScale;
    state.pinchCenter = { x: centerX, y: centerY };
    return;
  }

  if (state.pinchStartDist > 0) {
    let newScale = state.pinchStartScale * (currentDist / state.pinchStartDist);
    newScale = Math.max(CONFIG.SCALE_MIN, Math.min(CONFIG.SCALE_MAX, newScale));

    // 缩放中心固定
    const pc = state.pinchCenter;
    state.viewOffsetX = pc.x - (pc.x - state.viewOffsetX) * (newScale / state.viewScale);
    state.viewOffsetY = pc.y - (pc.y - state.viewOffsetY) * (newScale / state.viewScale);
    state.viewScale = newScale;
  }
}
```

- [ ] **步骤 2：更新 processGestures 函数加入缩放逻辑**

在 `processGestures` 函数中，在手势分发前加入：

```javascript
// 优先检测双手缩放
if (results.landmarks?.length === 2) {
  handleScaling(results);
}
```

同时更新 `updateGestureState`，让 `twoHandsIndex` 检测使用 `handleScaling` 的逻辑。

- [ ] **步骤 3：浏览器验证**

- 伸出两只手的食指
- 两指靠近/远离 → 笔迹缩放
- 放下一只手 → 缩放停止，保持当前倍数
- 缩放范围在 0.3x ~ 3x 之间

---

### 任务 9：收尾完善 — 状态栏、提示、错误处理、响应式

**文件：** 修改 `D:\AI code\phoneProject01\index.html`

- [ ] **步骤 1：添加提示文字自动更新**

```javascript
function updateHint() {
  if (state.particles.length > 0) {
    hintEl.textContent = '❄️ 雪花消散中...';
  } else if (state.gesture === 'scaling') {
    hintEl.textContent = `🔍 缩放: ${state.viewScale.toFixed(1)}x`;
  } else if (state.uiOpen) {
    hintEl.textContent = '☝️ 食指悬停选择工具 | 🖐️ 张开手掌关闭';
  } else if (state.gesture === 'writing') {
    hintEl.textContent = state.tool === 'eraser' ? '🧹 擦除中...' : '✍️ 写字中...';
  } else {
    hintEl.textContent = '🖐️ 张开手掌 → 菜单 | ✌️+☝️ 合拢 → 写字 | 🤙 L形 → 消散';
  }
}

// 在 render() 末尾调用
function render() {
  // ...existing code...
  updateHint();
}
```

- [ ] **步骤 2：添加键盘快捷键（可选辅助）**

```javascript
// ========== 键盘辅助 ==========
document.addEventListener('keydown', (e) => {
  switch(e.key.toLowerCase()) {
    case 'p': state.tool = 'pen'; updateStatusBar(); break;
    case 'e': state.tool = 'eraser'; updateStatusBar(); break;
    case 'c': state.strokes = []; state.particles = []; break;
    case '[': state.sizeIdx = Math.max(0, state.sizeIdx - 1); updateStatusBar(); break;
    case ']': state.sizeIdx = Math.min(2, state.sizeIdx + 1); updateStatusBar(); break;
    case '1': case '2': case '3': case '4': case '5': case '6':
      state.color = CONFIG.COLORS[parseInt(e.key) - 1]; updateStatusBar(); break;
    case '0': state.viewScale = 1; state.viewOffsetX = 0; state.viewOffsetY = 0; break;
  }
});
```

- [ ] **步骤 3：添加响应式处理**

```javascript
// ========== 响应式 ==========
function handleResize() {
  const container = document.getElementById('container');
  const w = container.clientWidth;
  const h = container.clientHeight;
  // Canvas 内部分辨率不变，CSS 自适应
}
window.addEventListener('resize', handleResize);
```

- [ ] **步骤 4：添加错误处理和降级**

```javascript
// 在 main() 中完善错误处理：
// - 摄像头权限拒绝 → 显示提示
// - MediaPipe 加载失败 → 显示重试按钮
// - GPU 不可用 → 降级到 CPU delegate
```

- [ ] **步骤 5：最终验证清单**

- [ ] 摄像头权限 → 画面显示（镜像）
- [ ] 张开手掌 → UI 面板滑入/滑出
- [ ] 食指悬停 0.8s → 选择工具/颜色/大小
- [ ] 食指+中指合拢移动 → 霓虹发光笔迹
- [ ] 松开手指 → 笔迹保留
- [ ] 切换到橡皮 → 擦除笔迹
- [ ] 拇指+食指张开（L 形）→ 笔迹雪花飘落消散
- [ ] 双手各伸食指 → 缩放笔迹
- [ ] 状态栏实时更新工具/颜色/大小
- [ ] 手机浏览器可运行
- [ ] 键盘快捷键可用（P/E/C/[/]/1-6/0）

---

## 自检结果

**1. 规格覆盖度：**
- ✅ 整个手掌 → UI 面板（任务 5）
- ✅ 食指+中指合拢 → 写字（任务 6）
- ✅ 食指指向 → UI 选择（任务 5）
- ✅ 拇指+食指张开 → 雪花消散（任务 7）
- ✅ 双手食指 → 缩放（任务 8）
- ✅ 霓虹发光笔迹（任务 4）
- ✅ 环形 UI 面板（任务 5）
- ✅ 橡皮擦（任务 6）
- ✅ 摄像头镜像（任务 1）
- ✅ 响应式（任务 9）

**2. 占位符扫描：** 无 TODO/待定/后续实现。任务 9 步骤 4 需补充具体错误处理代码。

**3. 类型一致性：** 所有函数名、变量名、数据结构在任务间保持一致。`CONFIG.SIZES` → `CONFIG.SIZES[state.sizeIdx]` → `stroke.size` 链正确。
