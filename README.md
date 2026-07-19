# void-eater
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>CITY HOLE - Full Version</title>
<style>
  body { margin:0; background:#87ceeb; overflow:hidden; touch-action:none; font-family:sans-serif; }
  canvas { display:block; }
  #ui { position:absolute; top:20px; left:20px; color:white; font-size:24px; font-weight:900; pointer-events:none; }
</style>
</head>
<body>
<div id="ui">SCORE: <span id="score">0</span></div>
<canvas id="game"></canvas>

<script>
const CONFIG = { WORLD: 4000, OBJ_NUM: 800, CPU_NUM: 3 };
const TYPES = [
  { id:'prop', r:5, pts:1, col:'#f1c40f', move:0 },
  { id:'tree', r:12, pts:5, col:'#2ecc71', move:0 },
  { id:'car', r:18, pts:10, col:'#3498db', move:2 },
  { id:'spinner', r:25, pts:20, col:'#e67e22', move:1 },
  { id:'house', r:40, pts:50, col:'#e74c3c', move:0 }
];

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
let W, H, p = { x:2000, y:2000, r:20, score:0, vx:0, vy:0 }, objs = [], cpus = [];

function setup() {
  W = canvas.width = window.innerWidth; H = canvas.height = window.innerHeight;
  for(let i=0; i<CONFIG.OBJ_NUM; i++) {
    const t = TYPES[Math.floor(Math.random()*TYPES.length)];
    objs.push({ type:t, x:Math.random()*CONFIG.WORLD, y:Math.random()*CONFIG.WORLD, ang:0, fall:0 });
  }
  for(let i=0; i<CONFIG.CPU_NUM; i++) {
    cpus.push({ x:Math.random()*CONFIG.WORLD, y:Math.random()*CONFIG.WORLD, r:20, vx:0, vy:0 });
  }
}

let pointer = { x:0, y:0, down:false };
window.onpointerdown = e => { pointer.down=true; pointer.x=e.clientX; pointer.y=e.clientY; };
window.onpointermove = e => { if(pointer.down) { pointer.x=e.clientX; pointer.y=e.clientY; } };
window.onpointerup = () => pointer.down=false;

function loop() {
  // Move Player
  if(pointer.down) {
    const dx = pointer.x - W/2, dy = pointer.y - H/2;
    const dist = Math.hypot(dx,dy);
    const speed = Math.max(50, 300 - p.r*0.2);
    p.vx += (dx/dist*speed - p.vx)*0.1; p.vy += (dy/dist*speed - p.vy)*0.1;
  } else { p.vx*=0.8; p.vy*=0.8; }
  p.x += p.vx; p.y += p.vy;

  // Interactions
  objs.forEach(o => {
    const d = Math.hypot(p.x-o.x, p.y-o.y);
    if(o.type.r < p.r * 0.9 && d < p.r * 0.7) {
      o.fall += 0.05;
      if(o.fall >= 1) { p.score += o.type.pts; p.r += 0.5; o.fall=0; o.x=Math.random()*CONFIG.WORLD; o.y=Math.random()*CONFIG.WORLD; }
    } else o.fall = Math.max(0, o.fall - 0.02);
    if(o.type.move === 1) o.ang += 0.1;
  });

  // Draw
  ctx.clearRect(0,0,W,H);
  const zoom = Math.max(0.3, 1.2 - p.r/300);
  ctx.save();
  ctx.translate(W/2 - p.x*zoom, H/2 - p.y*zoom);
  ctx.scale(zoom, zoom);

  objs.forEach(o => {
    ctx.fillStyle = o.type.col;
    ctx.save(); ctx.translate(o.x, o.y); ctx.rotate(o.ang);
    const s = 1 - o.fall*0.5;
    ctx.fillRect(-o.type.r*s, -o.type.r*s, o.type.r*2*s, o.type.r*2*s);
    ctx.restore();
  });

  ctx.fillStyle = 'black';
  ctx.beginPath(); ctx.arc(p.x, p.y, p.r, 0, Math.PI*2); ctx.fill();
  
  ctx.restore();
  document.getElementById('score').textContent = Math.floor(p.score);
  requestAnimationFrame(loop);
}

setup();
loop();
</script>
</body>
</html>
