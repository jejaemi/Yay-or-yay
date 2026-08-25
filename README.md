<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<title>Gesture Love</title>
<style>
  * { box-sizing: border-box; }
  html, body {
    margin: 0;
    height: 100%;
    background: #05050a;
    color: #fff;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    overflow: hidden;
  }
  #app {
    display: flex;
    flex-direction: column;
    height: 100vh;
    width: 100vw;
  }

  /* ---- TOP: particle stage ---- */
  #topPanel {
    position: relative;
    flex: 1.1;
    background: #000;
    overflow: hidden;
    border-bottom: 2px solid #1a1a24;
  }
  #particleCanvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    display: block;
  }
  #messageLayer {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: none;
  }
  #messageText {
    font-size: clamp(1.6rem, 6vw, 3.2rem);
    font-weight: 800;
    text-align: center;
    padding: 0 20px;
    background: linear-gradient(90deg, #ff6ec7, #ffd166, #7ee8fa, #ff6ec7);
    background-size: 300% auto;
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
    text-shadow: 0 0 30px rgba(255,255,255,0.15);
    opacity: 0;
    transform: scale(0.85);
    transition: opacity 0.5s ease, transform 0.5s ease;
    animation: shimmer 3s linear infinite;
  }
  #messageText.show {
    opacity: 1;
    transform: scale(1);
  }
  @keyframes shimmer {
    to { background-position: 300% center; }
  }
  #hint {
    position: absolute;
    top: 12px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 0.75rem;
    color: #6b6f80;
    letter-spacing: 0.02em;
    text-align: center;
    width: 90%;
  }
  #gestureBadge {
    position: absolute;
    top: 12px;
    right: 14px;
    background: rgba(255,255,255,0.07);
    border: 1px solid rgba(255,255,255,0.12);
    padding: 6px 12px;
    border-radius: 999px;
    font-size: 0.78rem;
    font-weight: 600;
  }

  /* ---- BOTTOM: camera stage ---- */
  #bottomPanel {
    position: relative;
    flex: 1;
    background: #0d0e14;
    overflow: hidden;
  }
  #video {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    transform: scaleX(-1);
  }
  #handCanvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    transform: scaleX(-1);
    pointer-events: none;
  }
  #placeholder {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    gap: 14px;
    color: #7d818f;
    font-size: 0.9rem;
    text-align: center;
    padding: 24px;
  }
  button {
    background: linear-gradient(90deg, #ff6ec7, #7ee8fa);
    color: #05050a;
    border: none;
    padding: 10px 20px;
    border-radius: 10px;
    font-size: 0.9rem;
    font-weight: 700;
    cursor: pointer;
  }
  button:active { transform: scale(0.97); }
  #statusBar {
    position: absolute;
    bottom: 10px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 0.72rem;
    color: #6b6f80;
    text-align: center;
    width: 92%;
  }
  #legend {
    position: absolute;
    bottom: 32px;
    left: 12px;
    font-size: 1.3rem;
    line-height: 1;
    background: rgba(255,255,255,0.05);
    padding: 8px 14px;
    border-radius: 10px;
  }
</style>
</head>
<body>

<div id="app">
  <div id="topPanel">
    <canvas id="particleCanvas"></canvas>
    <div id="hint">Tunjukin salah satu gestur ke kamera di bawah ✨</div>
    <div id="gestureBadge">Menunggu gestur...</div>
    <div id="messageLayer"><div id="messageText"></div></div>
  </div>

  <div id="bottomPanel">
    <div id="placeholder">
      <div>Nyalain kamera dulu buat mulai deteksi gestur 🤍</div>
      <button id="startBtn">▶️ Mulai Kamera</button>
    </div>
    <video id="video" autoplay playsinline muted></video>
    <canvas id="handCanvas"></canvas>
    <div id="legend">🤟&nbsp;&nbsp;&nbsp;✌️&nbsp;&nbsp;&nbsp;🖐️</div>
    <div id="statusBar"></div>
  </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>

<script>
/* ============ PARTICLE SYSTEM ============ */
const pCanvas = document.getElementById('particleCanvas');
const pctx = pCanvas.getContext('2d');
let particles = [];
const colors = ['#ff6ec7', '#ffd166', '#7ee8fa', '#c1fba4', '#c77dff', '#ff9f9f'];

function resizeParticleCanvas() {
  const rect = pCanvas.getBoundingClientRect();
  pCanvas.width = rect.width * devicePixelRatio;
  pCanvas.height = rect.height * devicePixelRatio;
  pctx.setTransform(devicePixelRatio, 0, 0, devicePixelRatio, 0, 0);
}
resizeParticleCanvas();
window.addEventListener('resize', resizeParticleCanvas);

function spawnParticle(burst = false) {
  const rect = pCanvas.getBoundingClientRect();
  const w = rect.width, h = rect.height;
  particles.push({
    x: burst ? w / 2 + (Math.random() - 0.5) * 80 : Math.random() * w,
    y: burst ? h / 2 + (Math.random() - 0.5) * 80 : h + 10,
    vx: (Math.random() - 0.5) * (burst ? 6 : 0.6),
    vy: burst ? (Math.random() - 0.5) * 6 - 2 : -(0.4 + Math.random() * 0.8),
    r: burst ? 2 + Math.random() * 4 : 1 + Math.random() * 2.5,
    color: colors[Math.floor(Math.random() * colors.length)],
    life: 1,
    decay: burst ? 0.012 + Math.random() * 0.01 : 0.004 + Math.random() * 0.004
  });
}

function ambientLoop() {
  const rect = pCanvas.getBoundingClientRect();
  if (Math.random() < 0.5) spawnParticle(false);
  pctx.clearRect(0, 0, rect.width, rect.height);

  particles.forEach(p => {
    p.x += p.vx;
    p.y += p.vy;
    p.vy *= 0.99;
    p.life -= p.decay;
    pctx.globalAlpha = Math.max(p.life, 0);
    pctx.fillStyle = p.color;
    pctx.shadowColor = p.color;
    pctx.shadowBlur = 8;
    pctx.beginPath();
    pctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
    pctx.fill();
  });
  pctx.globalAlpha = 1;
  particles = particles.filter(p => p.life > 0);

  requestAnimationFrame(ambientLoop);
}
ambientLoop();

function burstParticles(n = 60) {
  for (let i = 0; i < n; i++) spawnParticle(true);
}

/* ============ MESSAGE DISPLAY ============ */
const messageText = document.getElementById('messageText');
const gestureBadge = document.getElementById('gestureBadge');
let messageTimeout = null;

const GESTURES = {
  ily: { emoji: '🤟', text: 'I LOVE YOUU', label: '🤟 ILY' },
  peace: { emoji: '✌️', text: 'MY PRETTY JEAMII', label: '✌️ Peace' },
  open: { emoji: '🖐️', text: 'DATE WITH ME?', label: '🖐️ Open hand' }
};

function showMessage(key) {
  const g = GESTURES[key];
  if (!g) return;
  messageText.textContent = `${g.emoji} ${g.text} ${g.emoji}`;
  messageText.classList.add('show');
  burstParticles(70);
  clearTimeout(messageTimeout);
  messageTimeout = setTimeout(() => {
    messageText.classList.remove('show');
  }, 2200);
}

/* ============ HAND TRACKING ============ */
const video = document.getElementById('video');
const handCanvas = document.getElementById('handCanvas');
const hctx = handCanvas.getContext('2d');
const startBtn = document.getElementById('startBtn');
const placeholder = document.getElementById('placeholder');
const statusBar = document.getElementById('statusBar');

let lastGesture = null;
let stableCount = 0;
let lastTriggerTime = 0;
const STABLE_FRAMES = 6;
const COOLDOWN_MS = 1800;

function dist(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function classifyGesture(lm) {
  const tips = [8, 12, 16, 20];
  const pips = [6, 10, 14, 18];
  const fingerUp = tips.map((tip, i) => lm[tip].y < lm[pips[i]].y - 0.02);
  const [index, middle, ring, pinky] = fingerUp;

  const thumbTipDist = dist(lm[4], lm[17]);
  const thumbBaseDist = dist(lm[2], lm[17]);
  const thumbUp = thumbTipDist > thumbBaseDist * 1.15;

  if (index && middle && ring && pinky && thumbUp) return 'open';
  if (index && middle && !ring && !pinky) return 'peace';
  if (index && !middle && !ring && pinky && thumbUp) return 'ily';
  return null;
}

function drawLandmarks(lm) {
  const rect = handCanvas.getBoundingClientRect();
  hctx.clearRect(0, 0, rect.width, rect.height);
  hctx.fillStyle = '#7ee8fa';
  lm.forEach(pt => {
    hctx.beginPath();
    hctx.arc(pt.x * rect.width, pt.y * rect.height, 3, 0, Math.PI * 2);
    hctx.fill();
  });
}

function onResults(results) {
  const rect = handCanvas.getBoundingClientRect();
  if (handCanvas.width !== rect.width * devicePixelRatio) {
    handCanvas.width = rect.width * devicePixelRatio;
    handCanvas.height = rect.height * devicePixelRatio;
    hctx.setTransform(devicePixelRatio, 0, 0, devicePixelRatio, 0, 0);
  }

  if (!results.multiHandLandmarks || results.multiHandLandmarks.length === 0) {
    hctx.clearRect(0, 0, rect.width, rect.height);
    gestureBadge.textContent = 'Menunggu gestur...';
    stableCount = 0;
    lastGesture = null;
    return;
  }

  const lm = results.multiHandLandmarks[0];
  drawLandmarks(lm);
  const g = classifyGesture(lm);

  if (g === lastGesture) {
    stableCount++;
  } else {
    stableCount = 0;
    lastGesture = g;
  }

  if (g) {
    gestureBadge.textContent = GESTURES[g].label;
  } else {
    gestureBadge.textContent = 'Tangan terdeteksi';
  }

  const now = Date.now();
  if (g && stableCount === STABLE_FRAMES && now - lastTriggerTime > COOLDOWN_MS) {
    showMessage(g);
    lastTriggerTime = now;
  }
}

const hands = new Hands({
  locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
});
hands.setOptions({
  maxNumHands: 1,
  modelComplexity: 1,
  minDetectionConfidence: 0.65,
  minTrackingConfidence: 0.6
});
hands.onResults(onResults);

let camera = null;

startBtn.addEventListener('click', async () => {
  try {
    startBtn.disabled = true;
    startBtn.textContent = 'Memuat...';
    camera = new Camera(video, {
      onFrame: async () => {
        await hands.send({ image: video });
      },
      width: 640,
      height: 480
    });
    await camera.start();
    placeholder.style.display = 'none';
    statusBar.textContent = 'Kamera aktif — tunjukin gesturnya ✨';
  } catch (err) {
    statusBar.textContent = 'Gagal mengakses kamera: ' + err.message;
    startBtn.disabled = false;
    startBtn.textContent = '▶️ Mulai Kamera';
  }
});
</script>

</body>
</html>
