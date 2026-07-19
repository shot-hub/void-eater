# void-eater
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>CITY HOLE 3D</title>
<style>
  :root {
    --bg: #bfe3f2;
    --panel: rgba(255, 255, 255, 0.92);
    --text: #26333d;
    --accent: #1fb6a8;
    --accent-dark: #128f84;
  }
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  html, body {
    margin: 0; padding: 0; width: 100%; height: 100%;
    background: var(--bg); overflow: hidden;
    font-family: 'Helvetica Neue', Arial, sans-serif;
    touch-action: none; user-select: none;
    position: fixed; inset: 0;
  }
  #canvasWrap { position: fixed; inset: 0; }
  canvas { display: block; width: 100%; height: 100%; touch-action: none; }

  #ui-layer { position: fixed; inset: 0; pointer-events: none; z-index: 10; }
  .hud { position: absolute; top: 24px; left: 24px; right: 24px; display: flex; justify-content: space-between; align-items: flex-start; }
  .hud-box { background: var(--panel); border-radius: 20px; padding: 12px 28px; text-align: center; box-shadow: 0 6px 18px rgba(30,50,60,0.08); }
  .hud-label { font-size: 11px; font-weight: 700; color: #a4b0b6; letter-spacing: 2px; margin-bottom: 2px; }
  .hud-value { font-size: 26px; font-weight: 800; color: var(--text); }
  #timer-box.danger .hud-value { color: #e0574a; }

  .leaderboard { position: absolute; top: 96px; right: 24px; background: var(--panel); border-radius: 18px; padding: 14px 18px; box-shadow: 0 6px 18px rgba(30,50,60,0.08); min-width: 150px; }
  .lb-row { display: flex; align-items: center; justify-content: space-between; margin-bottom: 9px; font-size: 13px; font-weight: 600; color: #9aa5aa; }
  .lb-row:last-child { margin-bottom: 0; }
  .lb-row.me { color: var(--accent); font-weight: 700; }
  .lb-dot { width: 10px; height: 10px; border-radius: 50%; margin-right: 8px; flex-shrink: 0; }
  .lb-name { flex: 1; text-align: left; }
  .lb-score { font-family: monospace; font-size: 14px; }

  #overlay { position: fixed; inset: 0; z-index: 20; background: rgba(40, 55, 65, 0.55); backdrop-filter: blur(6px); display: flex; align-items: center; justify-content: center; pointer-events: auto; }
  #overlay.hidden { display: none; }
  .panel { background: #fff; padding: 44px 40px; border-radius: 32px; text-align: center; box-shadow: 0 24px 60px rgba(20,35,45,0.25); width: 90%; max-width: 400px; }
  .title { font-size: 40px; font-weight: 800; color: var(--text); margin: 0 0 10px; letter-spacing: -1.5px; }
  .title span { color: var(--accent); }
  .desc { font-size: 14px; color: #93a0a6; margin-bottom: 32px; font-weight: 600; line-height: 1.6; }
  .result-rank { font-size: 30px; font-weight: 800; color: var(--accent); margin: 22px 0 6px; }
  .result-score { font-size: 19px; font-weight: 700; color: var(--text); margin-bottom: 32px; }
  button { background: var(--accent); color: #fff; border: none; border-radius: 100px; font-size: 18px; font-weight: 800; padding: 17px 48px; cursor: pointer; box-shadow: 0 6px 0 var(--accent-dark), 0 14px 20px rgba(31,182,168,0.25); width: 100%; }
  button:active { transform: translateY(6px); box-shadow: 0 0 0 var(--accent-dark); }

  #joystick { position: fixed; z-index: 15; pointer-events: none; width: 104px; height: 104px; margin: -52px 0 0 -52px; display: none; }
  #joystick .base { position: absolute; inset: 0; border-radius: 50%; border: 2px solid rgba(31,182,168,0.35); background: rgba(31,182,168,0.06); }
  #joystick .knob { position: absolute; width: 46px; height: 46px; border-radius: 50%; left: 29px; top: 29px; background: rgba(31,182,168,0.45); border: 2px solid rgba(31,182,168,0.85); }

  #loading { position: fixed; inset: 0; z-index: 30; background: #bfe3f2; display: flex; align-items: center; justify-content: center; color: #26333d; font-weight: 700; }
</style>
</head>
<body>

<div id="loading">読み込み中…</div>
<div id="canvasWrap"></div>

<div id="ui-layer">
  <div class="hud">
    <div class="hud-box"><div class="hud-label">Score</div><div class="hud-value" id="scoreVal">0</div></div>
    <div class="hud-box" id="timer-box"><div class="hud-label">Time</div><div class="hud-value" id="timeVal">90</div></div>
  </div>
  <div class="leaderboard" id="lb"></div>
</div>
<div id="joystick"><div class="base"></div><div class="knob"></div></div>

<div id="overlay">
  <div class="panel" id="screen-content">
    <p class="title">CITY HOLE <span>3D</span></p>
    <p class="desc">最初のThree.js版です。ドラッグして移動し、街を飲み込め。</p>
    <button id="startBtn">START</button>
  </div>
</div>

<script type="importmap">
{ "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js" } }
</script>
<script type="module">
import * as THREE from 'three';

const CONFIG = {
  WORLD_SIZE: 2400,
  NUM_BOTS: 2,
  OBJECT_COUNT: 140,
  INIT_R: 15,
  MAX_R: 320,
  GAME_TIME: 120,
  SPEED_MIN: 90,
  SPEED_MAX: 260,
  GROWTH_MULT: 30,
};

// ---------------- three.js 基本セットアップ ----------------
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xbfe3f2);
scene.fog = new THREE.Fog(0xbfe3f2, 700, 2200);

const camera = new THREE.PerspectiveCamera(52, innerWidth / innerHeight, 1, 4000);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.setSize(innerWidth, innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.getElementById('canvasWrap').appendChild(renderer.domElement);

window.addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});

const hemi = new THREE.HemisphereLight(0xffffff, 0x6b8e4e, 0.85);
scene.add(hemi);
const sun = new THREE.DirectionalLight(0xfff4d6, 1.15);
sun.position.set(-500, 700, -350);
sun.castShadow = true;
sun.shadow.mapSize.set(2048, 2048);
sun.shadow.camera.left = -700; sun.shadow.camera.right = 700;
sun.shadow.camera.top = 700; sun.shadow.camera.bottom = -700;
sun.shadow.camera.far = 2000;
scene.add(sun);
scene.add(sun.target);

const groundMat = new THREE.MeshStandardMaterial({ color: 0x7fae52 });
const ground = new THREE.Mesh(new THREE.PlaneGeometry(CONFIG.WORLD_SIZE, CONFIG.WORLD_SIZE), groundMat);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);

// マップの端をハザードストライプで示す
const edgeMat = new THREE.MeshBasicMaterial({ color: 0x1a1a1a });
const edgeGeo = new THREE.BoxGeometry(CONFIG.WORLD_SIZE + 20, 8, 6);
const half = CONFIG.WORLD_SIZE / 2;
[[0, -half - 3], [0, half + 3]].forEach(([x, z]) => {
  const e1 = new THREE.Mesh(edgeGeo, edgeMat); e1.position.set(x, 4, z); scene.add(e1);
});
const edgeGeoV = new THREE.BoxGeometry(6, 8, CONFIG.WORLD_SIZE + 20);
[[-half - 3, 0], [half + 3, 0]].forEach(([x, z]) => {
  const e1 = new THREE.Mesh(edgeGeoV, edgeMat); e1.position.set(x, 4, z); scene.add(e1);
});

// ---------------- 穴(プレイヤー / CPU) ----------------
const holeCircleGeo = new THREE.CircleGeometry(1, 40);
const BOT_COLORS = [0xff9f43, 0xee5253, 0x5f27cd];
const BOT_NAMES = ['CPU 1', 'CPU 2', 'CPU 3'];

function makeEntity(isPlayer, idx) {
  const color = isPlayer ? 0x00d2d3 : BOT_COLORS[idx % BOT_COLORS.length];
  const mat = new THREE.MeshBasicMaterial({ color: 0x030303 });
  const mesh = new THREE.Mesh(holeCircleGeo, mat);
  mesh.rotation.x = -Math.PI / 2;
  mesh.position.y = 0.08;
  scene.add(mesh);
  const ringMat = new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.6, side: THREE.DoubleSide });
  const ring = new THREE.Mesh(new THREE.RingGeometry(0.98, 1.05, 40), ringMat);
  ring.rotation.x = -Math.PI / 2;
  ring.position.y = 0.09;
  scene.add(ring);
  return {
    isPlayer, name: isPlayer ? 'YOU' : BOT_NAMES[idx % BOT_NAMES.length], colorHex: color,
    mesh, ring, position: new THREE.Vector3(0, 0, 0), r: CONFIG.INIT_R,
    vx: 0, vz: 0, score: 0, wanderAngle: Math.random() * Math.PI * 2, wanderT: 0, target: null
  };
}

let entities = [];
function player() { return entities[0]; }

function speedFor(r) {
  const t = Math.max(0, Math.min(1, (r - CONFIG.INIT_R) / (CONFIG.MAX_R - CONFIG.INIT_R)));
  return CONFIG.SPEED_MIN + Math.pow(t, 0.55) * (CONFIG.SPEED_MAX - CONFIG.SPEED_MIN);
}
function updateEntityMesh(e) {
  e.mesh.position.set(e.position.x, 0.08, e.position.z);
  e.mesh.scale.set(e.r, e.r, 1);
  e.ring.position.set(e.position.x, 0.09, e.position.z);
  e.ring.scale.set(e.r, e.r, 1);
}

// ---------------- 建物などの型(footprintは半径ベースの矩形/円で近似) ----------------
function shadeHex(hex, amt) {
  const r = (hex >> 16) & 0xff, g = (hex >> 8) & 0xff, b = hex & 0xff;
  const c = v => Math.max(0, Math.min(255, Math.round(v + 255 * amt)));
  return (c(r) << 16) | (c(g) << 8) | c(b);
}
function stdMat(color) { return new THREE.MeshStandardMaterial({ color }); }

const TYPES = [
  { id: 'cone', hw: 6, hd: 6, round: false, pts: 3, build: buildCone },
  { id: 'bench', hw: 14, hd: 7, round: false, pts: 8, build: buildBench },
  { id: 'tree', hw: 18, hd: 18, round: true, pts: 16, build: buildTree },
  { id: 'car', hw: 22, hd: 12, round: false, pts: 24, build: buildCar },
  { id: 'silo', hw: 34, hd: 34, round: true, pts: 40, build: buildSilo },
  { id: 'house', hw: 46, hd: 34, round: false, pts: 70, build: buildHouse },
];

function buildCone(t) {
  const g = new THREE.Group();
  const m = new THREE.Mesh(new THREE.ConeGeometry(t.hw, 22, 16), stdMat(0xff8a3d));
  m.position.y = 11; m.castShadow = true; m.receiveShadow = true;
  g.add(m);
  return g;
}
function buildBench(t) {
  const g = new THREE.Group();
  const seat = new THREE.Mesh(new THREE.BoxGeometry(t.hw*2, 4, t.hd*2), stdMat(0x9b59b6));
  seat.position.y = 10; seat.castShadow = true; seat.receiveShadow = true;
  g.add(seat);
  const back = new THREE.Mesh(new THREE.BoxGeometry(t.hw*2, 14, 3), stdMat(0x8a4aa0));
  back.position.set(0, 18, -t.hd*0.85); back.castShadow = true;
  g.add(back);
  return g;
}
function buildTree(t) {
  const g = new THREE.Group();
  const trunk = new THREE.Mesh(new THREE.CylinderGeometry(4, 5, 20, 8), stdMat(0x7a5636));
  trunk.position.y = 10; trunk.castShadow = true;
  g.add(trunk);
  const canopy = new THREE.Mesh(new THREE.SphereGeometry(t.hw*0.85, 10, 8), stdMat(0x3fae5c));
  canopy.position.y = 32; canopy.castShadow = true;
  g.add(canopy);
  return g;
}
function buildCar(t) {
  const g = new THREE.Group();
  const body = new THREE.Mesh(new THREE.BoxGeometry(t.hw*2, 12, t.hd*2), stdMat(0x3498db));
  body.position.y = 8; body.castShadow = true; body.receiveShadow = true;
  g.add(body);
  const cabin = new THREE.Mesh(new THREE.BoxGeometry(t.hw*1.1, 10, t.hd*1.3), stdMat(0xd8ecf5));
  cabin.position.y = 18; cabin.castShadow = true;
  g.add(cabin);
  return g;
}
function buildSilo(t) {
  const g = new THREE.Group();
  const body = new THREE.Mesh(new THREE.CylinderGeometry(t.hw, t.hw, 46, 20), stdMat(0x8395a7));
  body.position.y = 23; body.castShadow = true; body.receiveShadow = true;
  g.add(body);
  const cap = new THREE.Mesh(new THREE.ConeGeometry(t.hw*1.02, 12, 20), stdMat(shadeHex(0x8395a7, 0.15)));
  cap.position.y = 46 + 6; cap.castShadow = true;
  g.add(cap);
  return g;
}
function buildHouse(t) {
  const g = new THREE.Group();
  const wallH = 34;
  const wall = new THREE.Mesh(new THREE.BoxGeometry(t.hw*2, wallH, t.hd*2), stdMat(0xc9814c));
  wall.position.y = wallH/2; wall.castShadow = true; wall.receiveShadow = true;
  g.add(wall);
  const roof = new THREE.Mesh(new THREE.ConeGeometry(t.hw*1.3, wallH*0.6, 4), stdMat(0x8a4a3a));
  roof.rotation.y = Math.PI/4; roof.position.y = wallH + wallH*0.3; roof.castShadow = true;
  g.add(roof);
  const winMat = new THREE.MeshStandardMaterial({ color: 0x9fd0ff, emissive: 0x224466, emissiveIntensity: 0.25 });
  const w1 = new THREE.Mesh(new THREE.BoxGeometry(t.hw*0.55, wallH*0.32, 2), winMat);
  w1.position.set(-t.hw*0.5, wallH*0.58, t.hd + 1);
  g.add(w1);
  const w2 = w1.clone(); w2.position.x = t.hw*0.5; g.add(w2);
  const door = new THREE.Mesh(new THREE.BoxGeometry(t.hw*0.42, wallH*0.55, 2), stdMat(0x5c3a26));
  door.position.set(0, wallH*0.28, t.hd + 1);
  g.add(door);
  const chimney = new THREE.Mesh(new THREE.BoxGeometry(t.hw*0.3, wallH*0.55, t.hw*0.3), stdMat(0x7a6a5a));
  chimney.position.set(t.hw*0.6, wallH + wallH*0.28, -t.hd*0.3);
  chimney.castShadow = true;
  g.add(chimney);
  return g;
}

// ---------------- オブジェクト管理 ----------------
let objects = [];
function pickType() { return TYPES[Math.floor(Math.random() * TYPES.length)]; }

function spawnObjectAt(x, z) {
  const t = pickType();
  const model = t.build(t);
  const pivot = new THREE.Group();
  model.position.set(x, 0, z);
  pivot.add(model);
  scene.add(pivot);
  objects.push({
    type: t, x, z, model, pivot,
    falling: false, fallT: 0, eater: null, topple: false, fallAxis: null
  });
}
function initObjects() {
  for (const o of objects) { scene.remove(o.pivot); }
  objects = [];
  for (let i = 0; i < CONFIG.OBJECT_COUNT; i++) {
    const x = (Math.random() - 0.5) * (CONFIG.WORLD_SIZE - 100);
    const z = (Math.random() - 0.5) * (CONFIG.WORLD_SIZE - 100);
    spawnObjectAt(x, z);
  }
}
function respawnObject(old) {
  scene.remove(old.pivot);
  const x = (Math.random() - 0.5) * (CONFIG.WORLD_SIZE - 100);
  const z = (Math.random() - 0.5) * (CONFIG.WORLD_SIZE - 100);
  spawnObjectAt(x, z);
}

function addArea(e, area) {
  const cur = Math.PI * e.r * e.r;
  e.r = Math.min(CONFIG.MAX_R, Math.sqrt((cur + area) / Math.PI));
}

function triggerFall(o, e) {
  const halfW = o.type.hw, halfD = o.type.hd;
  let pivotPos;
  if (o.type.round) {
    pivotPos = new THREE.Vector3(o.x, 0, o.z);
    o.topple = false;
  } else {
    const corners = [[-halfW,-halfD],[halfW,-halfD],[halfW,halfD],[-halfW,halfD]];
    const outside = [];
    for (const [lx, lz] of corners) {
      const wx = o.x + lx, wz = o.z + lz;
      const d = Math.hypot(wx - e.position.x, wz - e.position.z);
      if (d >= e.r) outside.push({ x: wx, z: wz });
    }
    if (outside.length > 0) {
      let px = 0, pz = 0;
      for (const c of outside) { px += c.x; pz += c.z; }
      px /= outside.length; pz /= outside.length;
      pivotPos = new THREE.Vector3(px, 0, pz);
      o.topple = true;
    } else {
      pivotPos = new THREE.Vector3(o.x, 0, o.z);
      o.topple = false;
    }
  }

  o.pivot.position.copy(pivotPos);
  o.model.position.set(o.x - pivotPos.x, 0, o.z - pivotPos.z);

  if (o.topple) {
    const dirX = e.position.x - pivotPos.x, dirZ = e.position.z - pivotPos.z;
    const dlen = Math.hypot(dirX, dirZ) || 1;
    o.fallAxis = new THREE.Vector3(-dirZ / dlen, 0, dirX / dlen);
  }

  o.falling = true; o.fallT = 0; o.eater = e;
}

function updateFalling(dt) {
  for (let i = objects.length - 1; i >= 0; i--) {
    const o = objects[i];
    if (!o.falling) continue;
    o.fallT += dt / 0.9;
    const t = Math.min(o.fallT, 1);
    if (o.topple) {
      o.pivot.quaternion.setFromAxisAngle(o.fallAxis, t * Math.PI * 0.58);
    } else {
      o.model.scale.setScalar(1 - t);
    }
    o.pivot.position.y = -t * 60;
    const fadeStart = 0.6;
    if (t > fadeStart) {
      const alpha = 1 - (t - fadeStart) / (1 - fadeStart);
      o.model.traverse(m => {
        if (m.material) { m.material.transparent = true; m.material.opacity = Math.max(0, alpha); }
      });
    }
    if (o.fallT >= 1) {
      o.eater.score += o.type.pts;
      addArea(o.eater, o.type.pts * CONFIG.GROWTH_MULT);
      if (o.eater.isPlayer) updateScoreUI();
      respawnObject(o);
      objects.splice(i, 1);
    }
  }
}

// ---------------- 入力(ジョイスティック) ----------------
let joy = { active: false, ox: 0, oy: 0, dx: 0, dy: 0, mag: 0 };
const JOY_MAX = 46;
const joystickEl = document.getElementById('joystick');
function joyStart(x, y) {
  joy.active = true; joy.ox = x; joy.oy = y; joy.dx = 0; joy.dy = 0; joy.mag = 0;
  joystickEl.style.left = x + 'px'; joystickEl.style.top = y + 'px';
  joystickEl.style.display = 'block';
  joystickEl.querySelector('.knob').style.left = '29px';
  joystickEl.querySelector('.knob').style.top = '29px';
}
function joyMove(x, y) {
  if (!joy.active) return;
  const dx = x - joy.ox, dy = y - joy.oy;
  const d = Math.hypot(dx, dy);
  const clamped = Math.min(d, JOY_MAX);
  const nx = d > 0 ? dx / d : 0, ny = d > 0 ? dy / d : 0;
  joy.dx = nx; joy.dy = ny; joy.mag = clamped / JOY_MAX;
  joystickEl.querySelector('.knob').style.left = (29 + nx * clamped) + 'px';
  joystickEl.querySelector('.knob').style.top = (29 + ny * clamped) + 'px';
}
function joyEnd() { joy.active = false; joy.mag = 0; joystickEl.style.display = 'none'; }

renderer.domElement.addEventListener('pointerdown', e => { e.preventDefault(); joyStart(e.clientX, e.clientY); });
renderer.domElement.addEventListener('pointermove', e => { if (joy.active) { e.preventDefault(); joyMove(e.clientX, e.clientY); } });
renderer.domElement.addEventListener('pointerup', e => { e.preventDefault(); joyEnd(); });
renderer.domElement.addEventListener('pointercancel', () => joyEnd());
renderer.domElement.addEventListener('touchstart', e => e.preventDefault(), { passive: false });
renderer.domElement.addEventListener('touchmove', e => e.preventDefault(), { passive: false });

// ---------------- ゲーム進行 ----------------
let running = false, timeLeft = CONFIG.GAME_TIME, timerAccum = 0, score = 0;
const scoreVal = document.getElementById('scoreVal');
const timeVal = document.getElementById('timeVal');
const timerBox = document.getElementById('timer-box');
const lbEl = document.getElementById('lb');

function updateScoreUI() { score = player().score; scoreVal.textContent = score; }

function resetGame() {
  entities.forEach(e => { scene.remove(e.mesh); scene.remove(e.ring); });
  entities = [];
  entities.push(makeEntity(true, 0));
  for (let i = 0; i < CONFIG.NUM_BOTS; i++) entities.push(makeEntity(false, i));
  entities.forEach(e => { e.position.set((Math.random()-0.5)*400, 0, (Math.random()-0.5)*400); e.r = CONFIG.INIT_R; e.score = 0; updateEntityMesh(e); });
  player().position.set(0,0,0);
  initObjects();
  score = 0; timeLeft = CONFIG.GAME_TIME; timerAccum = 0; running = true;
  scoreVal.textContent = '0'; timeVal.textContent = String(CONFIG.GAME_TIME);
  timerBox.classList.remove('danger');
  document.getElementById('overlay').classList.add('hidden');
}

function botAI(bot, dt) {
  let target = null, flee = false, bestD = Infinity;
  for (const other of entities) {
    if (other === bot) continue;
    const d = Math.hypot(other.position.x - bot.position.x, other.position.z - bot.position.z);
    if (other.r > bot.r * 1.2 && d < 260) { target = { x: bot.position.x - (other.position.x-bot.position.x), z: bot.position.z - (other.position.z-bot.position.z) }; flee = true; break; }
  }
  if (!flee) {
    if (!bot.target || bot.target.falling) {
      let best = null, bd = Infinity;
      for (const o of objects) {
        if (o.falling) continue;
        const need = o.type.round ? bot.r*0.9 : bot.r*0.9;
        if (bot.r < o.type.hw*0.7 && bot.r < o.type.hd*0.7) continue;
        const d = Math.hypot(o.x-bot.position.x, o.z-bot.position.z);
        if (d < bd) { bd = d; best = o; }
      }
      bot.target = best;
    }
    if (bot.target) target = { x: bot.target.x, z: bot.target.z };
  }
  const speed = speedFor(bot.r);
  if (target) {
    const dx = target.x - bot.position.x, dz = target.z - bot.position.z;
    const d = Math.hypot(dx,dz) || 1;
    bot.vx = dx/d*speed; bot.vz = dz/d*speed;
  } else {
    bot.wanderT -= dt;
    if (bot.wanderT <= 0) { bot.wanderAngle = Math.random()*Math.PI*2; bot.wanderT = 1.5+Math.random()*2; }
    bot.vx = Math.cos(bot.wanderAngle)*speed*0.6; bot.vz = Math.sin(bot.wanderAngle)*speed*0.6;
  }
  bot.position.x = Math.max(-CONFIG.WORLD_SIZE/2+bot.r, Math.min(CONFIG.WORLD_SIZE/2-bot.r, bot.position.x + bot.vx*dt));
  bot.position.z = Math.max(-CONFIG.WORLD_SIZE/2+bot.r, Math.min(CONFIG.WORLD_SIZE/2-bot.r, bot.position.z + bot.vz*dt));
}

function updateEntities(dt) {
  const p = player();
  const speed = speedFor(p.r);
  if (joy.active && joy.mag > 0.05) {
    p.vx = joy.dx * speed * joy.mag;
    p.vz = joy.dy * speed * joy.mag;
  } else { p.vx *= 0.85; p.vz *= 0.85; }
  p.position.x = Math.max(-CONFIG.WORLD_SIZE/2+p.r, Math.min(CONFIG.WORLD_SIZE/2-p.r, p.position.x + p.vx*dt));
  p.position.z = Math.max(-CONFIG.WORLD_SIZE/2+p.r, Math.min(CONFIG.WORLD_SIZE/2-p.r, p.position.z + p.vz*dt));

  for (let i = 1; i < entities.length; i++) botAI(entities[i], dt);
  entities.forEach(updateEntityMesh);
}

function checkEating() {
  for (const e of entities) {
    for (const o of objects) {
      if (o.falling) continue;
      if (o.type.round) {
        const d = Math.hypot(o.x - e.position.x, o.z - e.position.z);
        if (d < e.r * 0.72 && e.r > o.type.hw * 0.75) triggerFall(o, e);
      } else {
        if (e.r < o.type.hw * 0.55 && e.r < o.type.hd * 0.55) continue;
        const corners = [[-o.type.hw,-o.type.hd],[o.type.hw,-o.type.hd],[o.type.hw,o.type.hd],[-o.type.hw,o.type.hd]];
        let anyInside = false;
        for (const [lx, lz] of corners) {
          const d = Math.hypot(o.x+lx-e.position.x, o.z+lz-e.position.z);
          if (d < e.r) { anyInside = true; break; }
        }
        if (anyInside) triggerFall(o, e);
      }
    }
  }
}

function updateBoard() {
  const sorted = [...entities].sort((a,b)=>b.score-a.score);
  lbEl.innerHTML = sorted.map(e => `<div class="lb-row ${e.isPlayer?'me':''}"><span class="lb-dot" style="background:#${e.colorHex.toString(16).padStart(6,'0')}"></span><span class="lb-name">${e.isPlayer?'YOU':e.name}</span><span class="lb-score">${e.score}</span></div>`).join('');
}

function endGame() {
  running = false;
  const sorted = [...entities].sort((a,b)=>b.score-a.score);
  const rank = sorted.findIndex(e=>e.isPlayer)+1;
  const panel = document.getElementById('screen-content');
  panel.innerHTML = `<p class="title">TIME UP!</p><div class="result-rank">RANK: ${rank} / ${entities.length}</div><div class="result-score">FINAL SCORE: ${player().score}</div><button id="retryBtn">PLAY AGAIN</button>`;
  document.getElementById('overlay').classList.remove('hidden');
  document.getElementById('retryBtn').addEventListener('click', resetGame);
}

document.getElementById('startBtn').addEventListener('click', resetGame);

// ---------------- カメラ追従 ----------------
function updateCamera() {
  const p = player();
  const dist = 260 + p.r * 2.2;
  const height = 220 + p.r * 1.7;
  camera.position.set(p.position.x - dist*0.35, height, p.position.z + dist*0.9);
  camera.lookAt(p.position.x, 10, p.position.z);
}

// ---------------- メインループ ----------------
let lastTime = performance.now();
function loop(now) {
  const dt = Math.min((now - lastTime) / 1000, 0.05);
  lastTime = now;
  if (running) {
    updateEntities(dt);
    checkEating();
    updateFalling(dt);
    updateBoard();
    timerAccum += dt;
    if (timerAccum >= 1) {
      timerAccum -= 1; timeLeft--;
      timeVal.textContent = String(Math.max(0, timeLeft));
      if (timeLeft <= 10) timerBox.classList.add('danger');
      if (timeLeft <= 0) endGame();
    }
  }
  updateCamera();
  renderer.render(scene, camera);
  requestAnimationFrame(loop);
}

resetGame();
running = false;
document.getElementById('loading').style.display = 'none';
requestAnimationFrame(loop);
</script>
</body>
</html>
