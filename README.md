# void-eater
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>CITY HOLE - 街を飲み込め</title>
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
  }
  canvas { display: block; width: 100%; height: 100%; }

  #ui-layer {
    position: fixed; inset: 0; pointer-events: none; z-index: 10;
  }

  .hud {
    position: absolute; top: 24px; left: 24px; right: 24px;
    display: flex; justify-content: space-between; align-items: flex-start;
  }
  .hud-box {
    background: var(--panel); border-radius: 20px;
    padding: 12px 28px; text-align: center;
    box-shadow: 0 6px 18px rgba(30,50,60,0.08);
  }
  .hud-label {
    font-size: 11px; font-weight: 700; color: #a4b0b6; letter-spacing: 2px; margin-bottom: 2px;
  }
  .hud-value {
    font-size: 26px; font-weight: 800; color: var(--text);
  }
  #timer-box.danger .hud-value { color: #e0574a; }

  .leaderboard {
    position: absolute; top: 96px; right: 24px;
    background: var(--panel); border-radius: 18px; padding: 14px 18px;
    box-shadow: 0 6px 18px rgba(30,50,60,0.08);
    min-width: 150px;
  }
  .lb-row {
    display: flex; align-items: center; justify-content: space-between;
    margin-bottom: 9px; font-size: 13px; font-weight: 600; color: #9aa5aa;
  }
  .lb-row:last-child { margin-bottom: 0; }
  .lb-row.me { color: var(--accent); font-weight: 700; }
  .lb-dot { width: 10px; height: 10px; border-radius: 50%; margin-right: 8px; flex-shrink: 0; }
  .lb-name { flex: 1; text-align: left; }
  .lb-score { font-family: monospace; font-size: 14px; }

  #overlay {
    position: fixed; inset: 0; z-index: 20;
    background: rgba(40, 55, 65, 0.55); backdrop-filter: blur(6px);
    display: flex; align-items: center; justify-content: center; pointer-events: auto;
  }
  #overlay.hidden { display: none; }
  .panel {
    background: #fff; padding: 44px 40px; border-radius: 32px; text-align: center;
    box-shadow: 0 24px 60px rgba(20,35,45,0.25); width: 90%; max-width: 400px;
  }
  .title { font-size: 44px; font-weight: 800; color: var(--text); margin: 0 0 10px; letter-spacing: -1.5px;}
  .title span { color: var(--accent); }
  .desc { font-size: 14px; color: #93a0a6; margin-bottom: 32px; font-weight: 600; line-height: 1.6;}

  .result-rank { font-size: 30px; font-weight: 800; color: var(--accent); margin: 22px 0 6px; }
  .result-score { font-size: 19px; font-weight: 700; color: var(--text); margin-bottom: 32px; }

  button {
    background: var(--accent); color: #fff; border: none; border-radius: 100px;
    font-size: 18px; font-weight: 800; padding: 17px 48px; cursor: pointer;
    box-shadow: 0 6px 0 var(--accent-dark), 0 14px 20px rgba(31,182,168,0.25);
    transition: all 0.1s; width: 100%;
  }
  button:active { transform: translateY(6px); box-shadow: 0 0 0 var(--accent-dark); }
</style>
</head>
<body>

<canvas id="game"></canvas>

<div id="ui-layer">
  <div class="hud">
    <div class="hud-box">
      <div class="hud-label">SCORE</div>
      <div class="hud-value" id="scoreVal">0</div>
    </div>
    <div class="hud-box" id="timer-box">
      <div class="hud-label">TIME</div>
      <div class="hud-value" id="timeVal">90</div>
    </div>
  </div>
  <div class="leaderboard" id="lb"></div>
</div>

<div id="overlay">
  <div class="panel" id="screen-content"></div>
</div>

<script>
(function(){
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');

  let W, H;
  function resize() {
    W = canvas.width = window.innerWidth;
    H = canvas.height = window.innerHeight;
  }
  window.addEventListener('resize', resize);
  resize();

  // --- チューニング用コンフィグ ---
  const CONFIG = {
    WORLD_SIZE: 3500,
    NUM_BOTS: 3,
    OBJECT_GRID: 100,
    OBJECT_SKIP_CHANCE: 0.08,
    OVERLAP_MARGIN: 0.9,      // 配置時に他オブジェクトとどこまで近づけて良いか(1で接触ギリギリ)

    // 重力方式:サイズ比の閾値ではなく、「対象の何割が穴の上に来たか」で自然に落ちる
    GRAVITY_RANGE_MULT: 1.5,        // 穴の半径の何倍まで重力が届くか
    GRAVITY_STRENGTH: 130,          // 重力の強さ
    FALL_THRESHOLD_BASE: 0.5,       // 対象の面積の何割が穴の上に来たら落ちるか(基準)
    FALL_THRESHOLD_ROUND_BONUS: 0.14, // 丸い物ほど閾値が下がる=落ちやすい

    GROWTH_MULT: 34,   // 食べた時の成長量の係数(旧25→34、最大サイズに届きやすく)

    SPEED_MIN: 95,   // 小さい穴の速度
    SPEED_MAX: 270,  // 最大に近い穴の速度

    TILT_Y: 0.78,     // 縦方向の圧縮率。1で真上、小さいほど斜めから見下ろす感じに近づく
  };
  const WORLD_SIZE = CONFIG.WORLD_SIZE;
  const INIT_R = 15;
  const MAX_R = 400;
  const GAME_TIME = 130;

  const COLORS = {
    bg: '#7fae52',
    bgDark: '#6f9c46',
    bgLight: '#8fc25e',
    road: '#c9c6bc',
    roadLine: '#f4efe0',
    curb: '#a6a396',
    player: '#00d2d3',
    bots: ['#ff9f43', '#ee5253', '#5f27cd', '#10ac84', '#ff6b81'],
  };

  // 太陽の方向(常に左上から)。全オブジェクト・地面・穴の陰影をこの一方向で統一する
  const LIGHT = (function(){ const x=-0.55, y=-0.7, len=Math.hypot(x,y); return { x:x/len, y:y/len }; })();

  // 街のオブジェクト定義。round: 1=丸くて綺麗に吸い込める / 0=角ばって引っかかりやすい
  // サイズは細かい段階を用意し、穴がどの大きさでも「ギリギリ挑戦できる相手」が必ずいるようにする
  const TYPES = [
    { id:'person',        r:3,   h:8,   pts:1,   col:'#f1c40f', kind:'walker',    round:0.5 },
    { id:'dog',           r:4,   h:5,   pts:2,   col:'#c0785a', kind:'wanderer',  round:0.6 },
    { id:'balloon',       r:5.5, h:16,  pts:4,   col:'#ff6b9d', kind:'floater',   round:1.0 },
    { id:'cone',          r:7.5, h:6,   pts:3,   col:'#e67e22', kind:'static',    round:0.7 },
    { id:'hydrant',       r:10,  h:9,   pts:6,   col:'#e74c3c', kind:'static',    round:0.7 },
    { id:'bicycle',       r:14,  h:11,  pts:9,   col:'#2c3e50', kind:'static',    round:0.5 },
    { id:'bench',         r:19,  h:6,   pts:13,  col:'#9b59b6', kind:'static',    round:0.2 },
    { id:'lamp',          r:25,  h:22,  pts:18,  col:'#f9ca24', kind:'static',    round:0.7 },
    { id:'tree',          r:34,  h:30,  pts:24,  col:'#2ecc71', kind:'static',    round:0.85},
    { id:'car',           r:46,  h:14,  pts:32,  col:'#3498db', kind:'driver',    round:0.4 },
    { id:'silo',          r:62,  h:60,  pts:45,  col:'#8395a7', kind:'static',    round:1.0 },
    { id:'house',         r:84,  h:40,  pts:65,  col:'#e74c3c', kind:'static',    round:0.3 },
    { id:'complex',       r:113, h:70,  pts:95,  col:'#57606f', kind:'static',    round:0.25},
    { id:'towerRound',    r:153, h:150, pts:145, col:'#4b6584', kind:'static',    round:0.95},
    { id:'skyscraper',    r:206, h:210, pts:210, col:'#34495e', kind:'static',    round:0.3 },
    { id:'complexTower',  r:278, h:260, pts:320, col:'#2f3542', kind:'static',    round:0.25},
    { id:'landmark',      r:375, h:320, pts:550, col:'#c0975a', kind:'static',    round:0.55},
  ];

  let objects = [];
  let entities = [];
  let particles = [];

  let running = false;
  let timeLeft = GAME_TIME;
  let timerAccum = 0;
  let lastTime = 0;

  let camX = WORLD_SIZE / 2;
  let camY = WORLD_SIZE / 2;
  let curZoom = 2.3;

  let isPointerDown = false;
  let pointerX = W/2, pointerY = H/2;
  let joyCenter = { x: W/2, y: H/2 };

  canvas.addEventListener('pointerdown', e => {
    isPointerDown = true;
    pointerX = e.clientX; pointerY = e.clientY;
    joyCenter = { x: e.clientX, y: e.clientY };
  });
  canvas.addEventListener('pointermove', e => {
    if (isPointerDown) {
      pointerX = e.clientX; pointerY = e.clientY;
      const dx = pointerX - joyCenter.x;
      const dy = pointerY - joyCenter.y;
      const dist = Math.hypot(dx, dy);
      const maxJoy = 60;
      if (dist > maxJoy) {
        joyCenter.x += (dist - maxJoy) * (dx / dist);
        joyCenter.y += (dist - maxJoy) * (dy / dist);
      }
    }
  });
  canvas.addEventListener('pointerup', () => isPointerDown = false);
  canvas.addEventListener('pointercancel', () => isPointerDown = false);
  canvas.addEventListener('contextmenu', e => e.preventDefault());

  function player() { return entities[0]; }

  // 穴のサイズに応じた移動速度:小さいうちは遅く、育つほど速くなる
  function speedFor(r) {
    const t = Math.max(0, Math.min(1, (r - INIT_R) / (MAX_R - INIT_R)));
    const eased = Math.pow(t, 0.55);
    return CONFIG.SPEED_MIN + eased * (CONFIG.SPEED_MAX - CONFIG.SPEED_MIN);
  }

  // 簡易空間ハッシュ:大きい建物(直径最大750)にも対応できるバケット幅
  const BUCKET = 400;
  let spatialBuckets = new Map();
  function bucketKeyOf(x, y) { return Math.floor(x / BUCKET) + ',' + Math.floor(y / BUCKET); }
  function addToBuckets(obj) {
    const key = bucketKeyOf(obj.x, obj.y);
    if (!spatialBuckets.has(key)) spatialBuckets.set(key, new Set());
    spatialBuckets.get(key).add(obj);
    obj._bucketKey = key;
  }
  function removeFromBuckets(obj) {
    const set = spatialBuckets.get(obj._bucketKey);
    if (set) set.delete(obj);
  }
  function overlapsExisting(type, x, y) {
    const bx = Math.floor(x / BUCKET), by = Math.floor(y / BUCKET);
    for (let dx = -1; dx <= 1; dx++) {
      for (let dy = -1; dy <= 1; dy++) {
        const set = spatialBuckets.get((bx+dx) + ',' + (by+dy));
        if (!set) continue;
        for (const other of set) {
          const d = Math.hypot(other.x - x, other.y - y);
          if (d < (other.type.r + type.r) * CONFIG.OVERLAP_MARGIN) return true;
        }
      }
    }
    return false;
  }

  function pickType() {
    const roll = Math.random();
    let type;
    if (roll < 0.15) type = 'person';
    else if (roll < 0.27) type = 'dog';
    else if (roll < 0.37) type = 'balloon';
    else if (roll < 0.46) type = 'cone';
    else if (roll < 0.54) type = 'hydrant';
    else if (roll < 0.61) type = 'bicycle';
    else if (roll < 0.68) type = 'bench';
    else if (roll < 0.75) type = 'lamp';
    else if (roll < 0.85) type = 'tree';
    else if (roll < 0.92) type = 'car';
    else if (roll < 0.955) type = 'silo';
    else if (roll < 0.975) type = 'house';
    else if (roll < 0.985) type = 'complex';
    else if (roll < 0.991) type = 'towerRound';
    else if (roll < 0.996) type = 'skyscraper';
    else if (roll < 0.9985) type = 'complexTower';
    else type = 'landmark';
    return TYPES.find(tt => tt.id === type);
  }

  function makeObject(t, x, y) {
    const obj = {
      type: t, x, y,
      falling: false, fallT: 0, eater: null, startX: 0, startY: 0,
      seed: Math.random() * 1000, strain: 0, _wasStrained: false,
      leanAngle: 0, leanPivotX: x, leanPivotY: y, leanSign: 1
    };
    if (t.kind === 'walker') {
      obj.patrolCenterX = x; obj.patrolCenterY = y;
      obj.patrolR = 15 + Math.random() * 25;
      obj.phase = Math.random() * Math.PI * 2;
      obj.vx = 0; obj.vy = 0;
    } else if (t.kind === 'wanderer') {
      obj.vx = 0; obj.vy = 0;
      obj.wanderAngle = Math.random() * Math.PI * 2;
      obj.wanderT = Math.random() * 2;
    } else if (t.kind === 'floater') {
      obj.baseX = x; obj.baseY = y;
      obj.bobPhase = Math.random() * Math.PI * 2;
    } else if (t.kind === 'driver') {
      const horiz = Math.random() < 0.5;
      const spd = 35 + Math.random() * 25;
      obj.vx = horiz ? (Math.random() < 0.5 ? spd : -spd) : 0;
      obj.vy = horiz ? 0 : (Math.random() < 0.5 ? spd : -spd);
    }
    return obj;
  }

  // 食べられた後の再配置。近くで重ならない場所を探し、駄目なら周囲を広げて再挑戦
  function spawnObject(x, y) {
    for (let attempt = 0; attempt < 10; attempt++) {
      const t = pickType();
      const spread = 150 + attempt * 60;
      const px = Math.max(t.r + 5, Math.min(WORLD_SIZE - t.r - 5, x + (Math.random()-0.5) * spread));
      const py = Math.max(t.r + 5, Math.min(WORLD_SIZE - t.r - 5, y + (Math.random()-0.5) * spread));
      if (!overlapsExisting(t, px, py)) {
        const obj = makeObject(t, px, py);
        objects.push(obj);
        addToBuckets(obj);
        return;
      }
    }
    // 最終手段:多少の重なりは許容して確実に配置する
    const t = pickType();
    const px = Math.max(t.r+5, Math.min(WORLD_SIZE-t.r-5, Math.random()*WORLD_SIZE));
    const py = Math.max(t.r+5, Math.min(WORLD_SIZE-t.r-5, Math.random()*WORLD_SIZE));
    const obj = makeObject(t, px, py);
    objects.push(obj);
    addToBuckets(obj);
  }

  function initWorld() {
    objects = [];
    spatialBuckets = new Map();
    const GRID = CONFIG.OBJECT_GRID;
    for (let x = GRID/2; x < WORLD_SIZE; x += GRID) {
      for (let y = GRID/2; y < WORLD_SIZE; y += GRID) {
        if (Math.random() < CONFIG.OBJECT_SKIP_CHANCE) continue;
        for (let attempt = 0; attempt < 6; attempt++) {
          const t = pickType();
          const rx = x + (Math.random() - 0.5) * GRID * 0.9;
          const ry = y + (Math.random() - 0.5) * GRID * 0.9;
          if (rx - t.r < 5 || rx + t.r > WORLD_SIZE - 5 || ry - t.r < 5 || ry + t.r > WORLD_SIZE - 5) continue;
          if (overlapsExisting(t, rx, ry)) continue;
          const obj = makeObject(t, rx, ry);
          objects.push(obj);
          addToBuckets(obj);
          break;
        }
      }
    }
  }

  function initEntities() {
    entities = [];
    entities.push({
      isPlayer: true, name: 'YOU', color: COLORS.player,
      x: WORLD_SIZE/2, y: WORLD_SIZE/2, r: INIT_R,
      vx: 0, vy: 0, score: 0, respawnT: 0,
      rim: makeRim(), rubble: makeRubble()
    });
    for (let i = 0; i < CONFIG.NUM_BOTS; i++) {
      entities.push({
        isPlayer: false, name: 'CPU ' + (i+1), color: COLORS.bots[i % COLORS.bots.length],
        x: Math.random() * WORLD_SIZE, y: Math.random() * WORLD_SIZE, r: INIT_R,
        vx: 0, vy: 0, score: 0, respawnT: 0, wanderAngle: Math.random() * Math.PI * 2,
        rim: makeRim(), rubble: makeRubble()
      });
    }
    camX = entities[0].x;
    camY = entities[0].y;
    curZoom = 2.3;
  }
  function makeRim() {
    const pts = 14, arr = [];
    for (let i = 0; i < pts; i++) arr.push(0.88 + Math.random() * 0.2);
    return arr;
  }
  function makeRubble() {
    const arr = [];
    for (let i = 0; i < 6; i++) arr.push({ angle: Math.random()*Math.PI*2, distF: 0.95+Math.random()*0.12, size: 0.06+Math.random()*0.05 });
    return arr;
  }

  function addArea(e, area) {
    const currentArea = Math.PI * e.r * e.r;
    const newArea = currentArea + area;
    e.r = Math.min(MAX_R, Math.sqrt(newArea / Math.PI));
  }

  function pullObjectToward(o, e, dt, speed) {
    const dx = e.x - o.x, dy = e.y - o.y;
    const d = Math.hypot(dx, dy) || 1;
    const move = Math.min(d, speed * dt);
    o.x += (dx / d) * move;
    o.y += (dy / d) * move;
  }

  // 2つの円の重なり面積(対象がどれだけ穴の上に来ているかを面積で判定するため)
  function circleOverlapArea(d, r1, r2) {
    if (d >= r1 + r2) return 0;
    if (d <= Math.abs(r1 - r2)) { const m = Math.min(r1, r2); return Math.PI * m * m; }
    if (d < 1e-6) d = 1e-6;
    const r1sq = r1*r1, r2sq = r2*r2;
    let a1 = (d*d + r1sq - r2sq) / (2*d*r1);
    let a2 = (d*d + r2sq - r1sq) / (2*d*r2);
    a1 = Math.max(-1, Math.min(1, a1));
    a2 = Math.max(-1, Math.min(1, a2));
    const alpha = Math.acos(a1), beta = Math.acos(a2);
    return r1sq*(alpha - Math.sin(2*alpha)/2) + r2sq*(beta - Math.sin(2*beta)/2);
  }

  function startFall(o, e, tight) {
    o.falling = true;
    o.eater = e;
    o.startX = o.x; o.startY = o.y;
    o.fallT = 0;
    o.tight = tight;
  }

  function updateMovingObjects(dt) {
    for (const o of objects) {
      if (o.falling) continue;
      const k = o.type.kind;
      if (k === 'walker') {
        o.phase += dt * 1.1;
        o.vx = -Math.sin(o.phase) * o.patrolR * 1.1;
        o.vy = Math.cos(o.phase * 0.7) * o.patrolR * 0.7;
        o.x = o.patrolCenterX + Math.cos(o.phase) * o.patrolR;
        o.y = o.patrolCenterY + Math.sin(o.phase * 0.7) * o.patrolR * 0.6;
      } else if (k === 'wanderer') {
        o.wanderT -= dt;
        if (o.wanderT <= 0) {
          o.wanderAngle = Math.random() * Math.PI * 2;
          o.wanderT = 1.2 + Math.random() * 1.8;
        }
        o.vx = Math.cos(o.wanderAngle) * 45;
        o.vy = Math.sin(o.wanderAngle) * 45;
        o.x = Math.max(o.type.r, Math.min(WORLD_SIZE - o.type.r, o.x + o.vx * dt));
        o.y = Math.max(o.type.r, Math.min(WORLD_SIZE - o.type.r, o.y + o.vy * dt));
      } else if (k === 'floater') {
        o.bobPhase += dt * 1.4;
        o.y = o.baseY + Math.sin(o.bobPhase) * 8;
        o.x = o.baseX + Math.cos(o.bobPhase * 0.5) * 5;
      } else if (k === 'driver') {
        o.x += o.vx * dt;
        o.y += o.vy * dt;
        if (o.x < o.type.r || o.x > WORLD_SIZE - o.type.r) o.vx *= -1;
        if (o.y < o.type.r || o.y > WORLD_SIZE - o.type.r) o.vy *= -1;
        o.x = Math.max(o.type.r, Math.min(WORLD_SIZE - o.type.r, o.x));
        o.y = Math.max(o.type.r, Math.min(WORLD_SIZE - o.type.r, o.y));
      }
    }
  }

  function update(dt) {
    const p = player();

    if (p.respawnT > 0) {
      p.respawnT -= dt;
      if (p.respawnT <= 0) {
        p.x = Math.random() * WORLD_SIZE; p.y = Math.random() * WORLD_SIZE;
        p.r = INIT_R; p.score = Math.floor(p.score / 2);
      }
    } else {
      const speed = speedFor(p.r);
      if (isPointerDown) {
        const dx = pointerX - joyCenter.x;
        const dy = pointerY - joyCenter.y;
        const dist = Math.hypot(dx, dy);
        if (dist > 5) {
          p.vx = (dx / dist) * speed;
          p.vy = (dy / dist) * speed;
        }
      } else {
        p.vx *= 0.85; p.vy *= 0.85;
      }
      p.x = Math.max(p.r, Math.min(WORLD_SIZE - p.r, p.x + p.vx * dt));
      p.y = Math.max(p.r, Math.min(WORLD_SIZE - p.r, p.y + p.vy * dt));
    }

    for (let i = 1; i < entities.length; i++) {
      const bot = entities[i];
      if (bot.respawnT > 0) {
        bot.respawnT -= dt;
        if (bot.respawnT <= 0) {
          bot.x = Math.random() * WORLD_SIZE; bot.y = Math.random() * WORLD_SIZE;
          bot.r = INIT_R; bot.score = Math.floor(bot.score / 2);
        }
        continue;
      }

      const speed = speedFor(bot.r);
      let target = null;
      let flee = false;

      for (const e of entities) {
        if (e === bot || e.respawnT > 0) continue;
        if (e.r > bot.r * 1.25) {
          const d = Math.hypot(e.x - bot.x, e.y - bot.y);
          if (d < bot.r * 4 + e.r) {
            target = { x: bot.x - (e.x - bot.x), y: bot.y - (e.y - bot.y) };
            flee = true; break;
          }
        }
      }

      if (!flee) {
        let minDist = Infinity;
        for (const o of objects) {
          if (o.falling || bot.r < o.type.r * 0.7) continue;
          const d = Math.hypot(o.x - bot.x, o.y - bot.y);
          if (d < minDist && d < 400) { minDist = d; target = o; }
        }
      }

      if (target) {
        const dx = target.x - bot.x; const dy = target.y - bot.y;
        const d = Math.hypot(dx, dy) || 1;
        bot.vx = (dx / d) * speed; bot.vy = (dy / d) * speed;
      } else {
        bot.wanderAngle += (Math.random() - 0.5) * 1.5 * dt;
        bot.vx = Math.cos(bot.wanderAngle) * speed * 0.7;
        bot.vy = Math.sin(bot.wanderAngle) * speed * 0.7;
      }

      bot.x = Math.max(bot.r, Math.min(WORLD_SIZE - bot.r, bot.x + bot.vx * dt));
      bot.y = Math.max(bot.r, Math.min(WORLD_SIZE - bot.r, bot.y + bot.vy * dt));
    }

    updateMovingObjects(dt);

    // 当たり判定(重力方式)
    for (const e of entities) {
      if (e.respawnT > 0) continue;

      for (const o of objects) {
        if (o.falling) continue;
        const round = o.type.round;
        const dist = Math.hypot(e.x - o.x, e.y - o.y);

        // 重力:大きさに関わらず、近づいた物は穴に引き寄せられる。
        // 大きい(重い)物ほど動きにくい。
        const gravityRange = e.r * CONFIG.GRAVITY_RANGE_MULT;
        if (dist < gravityRange) {
          const closeness = 1 - dist / gravityRange;
          const massFactor = 1 / (1 + o.type.r / e.r);
          pullObjectToward(o, e, dt, CONFIG.GRAVITY_STRENGTH * closeness * massFactor);
        }

        const isRoundFoot = round >= 0.7;

        if (isRoundFoot) {
          // 丸い設置面:今まで通り面積の重なりで綺麗に落ちる(引っかからない)
          const threshold = CONFIG.FALL_THRESHOLD_BASE - (round - 0.5) * CONFIG.FALL_THRESHOLD_ROUND_BONUS;
          const overlapArea = circleOverlapArea(dist, e.r, o.type.r);
          const frac = overlapArea / (Math.PI * o.type.r * o.type.r);
          o.strain = Math.max(0, Math.min(1, frac / threshold));
          if (frac >= threshold) {
            startFall(o, e, frac < threshold * 1.4);
            o.topple = false;
          }
        } else {
          // 角ばった設置面:縁を一歩でも越えたら、その瞬間から重心が傾いてリアルタイムに
          // 回転し続ける。穴が離れれば起き上がって戻り、重心(頭)が穴の中心を越えて
          // しまったら後戻りできず、そのまま回転して落ちる。
          const fw = o.type.r * 0.85, fd = o.type.r * 0.55;
          const perim = [[-fw,-fd],[0,-fd],[fw,-fd],[fw,0],[fw,fd],[0,fd],[-fw,fd],[-fw,0]];
          let insideCount = 0; const outsidePts = [];
          for (const [lx, ly] of perim) {
            const wx = o.x + lx, wy = o.y + ly;
            if (Math.hypot(e.x - wx, e.y - wy) < e.r) insideCount++;
            else outsidePts.push({ x: wx, y: wy });
          }
          const frac = insideCount / perim.length;
          o.strain = frac;

          if (frac > 0 && outsidePts.length > 0) {
            let px = 0, py = 0;
            for (const p of outsidePts) { px += p.x; py += p.y; }
            px /= outsidePts.length; py /= outsidePts.length;
            const v1x = o.x - px, v1y = o.y - py, v2x = e.x - px, v2y = e.y - py;
            const sign = (v1x*v2y - v1y*v2x) >= 0 ? 1 : -1;
            o.leanPivotX = px; o.leanPivotY = py; o.leanSign = sign;
            const targetAngle = Math.min(frac / 0.85, 1) * (Math.PI * 0.42);
            o.leanAngle += (targetAngle - o.leanAngle) * Math.min(1, dt * 7);
          } else {
            o.leanAngle += (0 - o.leanAngle) * Math.min(1, dt * 7);
          }

          // 重心が穴の中心の内側まで来てしまったら、もう起き上がれず落ちる
          if (dist < e.r * 0.85) {
            startFall(o, e, true);
            o.topple = true;
            o.pivotX = o.leanPivotX; o.pivotY = o.leanPivotY; o.toppleSign = o.leanSign;
            o.toppleStartAngle = o.leanAngle;
          }
        }

        // 地図の外に出ないように保険
        o.x = Math.max(o.type.r, Math.min(WORLD_SIZE - o.type.r, o.x));
        o.y = Math.max(o.type.r, Math.min(WORLD_SIZE - o.type.r, o.y));
      }

      for (const other of entities) {
        if (other === e || other.respawnT > 0) continue;
        if (e.r > other.r * 1.25) {
          const dist = Math.hypot(e.x - other.x, e.y - other.y);
          if (dist < e.r * 0.8) {
            e.score += Math.floor(other.score * 0.5) + 50;
            addArea(e, other.r * other.r * Math.PI * 0.6);
            other.respawnT = 3.0;
            createPopParticles(other.x, other.y, other.color, 15);
          }
        }
      }
    }

    for (let i = objects.length - 1; i >= 0; i--) {
      const o = objects[i];
      if (o.falling) {
        const rate = o.tight ? 1.9 : 3.5;
        o.fallT += dt * rate;
        if (o.fallT >= 1) {
          const bonus = o.tight ? 1.5 : 1;
          o.eater.score += Math.round(o.type.pts * bonus);
          addArea(o.eater, o.type.pts * CONFIG.GROWTH_MULT * (o.tight ? 1.2 : 1));
          createPopParticles(o.x, o.y, o.eater.color, o.tight ? 26 : 15);
          removeFromBuckets(o);
          objects.splice(i, 1);
          spawnObject(Math.random() * WORLD_SIZE, Math.random() * WORLD_SIZE);
        }
      }
    }

    for (let i = particles.length - 1; i >= 0; i--) {
      const pt = particles[i];
      pt.x += pt.vx * dt; pt.y += pt.vy * dt;
      pt.life -= dt;
      if (pt.life <= 0) particles.splice(i, 1);
    }

    if (p.respawnT <= 0) {
      camX += (p.x - camX) * dt * 4.0;
      camY += (p.y - camY) * dt * 4.0;
      const targetZoom = Math.max(0.16, 2.3 / Math.pow(p.r / INIT_R, 0.6));
      curZoom += (targetZoom - curZoom) * dt * 3.0;
    }

    updateUI();
  }

  function createPopParticles(x, y, color, count) {
    count = count || 15;
    for (let i = 0; i < count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const spd = 50 + Math.random() * 150;
      particles.push({
        x, y, vx: Math.cos(angle)*spd, vy: Math.sin(angle)*spd,
        life: 0.6, maxLife: 0.6, color, size: 3 + Math.random()*4
      });
    }
  }

  function shade(color, percent) {
    let f = parseInt(color.slice(1), 16), t = percent < 0 ? 0 : 255, p = percent < 0 ? percent * -1 : percent;
    let R = f >> 16, G = f >> 8 & 0x00FF, B = f & 0x0000FF;
    return "#" + (0x1000000 + (Math.round((t - R) * p) + R) * 0x10000 + (Math.round((t - G) * p) + G) * 0x100 + (Math.round((t - B) * p) + B)).toString(16).slice(1);
  }

  // 斜め見下ろし視点:高さのある物は常に同じ方向へ傾く(放射状ではなく一方向)
  const LEAN_X = 0.16;
  const LEAN_Y = -0.6;
  function getTopPos(sx, sy, h) {
    return {
      tx: sx + h * LEAN_X,
      ty: sy + h * LEAN_Y
    };
  }

  function drawQuad(p1, p2, p3, p4, color) {
    ctx.fillStyle = color;
    ctx.beginPath(); ctx.moveTo(p1.x, p1.y); ctx.lineTo(p2.x, p2.y);
    ctx.lineTo(p3.x, p3.y); ctx.lineTo(p4.x, p4.y); ctx.closePath();
    ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.1)'; ctx.lineWidth = 1; ctx.stroke();
  }

  // 太陽と反対方向へ伸びる落ち影。高いオブジェクトほど影が長くなる
  function drawCastShadow(sx, sy, footW, footH, heightPx) {
    const ox = -LIGHT.x * heightPx * 0.5;
    const oy = -LIGHT.y * heightPx * 0.5 * 0.6;
    ctx.save();
    ctx.globalAlpha = 0.28;
    ctx.fillStyle = '#1a2b1f';
    ctx.beginPath();
    ctx.ellipse(sx + ox, sy + oy, footW + heightPx*0.12, footH*0.6 + heightPx*0.06, 0, 0, Math.PI*2);
    ctx.fill();
    ctx.restore();
  }

  function drawBox(sx, sy, w, h, col, scale, fallT) {
    const size = w * curZoom * scale * 0.5;
    const actualH = h * curZoom * scale;
    drawCastShadow(sx, sy, size, size*0.7, actualH);
    const { tx, ty } = getTopPos(sx, sy, actualH);

    const b1 = {x: sx - size, y: sy - size}; const b2 = {x: sx + size, y: sy - size};
    const b3 = {x: sx + size, y: sy + size}; const b4 = {x: sx - size, y: sy + size};
    const t1 = {x: tx - size, y: ty - size}; const t2 = {x: tx + size, y: ty - size};
    const t3 = {x: tx + size, y: ty + size}; const t4 = {x: tx - size, y: ty + size};

    const cSide1 = shade(col, -0.06); const cSide2 = shade(col, -0.42);

    ctx.globalAlpha = 1 - fallT * 0.6;

    const dx = tx - sx, dy = ty - sy;
    if (dy < 0) drawQuad(b4, b3, t3, t4, cSide1); else drawQuad(b1, b2, t2, t1, cSide1);
    if (dx < 0) drawQuad(b2, b3, t3, t2, cSide2); else drawQuad(b1, b4, t4, t1, cSide2);
    drawQuad(t1, t2, t3, t4, shade(col, 0.22));

    ctx.globalAlpha = 1.0;
  }

  function drawCylinder(sx, sy, r, h, col, scale, fallT) {
    const size = r * curZoom * scale;
    const actualH = h * curZoom * scale;
    drawCastShadow(sx, sy, size, size*0.7, actualH);
    const { tx, ty } = getTopPos(sx, sy, actualH);

    ctx.globalAlpha = 1 - fallT * 0.6;

    const angle = Math.atan2(ty - sy, tx - sx);
    const p1 = { x: sx + Math.cos(angle + Math.PI/2) * size, y: sy + Math.sin(angle + Math.PI/2) * size };
    const p2 = { x: sx + Math.cos(angle - Math.PI/2) * size, y: sy + Math.sin(angle - Math.PI/2) * size };
    const p3 = { x: tx + Math.cos(angle - Math.PI/2) * size, y: ty + Math.sin(angle - Math.PI/2) * size };
    const p4 = { x: tx + Math.cos(angle + Math.PI/2) * size, y: ty + Math.sin(angle + Math.PI/2) * size };

    drawQuad(p1, p2, p3, p4, shade(col, -0.32));

    ctx.fillStyle = shade(col, 0.18);
    ctx.beginPath(); ctx.arc(tx, ty, size, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.12)'; ctx.stroke();

    ctx.globalAlpha = 1.0;
    return { tx, ty };
  }

  function drawBalloon(sx, sy, t, scale, fallT) {
    ctx.save();
    ctx.globalAlpha = 1 - fallT * 0.6;
    const rad = t.r * 1.6 * curZoom * scale;
    const cy = sy - t.h * curZoom * scale;
    const grad = ctx.createRadialGradient(sx - rad*0.3, cy - rad*0.3, rad*0.15, sx, cy, rad);
    grad.addColorStop(0, shade(t.col, 0.4));
    grad.addColorStop(1, shade(t.col, -0.1));
    ctx.fillStyle = grad;
    ctx.beginPath(); ctx.arc(sx, cy, rad, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.25)'; ctx.lineWidth = Math.max(1, curZoom);
    ctx.beginPath(); ctx.moveTo(sx, cy + rad); ctx.lineTo(sx, sy); ctx.stroke();
    ctx.restore();
  }

  // 自転車:実際に押し出された(立体の)ホイール2つ+フレーム+サドルで構成
  function drawBicycle(sx, sy, t, scale, fallT) {
    const sep = t.r * 0.85 * curZoom * scale;
    const wheelR = t.r * 0.42;
    const wheelH = t.r * 0.14;
    const wx1 = sx - sep, wx2 = sx + sep;
    drawCylinder(wx1, sy, wheelR, wheelH, '#1c2026', scale, fallT);
    drawCylinder(wx2, sy, wheelR, wheelH, '#1c2026', scale, fallT);
    const frameY = sy - wheelH * curZoom * scale * 0.5;
    drawBox(sx, frameY, t.r * 1.4, t.r * 0.28, t.col, scale, fallT);
    drawCylinder(sx + t.r*0.25*curZoom*scale, frameY, t.r * 0.11, t.r * 0.85, shade(t.col, -0.25), scale, fallT);
  }

  // 円柱状の建物:輪郭が丸いので、穴にすっと綺麗にはまる感触を出す
  function drawSilo(sx, sy, t, scale, fallT) {
    drawCylinder(sx, sy, t.r, t.h, t.col, scale, fallT);
    drawCylinder(sx, sy, t.r * 0.97, t.h * 0.5, shade(t.col, 0.18), scale, fallT);
  }
  function drawTowerRound(sx, sy, t, scale, fallT) {
    drawCylinder(sx, sy, t.r * 0.8, t.h, t.col, scale, fallT);
    drawCylinder(sx, sy, t.r * 0.55, t.h * 1.12, shade(t.col, 0.2), scale, fallT);
  }

  // 複雑な構造の建物:非対称な増築部分があり、穴に引っかかりやすい印象を出す
  function drawHydrant(sx, sy, t, scale, fallT) {
    // 郵便ポスト:丸胴+スロット+てっぺんの蓋
    drawCylinder(sx, sy, t.r * 0.65, t.h * 0.75, '#c0392b', scale, fallT);
    drawCylinder(sx, sy - t.h * curZoom * scale * 0.72, t.r * 0.72, t.r * 0.22, '#a5281e', scale, fallT);
    ctx.save();
    ctx.globalAlpha = 1 - fallT * 0.6;
    ctx.fillStyle = 'rgba(0,0,0,0.4)';
    const slotW = t.r * 0.5 * curZoom * scale, slotH = t.r * 0.08 * curZoom * scale;
    ctx.fillRect(sx - slotW/2, sy - t.h*curZoom*scale*0.58, slotW, slotH);
    ctx.restore();
  }
  function drawLamp(sx, sy, t, scale, fallT) {
    drawCylinder(sx, sy, t.r * 0.11, t.h, '#33363f', scale, fallT);
    const poleTopY = sy - t.h * curZoom * scale * 0.95;
    const armLen = t.r * 0.95 * curZoom * scale;
    ctx.save();
    ctx.globalAlpha = 1 - fallT * 0.6;
    ctx.strokeStyle = '#33363f';
    ctx.lineWidth = Math.max(1.5, t.r * 0.09 * curZoom * scale);
    ctx.lineCap = 'round';
    ctx.beginPath();
    ctx.moveTo(sx, poleTopY);
    ctx.lineTo(sx + armLen, poleTopY - t.r * 0.22 * curZoom * scale);
    ctx.stroke();
    const lampX = sx + armLen, lampY = poleTopY - t.r * 0.3 * curZoom * scale;
    const lampR = t.r * 0.55 * curZoom * scale;
    const glow = ctx.createRadialGradient(lampX, lampY, 0, lampX, lampY, lampR * 1.8);
    glow.addColorStop(0, 'rgba(255,244,200,0.85)');
    glow.addColorStop(1, 'rgba(255,244,200,0)');
    ctx.fillStyle = glow;
    ctx.beginPath(); ctx.arc(lampX, lampY, lampR * 1.8, 0, Math.PI*2); ctx.fill();
    ctx.fillStyle = '#fff6d8';
    ctx.beginPath(); ctx.arc(lampX, lampY, lampR * 0.55, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = '#2a2d34'; ctx.lineWidth = Math.max(1, t.r*0.06*curZoom*scale);
    ctx.stroke();
    ctx.restore();
  }
  // 戸建て:三角屋根・煙突・窓・玄関ドアがある家
  function drawHouse2(sx, sy, t, scale, fallT) {
    const wallH = t.h * 0.62;
    drawBox(sx, sy, t.r * 1.5, wallH, t.col, scale, fallT);
    const halfW = t.r * 1.5 * curZoom * scale * 0.5;
    const eaveY = sy - t.h * curZoom * scale * 0.6;
    const roofTopY = eaveY - t.r * 0.6 * curZoom * scale;
    ctx.save();
    ctx.globalAlpha = 1 - fallT * 0.6;
    ctx.beginPath();
    ctx.moveTo(sx - halfW * 1.1, eaveY);
    ctx.lineTo(sx + halfW * 1.1, eaveY);
    ctx.lineTo(sx, roofTopY);
    ctx.closePath();
    ctx.fillStyle = '#8a4a3a';
    ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.18)'; ctx.stroke();
    ctx.fillStyle = 'rgba(160,210,255,0.85)';
    const winY = sy - t.h * curZoom * scale * 0.35;
    ctx.fillRect(sx - halfW*0.65, winY - halfW*0.14, halfW*0.34, halfW*0.3);
    ctx.fillRect(sx + halfW*0.32, winY - halfW*0.14, halfW*0.34, halfW*0.3);
    ctx.strokeStyle = 'rgba(255,255,255,0.6)'; ctx.lineWidth=1;
    ctx.strokeRect(sx - halfW*0.65, winY - halfW*0.14, halfW*0.34, halfW*0.3);
    ctx.strokeRect(sx + halfW*0.32, winY - halfW*0.14, halfW*0.34, halfW*0.3);
    ctx.fillStyle = '#5c3a26';
    ctx.fillRect(sx - halfW*0.14, sy - halfW*0.1, halfW*0.28, halfW*0.62);
    ctx.restore();
    drawBox(sx + halfW*0.55, eaveY + halfW*0.1, t.r*0.16, t.h*0.32, '#7a6a5a', scale, fallT);
  }
  function drawComplex(sx, sy, t, scale, fallT) {
    const w = t.r * 1.5, h = t.h;
    drawBox(sx, sy, w, h, t.col, scale, fallT);
    const off = t.r * 0.85 * curZoom * scale;
    drawBox(sx + off, sy + off * 0.4, t.r * 0.85, t.h * 0.55, shade(t.col, -0.12), scale, fallT);
    // バルコニー(段ごとに小さく張り出す板)
    ctx.save();
    ctx.globalAlpha = 1 - fallT * 0.6;
    const halfW = w * curZoom * scale * 0.5;
    const topY = sy - h * curZoom * scale * 0.85;
    for (let i = 0; i < 3; i++) {
      const by = topY + i * h * curZoom * scale * 0.24;
      ctx.fillStyle = shade(t.col, 0.05);
      ctx.fillRect(sx - halfW*0.7, by, halfW*1.4, h*curZoom*scale*0.03);
      ctx.strokeStyle = 'rgba(0,0,0,0.2)'; ctx.lineWidth = 1;
      ctx.strokeRect(sx - halfW*0.7, by, halfW*1.4, h*curZoom*scale*0.03);
    }
    ctx.restore();
  }
  function drawComplexTower(sx, sy, t, scale, fallT) {
    drawBox(sx, sy, t.r * 1.25, t.h, t.col, scale, fallT);
    const off = t.r * 0.8 * curZoom * scale;
    drawBox(sx - off, sy + off * 0.35, t.r * 0.78, t.h * 0.6, shade(t.col, -0.15), scale, fallT);
    drawCylinder(sx, sy - t.h * curZoom * scale * 0.55, t.r * 0.1, t.r * 1.3, '#ff5555', scale, fallT);
  }
  function drawLandmark(sx, sy, t, scale, fallT) {
    drawCylinder(sx, sy, t.r, t.h * 0.35, t.col, scale, fallT);
    drawBox(sx, sy, t.r * 0.55, t.h, shade(t.col, 0.12), scale, fallT);
    drawCylinder(sx, sy - t.h * curZoom * scale * 0.9, t.r * 0.11, t.r * 1.1, '#ffd24d', scale, fallT);
  }
  function drawHydrant(sx, sy, t, scale, fallT) {
    drawCylinder(sx, sy, t.r * 0.75, t.h, t.col, scale, fallT);
  }
  function drawLamp(sx, sy, t, scale, fallT) {
    drawCylinder(sx, sy, t.r * 0.15, t.h, '#3d4150', scale, fallT);
    drawCylinder(sx, sy - t.h * curZoom * scale * 0.85, t.r * 0.5, t.r * 0.5, t.col, scale, fallT);
  }

  // 低ポリの多面クレーター。角度ごとに面を分割し、光源との向きでシェードを変える
  function craterPt(sx, sy, rad, radY, rim, ringT, i, segs) {
    const a = (i / segs) * Math.PI * 2;
    const wob = rim ? rim[i % rim.length] : 1;
    return { x: sx + Math.cos(a) * rad * ringT * wob, y: sy + Math.sin(a) * radY * ringT * wob, a };
  }
  function drawCrater(sx, sy, rad, radY, rim) {
    const SEGS = 14;
    const RING_T = [1.0, 0.6];
    for (let ring = 0; ring < RING_T.length - 1; ring++) {
      const rimOuter = ring === 0 ? rim : null;
      const depthDark = -0.02 - ring * 0.22;
      for (let i = 0; i < SEGS; i++) {
        const p1 = craterPt(sx, sy, rad, radY, rimOuter, RING_T[ring], i, SEGS);
        const p2 = craterPt(sx, sy, rad, radY, rimOuter, RING_T[ring], i+1, SEGS);
        const p3 = craterPt(sx, sy, rad, radY, null, RING_T[ring+1], i+1, SEGS);
        const p4 = craterPt(sx, sy, rad, radY, null, RING_T[ring+1], i, SEGS);
        const mid = (p1.a + p2.a) / 2;
        const facing = Math.cos(mid) * (-LIGHT.x) + Math.sin(mid) * (-LIGHT.y);
        ctx.beginPath();
        ctx.moveTo(p1.x, p1.y); ctx.lineTo(p2.x, p2.y); ctx.lineTo(p3.x, p3.y); ctx.lineTo(p4.x, p4.y);
        ctx.closePath();
        ctx.fillStyle = shade('#6b5a44', depthDark + facing * 0.14);
        ctx.fill();
      }
    }
    // 中は真っ暗闇。ここが主役になるよう、大きめの黒い円で占める
    const voidGrad = ctx.createRadialGradient(sx, sy, 0, sx, sy, rad*RING_T[RING_T.length-1]);
    voidGrad.addColorStop(0, '#000000');
    voidGrad.addColorStop(0.85, '#05070a');
    voidGrad.addColorStop(1, '#0d1116');
    ctx.beginPath();
    ctx.ellipse(sx, sy, rad*RING_T[RING_T.length-1], radY*RING_T[RING_T.length-1], 0, 0, Math.PI*2);
    ctx.fillStyle = voidGrad;
    ctx.fill();
  }
  function drawRubble(sx, sy, rad, radY, rubble) {
    for (const rb of rubble) {
      const rx = sx + Math.cos(rb.angle) * rad * rb.distF;
      const ry = sy + Math.sin(rb.angle) * radY * rb.distF;
      const s = rad * rb.size;
      ctx.beginPath();
      ctx.moveTo(rx, ry - s); ctx.lineTo(rx + s*0.85, ry + s*0.55); ctx.lineTo(rx, ry + s*0.55);
      ctx.closePath(); ctx.fillStyle = '#5d4f3c'; ctx.fill();
      ctx.beginPath();
      ctx.moveTo(rx, ry - s); ctx.lineTo(rx - s*0.85, ry + s*0.55); ctx.lineTo(rx, ry + s*0.55);
      ctx.closePath(); ctx.fillStyle = '#8a7960'; ctx.fill();
    }
  }

  function draw() {
    ctx.fillStyle = '#1b1e24'; // マップ外(圏外)の色
    ctx.fillRect(0, 0, W, H);

    function toSX(x) { return (x - camX) * curZoom + W/2; }
    function toSY(y) { return (y - camY) * curZoom * CONFIG.TILT_Y + H/2; }

    const wx0 = toSX(0), wy0 = toSY(0);
    const ww = WORLD_SIZE * curZoom, wh = WORLD_SIZE * curZoom * CONFIG.TILT_Y;

    ctx.fillStyle = COLORS.bg;
    ctx.fillRect(wx0, wy0, ww, wh);

    ctx.save();
    ctx.beginPath();
    ctx.rect(wx0, wy0, ww, wh);
    ctx.clip();

    const gridSizeX = 250 * curZoom;
    const gridSizeY = 250 * curZoom * CONFIG.TILT_Y;
    const offX = toSX(0) % gridSizeX;
    const offY = toSY(0) % gridSizeY;

    // 縁石(道路の少し外側、暗め)
    ctx.strokeStyle = COLORS.curb;
    ctx.lineWidth = 38 * curZoom;
    ctx.lineCap = 'square';
    for (let x = offX - gridSizeX; x < W + gridSizeX; x += gridSizeX) {
      ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke();
    }
    for (let y = offY - gridSizeY; y < H + gridSizeY; y += gridSizeY) {
      ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke();
    }
    // 舗装本体
    ctx.strokeStyle = COLORS.road;
    ctx.lineWidth = 30 * curZoom;
    for (let x = offX - gridSizeX; x < W + gridSizeX; x += gridSizeX) {
      ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke();
    }
    for (let y = offY - gridSizeY; y < H + gridSizeY; y += gridSizeY) {
      ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke();
    }
    // 車線の破線(ワールド座標基準でパターンを固定し、カメラ移動でずれないようにする)
    ctx.strokeStyle = COLORS.roadLine;
    ctx.lineWidth = 3 * curZoom;
    const dashPeriod = 26 * curZoom;
    ctx.setLineDash([14*curZoom, 12*curZoom]);
    ctx.lineDashOffset = ((-toSY(0) % dashPeriod) + dashPeriod) % dashPeriod;
    for (let x = offX - gridSizeX; x < W + gridSizeX; x += gridSizeX) {
      ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke();
    }
    ctx.lineDashOffset = ((-toSX(0) % dashPeriod) + dashPeriod) % dashPeriod;
    for (let y = offY - gridSizeY; y < H + gridSizeY; y += gridSizeY) {
      ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke();
    }
    ctx.setLineDash([]);

    // 芝の質感(地図座標に固定された疑似ランダムの葉っぱ模様)
    const patchStep = 55;
    const px0 = Math.max(0, Math.floor((camX - (W/2)/curZoom) / patchStep) * patchStep);
    const px1 = Math.min(WORLD_SIZE, px0 + (W/curZoom) + patchStep*2);
    const py0 = Math.max(0, Math.floor((camY - (H/2)/(curZoom*CONFIG.TILT_Y)) / patchStep) * patchStep);
    const py1 = Math.min(WORLD_SIZE, py0 + (H/(curZoom*CONFIG.TILT_Y)) + patchStep*2);
    for (let gx = px0; gx < px1; gx += patchStep) {
      for (let gy = py0; gy < py1; gy += patchStep) {
        const seed = Math.sin(gx*12.9898 + gy*78.233) * 43758.5453;
        const rv = seed - Math.floor(seed);
        if (rv > 0.5) continue;
        const jx = gx + ((seed*7)%1) * patchStep;
        const jy = gy + ((seed*13)%1) * patchStep;
        const sx2 = toSX(jx), sy2 = toSY(jy);
        if (sx2 < -10 || sx2 > W+10 || sy2 < -10 || sy2 > H+10) continue;
        ctx.strokeStyle = rv < 0.25 ? COLORS.bgDark : COLORS.bgLight;
        ctx.lineWidth = 2 * curZoom;
        const ang = rv * Math.PI;
        ctx.beginPath();
        ctx.moveTo(sx2 - Math.cos(ang)*5*curZoom, sy2 + Math.sin(ang)*2.5*curZoom);
        ctx.lineTo(sx2 + Math.cos(ang)*5*curZoom, sy2 - Math.sin(ang)*2.5*curZoom);
        ctx.stroke();
      }
    }
    ctx.restore();

    // マップの端をハザードテープ風の縞模様で明示する
    ctx.save();
    ctx.lineWidth = 16 * curZoom;
    ctx.setLineDash([20 * curZoom, 20 * curZoom]);
    ctx.strokeStyle = '#1a1a1a';
    ctx.strokeRect(wx0, wy0, ww, wh);
    ctx.lineDashOffset = 20 * curZoom;
    ctx.strokeStyle = '#f9ca24';
    ctx.strokeRect(wx0, wy0, ww, wh);
    ctx.restore();

    for (const e of entities) {
      if (e.respawnT > 0) continue;
      const sx = toSX(e.x); const sy = toSY(e.y);
      const rad = e.r * curZoom;
      const radY = rad * CONFIG.TILT_Y;

      // 掘り返された地面の縁(ダート)
      ctx.beginPath();
      for (let i = 0; i <= 14; i++) {
        const a = (i/14)*Math.PI*2;
        const wob = e.rim[i % e.rim.length];
        const rx = sx + Math.cos(a)*rad*1.06*wob, ry = sy + Math.sin(a)*radY*1.06*wob;
        if (i===0) ctx.moveTo(rx,ry); else ctx.lineTo(rx,ry);
      }
      ctx.closePath();
      ctx.fillStyle = shade('#6f9c46', -0.25);
      ctx.fill();

      drawCrater(sx, sy, rad, radY, e.rim);
      drawRubble(sx, sy, rad, radY, e.rubble);

      ctx.strokeStyle = e.color; ctx.globalAlpha = 0.55; ctx.lineWidth = Math.max(1.5, rad*0.045);
      ctx.beginPath(); ctx.ellipse(sx, sy, rad*1.01, radY*1.01, 0, 0, Math.PI*2); ctx.stroke();
      ctx.globalAlpha = 1;

      if (!e.isPlayer) {
        ctx.fillStyle = 'rgba(20,30,25,0.6)';
        ctx.font = `bold 12px sans-serif`; ctx.textAlign = 'center';
        ctx.fillText(e.name, sx, sy - rad - 15);
      }
    }

    let drawList = [...objects];
    particles.forEach(p => drawList.push({ isPt: true, ...p }));
    drawList.sort((a, b) => a.y - b.y);

    for (const o of drawList) {
      if (o.isPt) {
        const sx = toSX(o.x), sy = toSY(o.y);
        ctx.globalAlpha = o.life / o.maxLife;
        ctx.fillStyle = o.color;
        ctx.beginPath(); ctx.arc(sx, sy, o.size * curZoom, 0, Math.PI*2); ctx.fill();
        ctx.globalAlpha = 1.0;
        continue;
      }

      let curX = o.x, curY = o.y, scale = 1;
      let toppleAngle = 0, pivotSx = 0, pivotSy = 0;
      if (o.falling && o.eater) {
        const ease = o.fallT * o.fallT;
        if (o.topple) {
          curX = o.startX; curY = o.startY;
          scale = 1 - ease * 0.8;
          const startA = o.toppleStartAngle || 0;
          toppleAngle = (startA + ease * (Math.PI * 1.15 - startA)) * o.toppleSign;
          pivotSx = toSX(o.pivotX); pivotSy = toSY(o.pivotY);
        } else {
          curX = o.startX + (o.eater.x - o.startX) * ease;
          curY = o.startY + (o.eater.y - o.startY) * ease;
          scale = 1 - Math.pow(o.fallT, 2);
        }
        if (scale <= 0.05) continue;
      }

      let sx = toSX(curX); let sy = toSY(curY);
      const checkR = o.type.h * curZoom * 2;
      if (sx < -checkR || sx > W + checkR || sy < -checkR || sy > H + checkR) continue;

      const isRoundShape = o.type.round >= 0.7;
      if (!o.falling && isRoundShape && o.strain > 0) {
        sx += Math.sin(performance.now() * 0.025 + o.seed) * o.strain * 3 * curZoom;
      }

      // 角ばった物のリアルタイムな傾き(まだ落ちていない間も、縁を越えた分だけ常時傾く)
      if (!o.falling && !isRoundShape && o.leanAngle > 0.01) {
        ctx.save();
        const lpx = toSX(o.leanPivotX), lpy = toSY(o.leanPivotY);
        ctx.translate(lpx, lpy);
        ctx.rotate(o.leanAngle * o.leanSign);
        ctx.translate(-lpx, -lpy);
      }

      if (o.falling && o.topple) {
        ctx.save();
        ctx.translate(pivotSx, pivotSy);
        ctx.rotate(toppleAngle);
        ctx.translate(-pivotSx, -pivotSy);
      }

      const squeezing = o.falling && o.tight && !o.topple;
      if (squeezing) {
        const sq = 1 - o.fallT * 0.35;
        ctx.save();
        ctx.translate(sx, sy);
        ctx.scale(sq, 1 / Math.sqrt(sq));
        ctx.translate(-sx, -sy);
      }

      const t = o.type;
      if (t.id === 'person') {
        drawCylinder(sx, sy, t.r, t.h, t.col, scale, o.fallT);
        drawCylinder(sx, sy, t.r*0.8, t.h+3, '#ffddaa', scale, o.fallT);
      } else if (t.id === 'dog') {
        const ang = Math.atan2(o.vy || 0, o.vx || 1);
        const hOff = t.r * 1.3 * curZoom * scale;
        const hx = sx + Math.cos(ang) * hOff, hy = sy + Math.sin(ang) * hOff * 0.5;
        drawCylinder(sx, sy, t.r, t.h*0.7, t.col, scale, o.fallT);
        drawCylinder(hx, hy, t.r*0.55, t.h*0.9, shade(t.col, 0.15), scale, o.fallT);
      } else if (t.id === 'balloon') {
        drawBalloon(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'cone') {
        drawCylinder(sx, sy, t.r, t.h, t.col, scale, o.fallT);
      } else if (t.id === 'hydrant') {
        drawHydrant(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'bicycle') {
        drawBicycle(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'lamp') {
        drawLamp(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'tree') {
        drawCylinder(sx, sy, t.r*0.3, t.h*0.4, '#a0522d', scale, o.fallT);
        drawBox(sx, sy, t.r*1.8, t.h, t.col, scale, o.fallT);
      } else if (t.id === 'car') {
        drawBox(sx, sy, t.r*1.5, t.h*0.5, t.col, scale, o.fallT);
        drawBox(sx, sy, t.r*0.8, t.h, '#ecf0f1', scale, o.fallT);
      } else if (t.id === 'silo') {
        drawSilo(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'house') {
        drawHouse2(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'complex') {
        drawComplex(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'towerRound') {
        drawTowerRound(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'complexTower') {
        drawComplexTower(sx, sy, t, scale, o.fallT);
      } else if (t.id === 'landmark') {
        drawLandmark(sx, sy, t, scale, o.fallT);
      } else {
        drawBox(sx, sy, t.r*1.6, t.h, t.col, scale, o.fallT);
      }

      if (squeezing) ctx.restore();
      if (o.falling && o.topple) ctx.restore();
      if (!o.falling && !isRoundShape && o.leanAngle > 0.01) ctx.restore();
    }

    if (isPointerDown) {
      ctx.save();
      ctx.strokeStyle = 'rgba(255, 255, 255, 0.4)'; ctx.lineWidth = 3;
      ctx.beginPath(); ctx.arc(joyCenter.x, joyCenter.y, 60, 0, Math.PI*2); ctx.stroke();
      ctx.fillStyle = 'rgba(255, 255, 255, 0.9)';
      ctx.beginPath(); ctx.arc(pointerX, pointerY, 20, 0, Math.PI*2); ctx.fill();
      ctx.restore();
    }
  }

  function updateUI() {
    document.getElementById('scoreVal').textContent = player().score;
    document.getElementById('timeVal').textContent = timeLeft;
    const tb = document.getElementById('timer-box');
    if (timeLeft <= 10) tb.classList.add('danger');
    else tb.classList.remove('danger');

    const sorted = [...entities].sort((a, b) => b.score - a.score);
    const lbHTML = sorted.map((e, idx) => `
      <div class="lb-row ${e.isPlayer ? 'me' : ''}">
        <div class="lb-dot" style="background:${e.color}"></div>
        <div class="lb-name">${idx+1}. ${e.name}</div>
        <div class="lb-score">${e.score}</div>
      </div>
    `).join('');
    document.getElementById('lb').innerHTML = lbHTML;
  }

  function loop(now) {
    if (!lastTime) lastTime = now;
    const dt = Math.min((now - lastTime) / 1000, 0.05);
    lastTime = now;

    if (running) {
      update(dt);

      timerAccum += dt;
      if (timerAccum >= 1) {
        timerAccum -= 1;
        timeLeft--;
        if (timeLeft <= 0) endGame();
      }
    }

    draw();
    requestAnimationFrame(loop);
  }

  function startGame() {
    initWorld();
    initEntities();
    timeLeft = GAME_TIME;
    timerAccum = 0;
    lastTime = performance.now();
    running = true;
    document.getElementById('overlay').classList.add('hidden');
  }

  function endGame() {
    running = false;
    const sorted = [...entities].sort((a, b) => b.score - a.score);
    const rank = sorted.findIndex(e => e.isPlayer) + 1;

    const panel = document.getElementById('screen-content');
    panel.innerHTML = `
      <p class="title">TIME UP!</p>
      <div class="result-rank">RANK: ${rank} / ${entities.length}</div>
      <div class="result-score">FINAL SCORE: ${player().score}</div>
      <button id="retryBtn">PLAY AGAIN</button>
    `;
    document.getElementById('overlay').classList.remove('hidden');
    document.getElementById('retryBtn').addEventListener('click', startGame);
  }

  function showTitle() {
    const panel = document.getElementById('screen-content');
    panel.innerHTML = `
      <p class="title">CITY <span>HOLE</span></p>
      <p class="desc">指でスライドして街を飲み込め。<br>大きなライバルからは逃げろ！</p>
      <button id="startBtn">START</button>
    `;
    document.getElementById('startBtn').addEventListener('click', startGame);
  }

  showTitle();
  requestAnimationFrame(loop);

})();
</script>
</body>
</html>
