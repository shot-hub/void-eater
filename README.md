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
  GAME_TIME: 180,
  SPEED_MIN: 90,
  SPEED_MAX: 260,
  GROWTH_MULT: 30,
};

// ---------------- three.js 基本セットアップ ----------------
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xbfe3f2);
scene.fog = new THREE.Fog(0xbfe3f2, 700, 2200);

const camera = new THREE.PerspectiveCamera(52, innerWidth / innerHeight, 1, 4000);
let fadedMeshes = new Set();
function updateOcclusionFade() {
  const p = player();
  const holeWorldPos = new THREE.Vector3(p.position.x, p.r * 0.3, p.position.z);
  const holeDistToCam = camera.position.distanceTo(holeWorldPos);
  const holeScreen = holeWorldPos.clone().project(camera);
  const holeScreenR = (p.r / holeDistToCam) * 1.15;

  const hitSet = new Set();
  for (const o of objects) {
    if (o.falling) continue;
    const worldDist = Math.hypot(o.x - p.position.x, o.z - p.position.z);
    if (worldDist > 550) continue;

    const objWorldPos = new THREE.Vector3(o.x, o.type.height * 0.5, o.z);
    const objDistToCam = camera.position.distanceTo(objWorldPos);
    if (objDistToCam > holeDistToCam - 4) continue; // 穴より手前にある物だけを対象にする

    const objScreen = objWorldPos.clone().project(camera);
    if (objScreen.z > 1 || objScreen.z < -1) continue; // 画面の外(カメラの裏側等)

    const dxS = objScreen.x - holeScreen.x, dyS = objScreen.y - holeScreen.y;
    const screenDist = Math.hypot(dxS, dyS);
    const objScreenR = (Math.max(o.type.hw, o.type.hd) / objDistToCam) * 1.3;

    if (screenDist < holeScreenR + objScreenR) {
      o.model.traverse(m => { if (m.isMesh) hitSet.add(m); });
    }
  }

  for (const m of hitSet) {
    if (!fadedMeshes.has(m)) {
      m.userData._origOpacity = m.material.opacity;
      m.userData._origTransparent = m.material.transparent;
      m.material.transparent = true;
    }
    m.material.opacity = 0.32;
  }
  for (const m of fadedMeshes) {
    if (!hitSet.has(m)) {
      m.material.opacity = m.userData._origOpacity !== undefined ? m.userData._origOpacity : 1;
      m.material.transparent = m.userData._origTransparent || false;
    }
  }
  fadedMeshes = hitSet;
}

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
const MAX_HOLES_SHADER = 4;
groundMat.onBeforeCompile = (shader) => {
  shader.uniforms.holePos = { value: Array.from({length: MAX_HOLES_SHADER}, () => new THREE.Vector2(99999, 99999)) };
  shader.uniforms.holeR = { value: new Array(MAX_HOLES_SHADER).fill(0) };
  shader.vertexShader = 'varying vec3 vWorldPos;\n' + shader.vertexShader.replace(
    '#include <begin_vertex>',
    `#include <begin_vertex>\n    vWorldPos = (modelMatrix * vec4(position, 1.0)).xyz;`
  );
  shader.fragmentShader = 'varying vec3 vWorldPos;\nuniform vec2 holePos[' + MAX_HOLES_SHADER + '];\nuniform float holeR[' + MAX_HOLES_SHADER + '];\n' + shader.fragmentShader.replace(
    '#include <clipping_planes_fragment>',
    `#include <clipping_planes_fragment>
    for (int hi = 0; hi < ${MAX_HOLES_SHADER}; hi++) {
      if (distance(vWorldPos.xz, holePos[hi]) < holeR[hi]) discard;
    }`
  );
  groundMat.userData.shader = shader;
};
groundMat.customProgramCacheKey = () => 'groundHoleCutout';
const ground = new THREE.Mesh(new THREE.PlaneGeometry(CONFIG.WORLD_SIZE, CONFIG.WORLD_SIZE, 1, 1), groundMat);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);
function updateGroundHoles() {
  const sh = groundMat.userData.shader;
  if (!sh) return;
  for (let i = 0; i < MAX_HOLES_SHADER; i++) {
    if (entities[i]) {
      sh.uniforms.holePos.value[i].set(entities[i].position.x, entities[i].position.z);
      sh.uniforms.holeR.value[i] = entities[i].r * 0.98;
    } else {
      sh.uniforms.holeR.value[i] = 0;
    }
  }
}

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
const pitGeo = new THREE.CylinderGeometry(1, 0.72, 1, 40, 1, true);
const pitFloorGeo = new THREE.CircleGeometry(0.74, 40);
const BOT_COLORS = [0xff9f43, 0xee5253, 0x5f27cd];
const BOT_NAMES = ['CPU 1', 'CPU 2', 'CPU 3'];

function pitDepthFor(r) { return 40 + r * 0.6; }

function makeEntity(isPlayer, idx) {
  const color = isPlayer ? 0x00d2d3 : BOT_COLORS[idx % BOT_COLORS.length];
  // 穴の内側の壁(見下ろした時に「潜っていく空間」が実際に見える筒状の穴)
  const pitMat = new THREE.MeshStandardMaterial({ color: 0x050505, side: THREE.BackSide, roughness: 1 });
  const pit = new THREE.Mesh(pitGeo, pitMat);
  scene.add(pit);
  const floorMat = new THREE.MeshStandardMaterial({ color: 0x020202 });
  const floor = new THREE.Mesh(pitFloorGeo, floorMat);
  floor.rotation.x = -Math.PI / 2;
  scene.add(floor);
  const ringMat = new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.6, side: THREE.DoubleSide });
  const ring = new THREE.Mesh(new THREE.RingGeometry(0.98, 1.05, 40), ringMat);
  ring.rotation.x = -Math.PI / 2;
  ring.position.y = 0.09;
  scene.add(ring);
  return {
    isPlayer, name: isPlayer ? 'YOU' : BOT_NAMES[idx % BOT_NAMES.length], colorHex: color,
    pit, floor, ring, position: new THREE.Vector3(0, 0, 0), r: CONFIG.INIT_R,
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
  const depth = pitDepthFor(e.r);
  e.pit.position.set(e.position.x, -depth / 2, e.position.z);
  e.pit.scale.set(e.r, depth, e.r);
  e.floor.position.set(e.position.x, -depth + 0.05, e.position.z);
  e.floor.scale.set(e.r * 0.74, e.r * 0.74, 1);
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
  { id: 'hydrant', hw: 4, hd: 4, height: 9, round: true, pts: 1, build: buildHydrant },
  { id: 'mailbox', hw: 5, hd: 4, height: 13, round: false, pts: 2, build: buildMailbox },
  { id: 'trashcan', hw: 5, hd: 5, height: 11, round: true, pts: 2, build: buildTrashcan },
  { id: 'lamppost', hw: 3, hd: 3, height: 32, round: true, pts: 3, build: buildLamppost },
  { id: 'cone', hw: 6, hd: 6, height: 22, round: false, pts: 3, build: buildCone },
  { id: 'bench', hw: 14, hd: 7, height: 24, round: false, pts: 8, build: buildBench },
  { id: 'tree', hw: 18, hd: 18, height: 42, round: true, pts: 16, build: buildTree },
  { id: 'car', hw: 22, hd: 12, height: 24, round: false, pts: 24, build: buildCar },
  { id: 'kiosk', hw: 28, hd: 22, height: 34, round: false, pts: 34, build: buildKiosk },
  { id: 'silo', hw: 34, hd: 34, height: 58, round: true, pts: 46, build: buildSilo },
  { id: 'house', hw: 46, hd: 34, height: 54, round: false, pts: 68, build: buildHouse },
  { id: 'tank', hw: 60, hd: 60, height: 70, round: true, pts: 100, build: buildTank },
  { id: 'office', hw: 80, hd: 60, height: 105, round: false, pts: 155, build: buildOffice },
  { id: 'towerRound', hw: 105, hd: 105, height: 175, round: true, pts: 225, build: buildTowerRound },
  { id: 'towerBox', hw: 135, hd: 105, height: 205, round: false, pts: 300, build: buildTowerBox },
  { id: 'landmark', hw: 190, hd: 155, height: 255, round: false, pts: 430, build: buildLandmark },
];

function buildKiosk(t) {
  const g = new THREE.Group();
  const wall = new THREE.Mesh(new THREE.BoxGeometry(t.hw*2, 22, t.hd*2), stdMat(0x8b6ad6));
  wall.position.y = 11; wall.castShadow = true; wall.receiveShadow = true;
  g.add(wall);
  const roof = new THREE.Mesh(new THREE.ConeGeometry(t.hw*1.2, 12, 4), stdMat(0xac8148));
  roof.rotation.y = Math.PI/4; roof.position.y = 22+6; roof.castShadow = true;
  g.add(roof);
  return g;
}
function buildTank(t) {
  const g = new THREE.Group();
  const body = new THREE.Mesh(new THREE.CylinderGeometry(t.hw, t.hw, 62, 20), stdMat(0x9aa7b5));
  body.position.y = 31; body.castShadow = true; body.receiveShadow = true;
  g.add(body);
  const dome = new THREE.Mesh(new THREE.SphereGeometry(t.hw, 20, 12, 0, Math.PI*2, 0, Math.PI/2), stdMat(shadeHex(0x9aa7b5, 0.15)));
  dome.position.y = 62; dome.castShadow = true;
  g.add(dome);
  return g;
}
function buildOffice(t) {
  const g = new THREE.Group();
  const wallH = 90;
  const wall = new THREE.Mesh(new THREE.BoxGeometry(t.hw*2, wallH, t.hd*2), stdMat(0x767c8f));
  wall.position.y = wallH/2; wall.castShadow = true; wall.receiveShadow = true;
  g.add(wall);
  const winMat = new THREE.MeshStandardMaterial({ color: 0xffe08c, emissive: 0x554422, emissiveIntensity: 0.3 });
  for (let row=0; row<4; row++) {
    for (let col=-1; col<=1; col++) {
      const w = new THREE.Mesh(new THREE.BoxGeometry(t.hw*0.32, wallH*0.12, 2), winMat);
      w.position.set(col*t.hw*0.6, 12+row*18, t.hd+1);
      g.add(w);
    }
  }
  const cap = new THREE.Mesh(new THREE.BoxGeometry(t.hw*2.05, 4, t.hd*2.05), stdMat(shadeHex(0x767c8f,0.2)));
  cap.position.y = wallH+2; cap.castShadow = true;
  g.add(cap);
  return g;
}
function buildTowerRound(t) {
  const g = new THREE.Group();
  const h = 160;
  const body = new THREE.Mesh(new THREE.CylinderGeometry(t.hw*0.8, t.hw, h, 20), stdMat(0x4b6584));
  body.position.y = h/2; body.castShadow = true; body.receiveShadow = true;
  g.add(body);
  const antenna = new THREE.Mesh(new THREE.CylinderGeometry(2,2,20,8), stdMat(0x9096a6));
  antenna.position.y = h+10; antenna.castShadow = true;
  g.add(antenna);
  return g;
}
function buildTowerBox(t) {
  const g = new THREE.Group();
  const h = 190;
  const wall = new THREE.Mesh(new THREE.BoxGeometry(t.hw*2, h, t.hd*2), stdMat(0x2f3542));
  wall.position.y = h/2; wall.castShadow = true; wall.receiveShadow = true;
  g.add(wall);
  const wing = new THREE.Mesh(new THREE.BoxGeometry(t.hw*1.1, h*0.6, t.hd*1.1), stdMat(shadeHex(0x2f3542,-0.1)));
  wing.position.set(t.hw*0.9, h*0.3, -t.hd*0.4); wing.castShadow = true;
  g.add(wing);
  const spire = new THREE.Mesh(new THREE.CylinderGeometry(2,2,30,8), stdMat(0xff5555));
  spire.position.y = h+15; spire.castShadow = true;
  g.add(spire);
  return g;
}
function buildLandmark(t) {
  const g = new THREE.Group();
  const baseH = 90;
  const base = new THREE.Mesh(new THREE.CylinderGeometry(t.hw, t.hw*1.05, baseH, 24), stdMat(0xc0975a));
  base.position.y = baseH/2; base.castShadow = true; base.receiveShadow = true;
  g.add(base);
  const towerH = 140;
  const tower = new THREE.Mesh(new THREE.BoxGeometry(t.hw*1.1, towerH, t.hd*1.1), stdMat(shadeHex(0xc0975a,0.12)));
  tower.position.y = baseH + towerH/2; tower.castShadow = true;
  g.add(tower);
  const spireTip = new THREE.Mesh(new THREE.ConeGeometry(6, 25, 8), stdMat(0xffd24d));
  spireTip.position.y = baseH+towerH+12; spireTip.castShadow = true;
  g.add(spireTip);
  return g;
}

function buildHydrant(t) {
  const g = new THREE.Group();
  const body = new THREE.Mesh(new THREE.CylinderGeometry(t.hw*0.7, t.hw*0.8, 8, 10), stdMat(0xd7362b));
  body.position.y = 4; body.castShadow = true; body.receiveShadow = true;
  g.add(body);
  const cap = new THREE.Mesh(new THREE.SphereGeometry(t.hw*0.55, 10, 8), stdMat(0xb52a20));
  cap.position.y = 9; cap.castShadow = true;
  g.add(cap);
  return g;
}
function buildMailbox(t) {
  const g = new THREE.Group();
  const post = new THREE.Mesh(new THREE.BoxGeometry(1.4, 8, 1.4), stdMat(0x555555));
  post.position.y = 4; post.castShadow = true;
  g.add(post);
  const box = new THREE.Mesh(new THREE.BoxGeometry(t.hw*1.8, 5, t.hd*1.8), stdMat(0x2e5fa3));
  box.position.y = 10.5; box.castShadow = true; box.receiveShadow = true;
  g.add(box);
  return g;
}
function buildTrashcan(t) {
  const g = new THREE.Group();
  const body = new THREE.Mesh(new THREE.CylinderGeometry(t.hw, t.hw*0.85, 11, 12), stdMat(0x4a5a4a));
  body.position.y = 5.5; body.castShadow = true; body.receiveShadow = true;
  g.add(body);
  const lid = new THREE.Mesh(new THREE.CylinderGeometry(t.hw*1.05, t.hw*1.05, 1.5, 12), stdMat(0x3a463a));
  lid.position.y = 11.5; lid.castShadow = true;
  g.add(lid);
  return g;
}
function buildLamppost(t) {
  const g = new THREE.Group();
  const pole = new THREE.Mesh(new THREE.CylinderGeometry(t.hw*0.4, t.hw*0.5, 28, 8), stdMat(0x333640));
  pole.position.y = 14; pole.castShadow = true;
  g.add(pole);
  const lampMat = new THREE.MeshStandardMaterial({ color: 0xfff2b0, emissive: 0xffcc55, emissiveIntensity: 0.6 });
  const lamp = new THREE.Mesh(new THREE.SphereGeometry(t.hw*1.1, 10, 8), lampMat);
  lamp.position.y = 29; lamp.castShadow = true;
  g.add(lamp);
  return g;
}
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

// 簡易空間ハッシュ:大きい建物(半径190)にも対応できるバケット幅
const BUCKET = 420;
let spatialBuckets = new Map();
function bucketKeyOf(x, z) { return Math.floor(x / BUCKET) + ',' + Math.floor(z / BUCKET); }
function addToBuckets(o) {
  const key = bucketKeyOf(o.x, o.z);
  if (!spatialBuckets.has(key)) spatialBuckets.set(key, new Set());
  spatialBuckets.get(key).add(o);
  o._bucketKey = key;
}
function removeFromBuckets(o) {
  const set = spatialBuckets.get(o._bucketKey);
  if (set) set.delete(o);
}
function overlapsExisting(t, x, z) {
  const bx = Math.floor(x / BUCKET), bz = Math.floor(z / BUCKET);
  const rr = Math.max(t.hw, t.hd);
  for (let dx = -1; dx <= 1; dx++) {
    for (let dz = -1; dz <= 1; dz++) {
      const set = spatialBuckets.get((bx+dx) + ',' + (bz+dz));
      if (!set) continue;
      for (const other of set) {
        const d = Math.hypot(other.x - x, other.z - z);
        const otherR = Math.max(other.type.hw, other.type.hd);
        if (d < (rr + otherR) * 0.92) return true;
      }
    }
  }
  return false;
}

function spawnObjectAt(x, z, t) {
  t = t || pickType();
  const model = t.build(t);
  const pivot = new THREE.Group();
  model.position.set(0, 0, 0);
  pivot.position.set(x, 0, z);
  pivot.add(model);
  scene.add(pivot);
  const o = {
    type: t, x, z, model, pivot,
    leanAngle: 0, leanCorners: 0,
    falling: false, committed: false, caught: false,
    fallT: 0, eater: null, topple: false, fallAxis: null, startAngle: 0, catchAngle: 0
  };
  objects.push(o);
  addToBuckets(o);
}
function initObjects() {
  for (const o of objects) { scene.remove(o.pivot); }
  objects = [];
  spatialBuckets = new Map();
  for (let i = 0; i < CONFIG.OBJECT_COUNT; i++) {
    let placed = false;
    for (let attempt = 0; attempt < 8 && !placed; attempt++) {
      const t = pickType();
      const x = (Math.random() - 0.5) * (CONFIG.WORLD_SIZE - 100);
      const z = (Math.random() - 0.5) * (CONFIG.WORLD_SIZE - 100);
      if (overlapsExisting(t, x, z)) continue;
      spawnObjectAt(x, z, t);
      placed = true;
    }
  }
}
function respawnObject(old) {
  removeFromBuckets(old);
  scene.remove(old.pivot);
  for (let attempt = 0; attempt < 8; attempt++) {
    const t = pickType();
    const x = (Math.random() - 0.5) * (CONFIG.WORLD_SIZE - 100);
    const z = (Math.random() - 0.5) * (CONFIG.WORLD_SIZE - 100);
    if (overlapsExisting(t, x, z)) continue;
    spawnObjectAt(x, z, t);
    return;
  }
  const t = pickType();
  spawnObjectAt((Math.random()-0.5)*(CONFIG.WORLD_SIZE-100), (Math.random()-0.5)*(CONFIG.WORLD_SIZE-100), t);
}
function resetLean(o) {
  o.pivot.position.set(o.x, 0, o.z);
  o.model.position.set(0, 0, 0);
  o.pivot.quaternion.identity();
  o.leanAngle = 0; o.leanCorners = 0;
}

function addArea(e, area) {
  const cur = Math.PI * e.r * e.r;
  e.r = Math.min(CONFIG.MAX_R, Math.sqrt((cur + area) / Math.PI));
}

// 倒れる向きの回転軸(穴の方向Dへ正しく倒れ込む向き)
function fallAxisToward(pivotX, pivotZ, targetX, targetZ) {
  const dx = targetX - pivotX, dz = targetZ - pivotZ;
  const dlen = Math.hypot(dx, dz) || 1;
  const Dx = dx / dlen, Dz = dz / dlen;
  return new THREE.Vector3(Dz, 0, -Dx);
}

function commitStraightDrop(o, e) {
  o.falling = true; o.committed = true; o.topple = false; o.fallT = 0; o.eater = e;
}
function commitTopple(o, e, pivotPos) {
  o.pivot.position.copy(pivotPos);
  o.model.position.set(o.x - pivotPos.x, 0, o.z - pivotPos.z);
  o.fallAxis = fallAxisToward(pivotPos.x, pivotPos.z, e.position.x, e.position.z);
  o.startAngle = o.leanAngle;

  const D = 2 * e.r; // 穴の直径(この瞬間でスナップショット)
  const L = o.type.height;
  if (L >= D) {
    o.caught = true;
    o.catchAngle = Math.asin(Math.min(1, D / L)) * 0.94; // 縁で止まる角度
  } else {
    o.caught = false;
  }
  o.falling = true; o.committed = true; o.topple = true; o.fallT = 0; o.eater = e;
}

// 底面がどれだけ穴の円に入っているかで重心を計算し、リアルタイムに傾ける。
// 中心(重心)が穴を越えたら後戻りできず確定する。
function updateLeanPhysics(o, e) {
  const halfW = o.type.hw, halfD = o.type.hd;
  const corners = [[-halfW,-halfD],[halfW,-halfD],[halfW,halfD],[-halfW,halfD]];
  let insideCount = 0; const outside = [];
  for (const [lx, lz] of corners) {
    const wx = o.x + lx, wz = o.z + lz;
    const d = Math.hypot(wx - e.position.x, wz - e.position.z);
    if (d < e.r) insideCount++; else outside.push({ x: wx, z: wz });
  }

  if (insideCount === 0) {
    if (o.leanAngle > 0.001) { resetLean(o); }
    return;
  }

  if (insideCount === 4) {
    // 円が底面よりずっと大きい:一瞬で全部入るので、倒れる猶予がなく垂直に落ちる
    commitStraightDrop(o, e);
    return;
  }

  // 1〜3角が入っている:重心方向へ連続的に(可逆的に)傾ける
  let px = 0, pz = 0;
  for (const c of outside) { px += c.x; pz += c.z; }
  px /= outside.length; pz /= outside.length;

  const MAX_LEAN = Math.PI * 0.24;
  const targetAngle = (insideCount / 4) * MAX_LEAN;
  o.leanAngle += (targetAngle - o.leanAngle) * 0.25;
  o.leanCorners = insideCount;

  const pivotPos = new THREE.Vector3(px, 0, pz);
  o.pivot.position.copy(pivotPos);
  o.model.position.set(o.x - pivotPos.x, 0, o.z - pivotPos.z);
  const axis = fallAxisToward(px, pz, e.position.x, e.position.z);
  o.pivot.quaternion.setFromAxisAngle(axis, o.leanAngle);

  // 重心(中心)が穴に十分深く入り、角もほとんど飲み込まれた時だけ後戻りできなくなる。
  // ここを緩くしすぎると、普通の速度で近づいただけで一瞬で確定してしまい、
  // 可逆的に傾く区間が体感できなくなる。
  const centerDist = Math.hypot(o.x - e.position.x, o.z - e.position.z);
  if (centerDist < e.r * 0.55 && insideCount >= 3) {
    commitTopple(o, e, pivotPos);
  }
}

function updateFalling(dt) {
  for (let i = objects.length - 1; i >= 0; i--) {
    const o = objects[i];
    if (!o.falling) continue;
    o.fallT += dt / 0.9;

    if (!o.topple) {
      // 垂直に落ちる(縮小はしない、沈むだけ)
      const t = Math.min(o.fallT, 1);
      o.pivot.position.y = -t * pitDepthFor(o.eater.r) * 0.85;
      if (t > 0.55) {
        const alpha = 1 - (t - 0.55) / 0.45;
        o.model.traverse(m => { if (m.material) { m.material.transparent = true; m.material.opacity = Math.max(0, alpha); } });
      }
      if (o.fallT >= 1) {
        o.eater.score += o.type.pts;
        addArea(o.eater, o.type.pts * CONFIG.GROWTH_MULT);
        if (o.eater.isPlayer) updateScoreUI();
        respawnObject(o); objects.splice(i, 1);
      }
      continue;
    }

    if (o.caught) {
      // 縁に引っかかって、途中まで倒れて起き上がる(食べられない)
      const riseStart = 0.45, holdEnd = 0.62;
      let angle;
      if (o.fallT < riseStart) {
        const tt = o.fallT / riseStart;
        angle = o.startAngle + tt * (o.catchAngle - o.startAngle);
      } else if (o.fallT < holdEnd) {
        angle = o.catchAngle;
      } else {
        const tt = Math.min(1, (o.fallT - holdEnd) / (1 - holdEnd));
        angle = o.catchAngle * (1 - tt);
      }
      o.pivot.quaternion.setFromAxisAngle(o.fallAxis, angle);
      if (o.fallT >= 1) {
        o.falling = false; o.committed = false; o.caught = false;
        resetLean(o);
      }
    } else {
      // 遮る物がないので、そのまま倒れ込んで穴に落ちる(縮小はしない、沈んで消える)
      const t = Math.min(o.fallT, 1);
      const angle = o.startAngle + t * (Math.PI * 0.62 - o.startAngle);
      o.pivot.quaternion.setFromAxisAngle(o.fallAxis, angle);
      o.pivot.position.y = -t * pitDepthFor(o.eater.r) * 0.85;
      if (t > 0.55) {
        const alpha = 1 - (t - 0.55) / 0.45;
        o.model.traverse(m => { if (m.material) { m.material.transparent = true; m.material.opacity = Math.max(0, alpha); } });
      }
      if (o.fallT >= 1) {
        o.eater.score += o.type.pts;
        addArea(o.eater, o.type.pts * CONFIG.GROWTH_MULT);
        if (o.eater.isPlayer) updateScoreUI();
        respawnObject(o); objects.splice(i, 1);
      }
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
  entities.forEach(e => { scene.remove(e.pit); scene.remove(e.floor); scene.remove(e.ring); });
  entities = [];
  entities.push(makeEntity(true, 0));
  for (let i = 0; i < CONFIG.NUM_BOTS; i++) entities.push(makeEntity(false, i));
  entities.forEach(e => { e.position.set((Math.random()-0.5)*400, 0, (Math.random()-0.5)*400); e.r = CONFIG.INIT_R; e.score = 0; updateEntityMesh(e); });
  player().position.set(0,0,0);
  initObjects();
  score = 0; timeLeft = CONFIG.GAME_TIME; timerAccum = 0;
  scoreVal.textContent = '0'; timeVal.textContent = String(CONFIG.GAME_TIME);
  timerBox.classList.remove('danger');
}
function startPlaying() {
  resetGame();
  running = true;
  document.getElementById('overlay').classList.add('hidden');
}

function botAI(bot, dt) {
  bot.targetStuckT = (bot.targetStuckT || 0) + dt;
  let target = null, flee = false, bestD = Infinity;
  for (const other of entities) {
    if (other === bot) continue;
    const d = Math.hypot(other.position.x - bot.position.x, other.position.z - bot.position.z);
    if (other.r > bot.r * 1.2 && d < 260) { target = { x: bot.position.x - (other.position.x-bot.position.x), z: bot.position.z - (other.position.z-bot.position.z) }; flee = true; break; }
  }
  if (!flee) {
    if (!bot.target || bot.target.falling || bot.targetStuckT > 2.5) {
      let best = null, bd = Infinity;
      for (const o of objects) {
        if (o.falling) continue;
        if (o.type.round) { if (bot.r < o.type.hw * 0.75) continue; }
        else { if (bot.r < Math.min(o.type.hw, o.type.hd) * 0.85) continue; }
        const d = Math.hypot(o.x-bot.position.x, o.z-bot.position.z);
        if (d < bd) { bd = d; best = o; }
      }
      bot.target = best;
      bot.targetStuckT = 0;
    }
    if (bot.target) target = { x: bot.target.x, z: bot.target.z };
  }
  const speed = speedFor(bot.r);
  if (target) {
    const dx = target.x - bot.position.x, dz = target.z - bot.position.z;
    const d = Math.hypot(dx,dz) || 1;
    if (!flee && d < bot.r * 0.4) {
      // 既に対象の真上まで来ている:これ以上追いかけようとすると
      // 毎フレーム行き過ぎては戻りを繰り返して震えて見えるので、ここで待つ
      bot.vx *= 0.7; bot.vz *= 0.7;
    } else {
      bot.vx = dx/d*speed; bot.vz = dz/d*speed;
    }
  } else {
    bot.wanderT -= dt;
    if (bot.wanderT <= 0) { bot.wanderAngle = Math.random()*Math.PI*2; bot.wanderT = 1.5+Math.random()*2; }
    bot.vx = Math.cos(bot.wanderAngle)*speed*0.6; bot.vz = Math.sin(bot.wanderAngle)*speed*0.6;
  }
  bot.position.x = Math.max(-CONFIG.WORLD_SIZE/2+6, Math.min(CONFIG.WORLD_SIZE/2-6, bot.position.x + bot.vx*dt));
  bot.position.z = Math.max(-CONFIG.WORLD_SIZE/2+6, Math.min(CONFIG.WORLD_SIZE/2-6, bot.position.z + bot.vz*dt));
}

function updateEntities(dt) {
  const p = player();
  const speed = speedFor(p.r);
  if (joy.active && joy.mag > 0.05) {
    p.vx = joy.dx * speed * joy.mag;
    p.vz = joy.dy * speed * joy.mag;
  } else { p.vx *= 0.85; p.vz *= 0.85; }
  p.position.x = Math.max(-CONFIG.WORLD_SIZE/2+6, Math.min(CONFIG.WORLD_SIZE/2-6, p.position.x + p.vx*dt));
  p.position.z = Math.max(-CONFIG.WORLD_SIZE/2+6, Math.min(CONFIG.WORLD_SIZE/2-6, p.position.z + p.vz*dt));

  for (let i = 1; i < entities.length; i++) botAI(entities[i], dt);
  entities.forEach(updateEntityMesh);
}

function checkEating() {
  for (const e of entities) {
    for (const o of objects) {
      if (o.falling) continue;
      if (o.type.round) {
        const d = Math.hypot(o.x - e.position.x, o.z - e.position.z);
        if (d < e.r * 0.72 && e.r > o.type.hw * 0.75) commitStraightDrop(o, e);
      } else {
        if (e.r < Math.min(o.type.hw, o.type.hd) * 0.85) continue;
        updateLeanPhysics(o, e);
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
  document.getElementById('retryBtn').addEventListener('click', startPlaying);
}

document.getElementById('startBtn').addEventListener('click', startPlaying);

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
  updateOcclusionFade();
  updateGroundHoles();
  updateCamera();
  renderer.render(scene, camera);
  requestAnimationFrame(loop);
}

resetGame();
document.getElementById('loading').style.display = 'none';
requestAnimationFrame(loop);
</script>
</body>
</html>
