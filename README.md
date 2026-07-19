# void-eater
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>CITY HOLE - 街を飲み込め</title>
<style>
  :root {
    --bg: #87ceeb;
    --panel: rgba(255, 255, 255, 0.95);
    --text: #2c3e50;
    --accent: #00d2d3;
    --accent-dark: #00a8a8;
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
    position: absolute; top: 20px; left: 20px; right: 20px;
    display: flex; justify-content: space-between; align-items: flex-start;
  }
  .hud-box {
    background: var(--panel); border-radius: 16px;
    padding: 10px 24px; text-align: center;
    box-shadow: 0 8px 24px rgba(0,0,0,0.1);
  }
  .hud-label {
    font-size: 12px; font-weight: 800; color: #95a5a6; letter-spacing: 1px; margin-bottom: 2px;
  }
  .hud-value {
    font-size: 28px; font-weight: 900; color: var(--text);
  }
  #timer-box.danger .hud-value { color: #e74c3c; }

  .leaderboard {
    position: absolute; top: 90px; right: 20px;
    background: var(--panel); border-radius: 16px; padding: 12px 16px;
    box-shadow: 0 8px 24px rgba(0,0,0,0.1);
    min-width: 160px;
  }
  .lb-row {
    display: flex; align-items: center; justify-content: space-between;
    margin-bottom: 8px; font-size: 14px; font-weight: 700; color: #7f8c8d;
  }
  .lb-row:last-child { margin-bottom: 0; }
  .lb-row.me { color: var(--accent); }
  .lb-dot { width: 12px; height: 12px; border-radius: 50%; margin-right: 8px; flex-shrink: 0; }
  .lb-name { flex: 1; text-align: left; }
  .lb-score { font-family: monospace; font-size: 15px; }

  #overlay {
    position: fixed; inset: 0; z-index: 20;
    background: rgba(44, 62, 80, 0.7); backdrop-filter: blur(6px);
    display: flex; align-items: center; justify-content: center; pointer-events: auto;
  }
  #overlay.hidden { display: none; }
  .panel {
    background: #fff; padding: 40px; border-radius: 28px; text-align: center;
    box-shadow: 0 20px 50px rgba(0,0,0,0.3); width: 90%; max-width: 400px;
  }
  .title { font-size: 46px; font-weight: 900; color: var(--text); margin: 0 0 10px; letter-spacing: -2px;}
  .title span { color: var(--accent); }
  .desc { font-size: 15px; color: #7f8c8d; margin-bottom: 30px; font-weight: bold; line-height: 1.5;}

  .result-rank { font-size: 32px; font-weight: 900; color: var(--accent); margin: 20px 0 5px; }
  .result-score { font-size: 20px; font-weight: 800; color: var(--text); margin-bottom: 30px; }

  button {
    background: var(--accent); color: #fff; border: none; border-radius: 100px;
    font-size: 20px; font-weight: 900; padding: 18px 48px; cursor: pointer;
    box-shadow: 0 6px 0 var(--accent-dark), 0 15px 20px rgba(0,210,211,0.3);
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

    EAT_MIN_RATIO: 1.05,       // これ未満は完全にすり抜け
    EAT_EASY_RATIO: 1.6,       // 基準の「余裕吸い込み」しきい値(丸い物ほど下がる)

    EASY_PULL_RADIUS_MULT: 1.15,
    EASY_PULL_SPEED: 210,
    EASY_FALL_TOLERANCE_MULT: 0.6,

    TIGHT_PULL_RADIUS_MULT: 0.9,
    TIGHT_PULL_SPEED: 75,
    TIGHT_BASE_TOLERANCE_MULT: 0.17,   // 角ばった物の基準許容
    TIGHT_TOLERANCE_ROUND_BONUS: 0.18, // 丸い物ほど許容が広がる(最大 0.35)

    SPEED_MIN: 60,   // 小さい穴の速度(遅め)
    SPEED_MAX: 230,  // 最大に近い穴の速度(速め)
  };
  const WORLD_SIZE = CONFIG.WORLD_SIZE;
  const INIT_R = 15;
  const MAX_R = 400;
  const GAME_TIME = 90;

  const COLORS = {
    bg: '#e8ecef',
    road: '#cfd4d8',
    player: '#00d2d3',
    bots: ['#ff9f43', '#ee5253', '#5f27cd', '#10ac84', '#ff6b81'],
  };

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
  let curZoom = 1.5;

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

  function spawnObject(x, y) {
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

    const t = TYPES.find(tt => tt.id === type);
    const obj = {
      type: t, x, y,
      falling: false, fallT: 0, eater: null, startX: 0, startY: 0,
      seed: Math.random() * 1000, strain: 0, _wasStrained: false
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
    objects.push(obj);
  }

  function initWorld() {
    objects = [];
    const GRID = CONFIG.OBJECT_GRID;
    for (let x = GRID/2; x < WORLD_SIZE; x += GRID) {
      for (let y = GRID/2; y < WORLD_SIZE; y += GRID) {
        if (Math.random() < CONFIG.OBJECT_SKIP_CHANCE) continue;
        const rx = x + (Math.random() - 0.5) * GRID * 0.8;
        const ry = y + (Math.random() - 0.5) * GRID * 0.8;
        spawnObject(rx, ry);
      }
    }
  }

  function initEntities() {
    entities = [];
    entities.push({
      isPlayer: true, name: 'YOU', color: COLORS.player,
      x: WORLD_SIZE/2, y: WORLD_SIZE/2, r: INIT_R,
      vx: 0, vy: 0, score: 0, respawnT: 0
    });
    for (let i = 0; i < CONFIG.NUM_BOTS; i++) {
      entities.push({
        isPlayer: false, name: 'CPU ' + (i+1), color: COLORS.bots[i % COLORS.bots.length],
        x: Math.random() * WORLD_SIZE, y: Math.random() * WORLD_SIZE, r: INIT_R,
        vx: 0, vy: 0, score: 0, respawnT: 0, wanderAngle: Math.random() * Math.PI * 2
      });
    }
    camX = entities[0].x;
    camY = entities[0].y;
    curZoom = 1.5;
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
          if (o.falling || bot.r <= o.type.r * CONFIG.EAT_MIN_RATIO) continue;
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

    // 当たり判定
    for (const e of entities) {
      if (e.respawnT > 0) continue;

      for (const o of objects) {
        if (o.falling) continue;
        const round = o.type.round;
        const ratio = e.r / o.type.r;
        const dist = Math.hypot(e.x - o.x, e.y - o.y);

        if (ratio < CONFIG.EAT_MIN_RATIO) {
          o.strain = Math.max(0, o.strain - dt * 2.5);
        } else {
          // 丸い物ほど「余裕判定」に入りやすい(丸は綺麗に吸い込める)
          const easyRatio = CONFIG.EAT_EASY_RATIO - (round - 0.5) * 0.3;

          if (ratio >= easyRatio) {
            const pullR = e.r * CONFIG.EASY_PULL_RADIUS_MULT;
            if (dist < pullR) {
              pullObjectToward(o, e, dt, CONFIG.EASY_PULL_SPEED);
              if (dist < e.r * CONFIG.EASY_FALL_TOLERANCE_MULT) startFall(o, e, false);
            }
            o.strain = Math.max(0, o.strain - dt * 2.5);
          } else {
            // ギリギリサイズ:丸いほど許容が広く、角ばっているほどシビア(=引っかかる)
            const tolerance = CONFIG.TIGHT_BASE_TOLERANCE_MULT + CONFIG.TIGHT_TOLERANCE_ROUND_BONUS * round;
            const pullR = e.r * CONFIG.TIGHT_PULL_RADIUS_MULT;
            if (dist < pullR) {
              pullObjectToward(o, e, dt, CONFIG.TIGHT_PULL_SPEED);
              o.strain = Math.min(1, o.strain + dt * 2.2);
              if (dist < e.r * tolerance) startFall(o, e, true);
            } else {
              o.strain = Math.max(0, o.strain - dt * 2.5);
            }
          }
        }

        // 角ばった物を掴みかけて逃した時の「引っかかりジョルト」演出
        if (o._wasStrained && o.strain <= 0.05 && round < 0.5 && !o.falling) {
          const dx = e.x - o.x, dy = e.y - o.y;
          const d = Math.hypot(dx, dy) || 1;
          e.x -= (dx / d) * 10;
          e.y -= (dy / d) * 10;
        }
        o._wasStrained = o.strain > 0.4;
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
          addArea(o.eater, o.type.pts * 25 * (o.tight ? 1.2 : 1));
          createPopParticles(o.x, o.y, o.eater.color, o.tight ? 26 : 15);
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
      const targetZoom = Math.max(0.16, 1.8 / Math.pow(p.r / INIT_R, 0.6));
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

  function getTopPos(sx, sy, h) {
    const cx = W / 2; const cy = H / 2;
    const perspective = 0.0018;
    return {
      tx: sx + (sx - cx) * (h * perspective),
      ty: sy + (sy - cy) * (h * perspective)
    };
  }

  function drawQuad(p1, p2, p3, p4, color) {
    ctx.fillStyle = color;
    ctx.beginPath(); ctx.moveTo(p1.x, p1.y); ctx.lineTo(p2.x, p2.y);
    ctx.lineTo(p3.x, p3.y); ctx.lineTo(p4.x, p4.y); ctx.closePath();
    ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.08)'; ctx.lineWidth = 1; ctx.stroke();
  }

  function drawBox(sx, sy, w, h, col, scale, fallT) {
    const size = w * curZoom * scale * 0.5;
    const actualH = h * curZoom * scale;
    const { tx, ty } = getTopPos(sx, sy, actualH);

    const b1 = {x: sx - size, y: sy - size}; const b2 = {x: sx + size, y: sy - size};
    const b3 = {x: sx + size, y: sy + size}; const b4 = {x: sx - size, y: sy + size};
    const t1 = {x: tx - size, y: ty - size}; const t2 = {x: tx + size, y: ty - size};
    const t3 = {x: tx + size, y: ty + size}; const t4 = {x: tx - size, y: ty + size};

    const cSide1 = shade(col, -0.15); const cSide2 = shade(col, -0.3);

    ctx.globalAlpha = 1 - fallT * 0.6;

    const dx = tx - sx, dy = ty - sy;
    if (dy < 0) drawQuad(b4, b3, t3, t4, cSide1); else drawQuad(b1, b2, t2, t1, cSide1);
    if (dx < 0) drawQuad(b2, b3, t3, t2, cSide2); else drawQuad(b1, b4, t4, t1, cSide2);
    drawQuad(t1, t2, t3, t4, col);

    ctx.globalAlpha = 1.0;
  }

  function drawCylinder(sx, sy, r, h, col, scale, fallT) {
    const size = r * curZoom * scale;
    const actualH = h * curZoom * scale;
    const { tx, ty } = getTopPos(sx, sy, actualH);

    ctx.globalAlpha = 1 - fallT * 0.6;

    const angle = Math.atan2(ty - sy, tx - sx);
    const p1 = { x: sx + Math.cos(angle + Math.PI/2) * size, y: sy + Math.sin(angle + Math.PI/2) * size };
    const p2 = { x: sx + Math.cos(angle - Math.PI/2) * size, y: sy + Math.sin(angle - Math.PI/2) * size };
    const p3 = { x: tx + Math.cos(angle - Math.PI/2) * size, y: ty + Math.sin(angle - Math.PI/2) * size };
    const p4 = { x: tx + Math.cos(angle + Math.PI/2) * size, y: ty + Math.sin(angle + Math.PI/2) * size };

    drawQuad(p1, p2, p3, p4, shade(col, -0.2));

    ctx.fillStyle = col;
    ctx.beginPath(); ctx.arc(tx, ty, size, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.1)'; ctx.stroke();

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
  function drawComplex(sx, sy, t, scale, fallT) {
    drawBox(sx, sy, t.r * 1.5, t.h, t.col, scale, fallT);
    const off = t.r * 0.85 * curZoom * scale;
    drawBox(sx + off, sy + off * 0.4, t.r * 0.85, t.h * 0.55, shade(t.col, -0.12), scale, fallT);
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

  function draw() {
    ctx.fillStyle = COLORS.bg;
    ctx.fillRect(0, 0, W, H);

    function toSX(x) { return (x - camX) * curZoom + W/2; }
    function toSY(y) { return (y - camY) * curZoom + H/2; }

    const gridSize = 250 * curZoom;
    const offX = toSX(0) % gridSize;
    const offY = toSY(0) % gridSize;
    ctx.strokeStyle = COLORS.road;
    ctx.lineWidth = 30 * curZoom;
    ctx.lineCap = 'square';
    for (let x = offX - gridSize; x < W + gridSize; x += gridSize) {
      ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke();
    }
    for (let y = offY - gridSize; y < H + gridSize; y += gridSize) {
      ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke();
    }

    for (const e of entities) {
      if (e.respawnT > 0) continue;
      const sx = toSX(e.x); const sy = toSY(e.y);
      const rad = e.r * curZoom;

      ctx.fillStyle = e.color;
      ctx.beginPath(); ctx.arc(sx, sy, rad + 4 * curZoom, 0, Math.PI*2); ctx.fill();

      const grad = ctx.createRadialGradient(sx, sy, rad*0.3, sx, sy, rad);
      grad.addColorStop(0, '#0a0e17');
      grad.addColorStop(1, '#1c2836');
      ctx.fillStyle = grad;
      ctx.beginPath(); ctx.arc(sx, sy, rad, 0, Math.PI*2); ctx.fill();

      ctx.strokeStyle = 'rgba(0,0,0,0.6)'; ctx.lineWidth = Math.max(2, rad*0.1);
      ctx.beginPath(); ctx.arc(sx, sy, rad, 0, Math.PI*2); ctx.stroke();

      if (!e.isPlayer) {
        ctx.fillStyle = 'rgba(0,0,0,0.5)';
        ctx.font = `bold ${12}px sans-serif`; ctx.textAlign = 'center';
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
      if (o.falling && o.eater) {
        const ease = o.fallT * o.fallT;
        curX = o.startX + (o.eater.x - o.startX) * ease;
        curY = o.startY + (o.eater.y - o.startY) * ease;
        scale = 1 - Math.pow(o.fallT, 2);
        if (scale <= 0.05) continue;
      }

      let sx = toSX(curX); let sy = toSY(curY);
      const checkR = o.type.h * curZoom * 2;
      if (sx < -checkR || sx > W + checkR || sy < -checkR || sy > H + checkR) continue;

      if (!o.falling && o.strain > 0) {
        sx += Math.sin(performance.now() * 0.025 + o.seed) * o.strain * 3 * curZoom;
      }

      const squeezing = o.falling && o.tight;
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
