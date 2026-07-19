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

  /* UI Layer */
  #ui-layer {
    position: fixed; inset: 0; pointer-events: none; z-index: 10;
  }
  
  /* HUD */
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

  /* Leaderboard */
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

  /* Screens */
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
  <div class="panel" id="screen-content">
    <!-- コンテンツはJSで動的に挿入 -->
  </div>
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

  // --- 定数 & 設定 ---
  const WORLD_SIZE = 3500;
  const INIT_R = 15;
  const MAX_R = 400;
  const GAME_TIME = 90;

  const COLORS = {
    bg: '#e8ecef',        // 街の地面
    road: '#cfd4d8',      // 道路
    player: '#00d2d3',    // プレイヤー色（シアン）
    bots: ['#ff9f43', '#ee5253', '#5f27cd', '#10ac84', '#ff6b81'], // ライバル色
  };

  // 街のオブジェクト定義
  const TYPES = [
    { id: 'person', r: 3, h: 8, pts: 1, col: '#f1c40f' },
    { id: 'cone', r: 4, h: 6, pts: 2, col: '#e67e22' },
    { id: 'bench', r: 8, h: 5, pts: 5, col: '#9b59b6' },
    { id: 'tree', r: 12, h: 25, pts: 10, col: '#2ecc71' },
    { id: 'car', r: 16, h: 12, pts: 15, col: '#3498db' },
    { id: 'house', r: 30, h: 35, pts: 40, col: '#e74c3c' },
    { id: 'building', r: 55, h: 90, pts: 100, col: '#95a5a6' },
    { id: 'skyscraper', r: 90, h: 200, pts: 250, col: '#34495e' }
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

  // --- 操作関連 ---
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
      // ジョイスティックが指から離れすぎないように追従させる
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

  // --- 初期化ロジック ---
  function player() { return entities[0]; }

  function spawnObject(x, y) {
    const roll = Math.random();
    let type;
    if (roll < 0.3) type = 'person';
    else if (roll < 0.55) type = 'cone';
    else if (roll < 0.75) type = 'tree';
    else if (roll < 0.88) type = 'car';
    else if (roll < 0.95) type = 'house';
    else if (roll < 0.98) type = 'building';
    else type = 'skyscraper';

    const t = TYPES.find(t => t.id === type);
    objects.push({
      type: t, x, y,
      falling: false, fallT: 0, eater: null, startX: 0, startY: 0
    });
  }

  function initWorld() {
    objects = [];
    const GRID = 150;
    for (let x = GRID/2; x < WORLD_SIZE; x += GRID) {
      for (let y = GRID/2; y < WORLD_SIZE; y += GRID) {
        if (Math.random() < 0.15) continue; // 少し空き地を作る
        // グリッド内で位置をランダムにばらけさせる
        const rx = x + (Math.random() - 0.5) * GRID * 0.8;
        const ry = y + (Math.random() - 0.5) * GRID * 0.8;
        spawnObject(rx, ry);
      }
    }
  }

  function initEntities() {
    entities = [];
    // プレイヤー
    entities.push({
      isPlayer: true, name: 'YOU', color: COLORS.player,
      x: WORLD_SIZE/2, y: WORLD_SIZE/2, r: INIT_R,
      vx: 0, vy: 0, score: 0, respawnT: 0
    });
    // CPU
    for (let i = 0; i < 5; i++) {
      entities.push({
        isPlayer: false, name: 'CPU ' + (i+1), color: COLORS.bots[i],
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

  // --- 更新ロジック ---
  function update(dt) {
    const p = player();

    // プレイヤー移動
    if (p.respawnT > 0) {
      p.respawnT -= dt;
      if (p.respawnT <= 0) {
        p.x = Math.random() * WORLD_SIZE; p.y = Math.random() * WORLD_SIZE;
        p.r = INIT_R; p.score = Math.floor(p.score / 2);
      }
    } else {
      const speed = Math.max(90, 260 - p.r * 0.4);
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

    // CPU移動
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

      const speed = Math.max(90, 260 - bot.r * 0.4);
      let target = null;
      let flee = false;

      // 逃避行動
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

      // 捕食行動
      if (!flee) {
        let minDist = Infinity;
        for (const o of objects) {
          if (o.falling || bot.r <= o.type.r * 1.25) continue;
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

    // 当たり判定
    for (const e of entities) {
      if (e.respawnT > 0) continue;

      // オブジェクトを食べる
      for (const o of objects) {
        if (o.falling) continue;
        if (e.r > o.type.r * 1.25) {
          const dist = Math.hypot(e.x - o.x, e.y - o.y);
          if (dist < e.r * 0.85) { // 穴の少し内側に入ったら落下開始
            o.falling = true;
            o.eater = e;
            o.startX = o.x; o.startY = o.y;
          }
        }
      }

      // 他プレイヤーを食べる
      for (const other of entities) {
        if (other === e || other.respawnT > 0) continue;
        if (e.r > other.r * 1.25) {
          const dist = Math.hypot(e.x - other.x, e.y - other.y);
          if (dist < e.r * 0.8) {
            e.score += Math.floor(other.score * 0.5) + 50;
            addArea(e, other.r * other.r * Math.PI * 0.6);
            other.respawnT = 3.0; // 3秒でリスポーン
            createPopParticles(other.x, other.y, other.color);
          }
        }
      }
    }

    // 落下アニメーション処理
    for (let i = objects.length - 1; i >= 0; i--) {
      const o = objects[i];
      if (o.falling) {
        o.fallT += dt * 3.5; // 約0.28秒で完全に落ちる
        if (o.fallT >= 1) {
          o.eater.score += o.type.pts;
          // スコアに比例して面積を増やす
          addArea(o.eater, o.type.pts * 25);
          objects.splice(i, 1);
          // 枯渇を防ぐため、マップのどこかに新しく補充
          spawnObject(Math.random() * WORLD_SIZE, Math.random() * WORLD_SIZE);
        }
      }
    }

    // パーティクル更新
    for (let i = particles.length - 1; i >= 0; i--) {
      const pt = particles[i];
      pt.x += pt.vx * dt; pt.y += pt.vy * dt;
      pt.life -= dt;
      if (pt.life <= 0) particles.splice(i, 1);
    }

    // カメラのスムーズな追従
    if (p.respawnT <= 0) {
      camX += (p.x - camX) * dt * 4.0;
      camY += (p.y - camY) * dt * 4.0;
      const targetZoom = Math.max(0.18, 1.8 / Math.pow(p.r / INIT_R, 0.6));
      curZoom += (targetZoom - curZoom) * dt * 3.0;
    }

    updateUI();
  }

  function createPopParticles(x, y, color) {
    for (let i = 0; i < 15; i++) {
      const angle = Math.random() * Math.PI * 2;
      const spd = 50 + Math.random() * 150;
      particles.push({
        x, y, vx: Math.cos(angle)*spd, vy: Math.sin(angle)*spd,
        life: 0.6, maxLife: 0.6, color, size: 3 + Math.random()*4
      });
    }
  }

  // --- 描画ユーティリティ ---
  function shade(color, percent) {
    let f = parseInt(color.slice(1), 16), t = percent < 0 ? 0 : 255, p = percent < 0 ? percent * -1 : percent;
    let R = f >> 16, G = f >> 8 & 0x00FF, B = f & 0x0000FF;
    return "#" + (0x1000000 + (Math.round((t - R) * p) + R) * 0x10000 + (Math.round((t - G) * p) + G) * 0x100 + (Math.round((t - B) * p) + B)).toString(16).slice(1);
  }

  // 画面中心を消失点としたフェイク3D（パララックス）の上面座標を計算
  function getTopPos(sx, sy, h) {
    const cx = W / 2; const cy = H / 2;
    // hが大きいほど外側に傾いて見える
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

  // 直方体の描画
  function drawBox(sx, sy, w, h, col, scale, fallT) {
    const size = w * curZoom * scale * 0.5;
    const actualH = h * curZoom * scale;
    const { tx, ty } = getTopPos(sx, sy, actualH);

    const b1 = {x: sx - size, y: sy - size}; const b2 = {x: sx + size, y: sy - size};
    const b3 = {x: sx + size, y: sy + size}; const b4 = {x: sx - size, y: sy + size};
    const t1 = {x: tx - size, y: ty - size}; const t2 = {x: tx + size, y: ty - size};
    const t3 = {x: tx + size, y: ty + size}; const t4 = {x: tx - size, y: ty + size};

    const cSide1 = shade(col, -0.15); const cSide2 = shade(col, -0.3);
    
    // 吸い込まれるときは暗くフェードする
    ctx.globalAlpha = 1 - fallT * 0.6;

    const dx = tx - sx, dy = ty - sy;
    if (dy < 0) drawQuad(b4, b3, t3, t4, cSide1); else drawQuad(b1, b2, t2, t1, cSide1);
    if (dx < 0) drawQuad(b2, b3, t3, t2, cSide2); else drawQuad(b1, b4, t4, t1, cSide2);
    drawQuad(t1, t2, t3, t4, col);
    
    ctx.globalAlpha = 1.0;
  }

  // 円柱の描画
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
  }

  // --- メイン描画 ---
  function draw() {
    ctx.fillStyle = COLORS.bg;
    ctx.fillRect(0, 0, W, H);

    function toSX(x) { return (x - camX) * curZoom + W/2; }
    function toSY(y) { return (y - camY) * curZoom + H/2; }

    // 地面の道路グリッドを描画
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

    // 穴の描画（一番下）
    for (const e of entities) {
      if (e.respawnT > 0) continue;
      const sx = toSX(e.x); const sy = toSY(e.y);
      const rad = e.r * curZoom;

      // 枠線
      ctx.fillStyle = e.color;
      ctx.beginPath(); ctx.arc(sx, sy, rad + 4 * curZoom, 0, Math.PI*2); ctx.fill();
      
      // 穴の闇
      const grad = ctx.createRadialGradient(sx, sy, rad*0.3, sx, sy, rad);
      grad.addColorStop(0, '#0a0e17');
      grad.addColorStop(1, '#1c2836');
      ctx.fillStyle = grad;
      ctx.beginPath(); ctx.arc(sx, sy, rad, 0, Math.PI*2); ctx.fill();

      // 内側の影で深さを出す
      ctx.strokeStyle = 'rgba(0,0,0,0.6)'; ctx.lineWidth = Math.max(2, rad*0.1);
      ctx.beginPath(); ctx.arc(sx, sy, rad, 0, Math.PI*2); ctx.stroke();
      
      // 名前表示 (CPU)
      if (!e.isPlayer) {
        ctx.fillStyle = 'rgba(0,0,0,0.5)';
        ctx.font = `bold ${12}px sans-serif`; ctx.textAlign = 'center';
        ctx.fillText(e.name, sx, sy - rad - 15);
      }
    }

    // オブジェクトの描画（Y座標でソートして立体交差を正しくする）
    let drawList = [...objects];
    // パーティクルもソートに混ぜる
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

      // 落下中の位置計算
      let curX = o.x, curY = o.y, scale = 1;
      if (o.falling && o.eater) {
        // 吸い込まれる動き（イージング）
        const ease = o.fallT * o.fallT; 
        curX = o.startX + (o.eater.x - o.startX) * ease;
        curY = o.startY + (o.eater.y - o.startY) * ease;
        scale = 1 - Math.pow(o.fallT, 2); // 落下するにつれて小さくなる
        if(scale <= 0.05) continue;
      }

      const sx = toSX(curX); const sy = toSY(curY);
      // カリング（画面外は描かない）
      const checkR = o.type.h * curZoom * 2;
      if (sx < -checkR || sx > W + checkR || sy < -checkR || sy > H + checkR) continue;

      // 種別ごとの描画
      const t = o.type;
      if (t.id === 'person') {
        drawCylinder(sx, sy, t.r, t.h, t.col, scale, o.fallT);
        drawCylinder(sx, sy, t.r*0.8, t.h+3, '#ffddaa', scale, o.fallT); // 頭
      } else if (t.id === 'cone') {
        drawCylinder(sx, sy, t.r, t.h, t.col, scale, o.fallT);
      } else if (t.id === 'tree') {
        drawCylinder(sx, sy, t.r*0.3, t.h*0.4, '#a0522d', scale, o.fallT);
        drawBox(sx, sy, t.r*1.8, t.h, t.col, scale, o.fallT);
      } else if (t.id === 'car') {
        drawBox(sx, sy, t.r*1.5, t.h*0.5, t.col, scale, o.fallT);
        drawBox(sx, sy, t.r*0.8, t.h, '#ecf0f1', scale, o.fallT);
      } else {
        // bench, house, building, skyscraper
        drawBox(sx, sy, t.r*1.6, t.h, t.col, scale, o.fallT);
      }
    }

    // 操作UI（ジョイスティック）の描画
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

  // --- メインループ ---
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

  // --- ゲーム進行制御 ---
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

  // 初回起動
  showTitle();
  requestAnimationFrame(loop);

})();
</script>
</body>
</html>
