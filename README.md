# void-eater
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>Commercial Quality Neon Engine</title>
<style>
  /* 1. 操作競合の防止とフルスクリーン化 */
  :root {
    --ui-font: 'Arial Black', sans-serif;
    --ui-color: #0ff;
  }
  body, html {
    margin: 0; padding: 0; width: 100%; height: 100%;
    background-color: #050510;
    overflow: hidden;
    overscroll-behavior: none; /* 下スワイプ更新を無効化 */
    position: fixed;
    touch-action: none;
    font-family: var(--ui-font);
    user-select: none;
  }
  #gameCanvas {
    display: block;
    width: 100%; height: 100%;
  }
  
  /* 2. UIのDOM化（CanvasではなくHTMLで描画してクッキリさせる） */
  #ui-layer {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%;
    pointer-events: none; /* クリックをCanvasに貫通させる */
  }
  #score-board {
    position: absolute; top: 20px; left: 20px;
    color: #fff; font-size: 24px; font-weight: bold;
    text-shadow: 0 0 10px var(--ui-color), 0 0 20px var(--ui-color);
  }
  
  /* フローティングダメージテキスト用 */
  .floating-text {
    position: absolute;
    color: #ff0055;
    font-size: 28px; font-weight: bold;
    text-shadow: 0 0 10px #ff0055, 0 0 20px #fff;
    animation: floatUp 0.8s ease-out forwards;
  }
  @keyframes floatUp {
    0% { transform: translateY(0) scale(0.5); opacity: 1; }
    50% { transform: translateY(-40px) scale(1.2); opacity: 1; }
    100% { transform: translateY(-80px) scale(1); opacity: 0; }
  }
</style>
</head>
<body>

<!-- ゲーム画面 -->
<canvas id="gameCanvas"></canvas>

<!-- UIオーバーレイ -->
<div id="ui-layer">
  <div id="score-board">SCORE: <span id="score-val">0</span></div>
</div>

<script>
/**
 * 3. パフォーマンスとビジュアルを両立する描画エンジン
 */
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d', { alpha: false }); // alpha:falseで描画最適化
let width, height;

// 画面サイズのリサイズ対応
function resize() {
    width = canvas.width = window.innerWidth;
    height = canvas.height = window.innerHeight;
}
window.addEventListener('resize', resize);
resize();

let score = 0;
const scoreVal = document.getElementById('score-val');
const uiLayer = document.getElementById('ui-layer');

// ---- A. オフスクリーンキャンバスによる背景事前レンダリング ----
// ビル群を毎フレーム描くのではなく、1枚の「画像」として生成しておく（超軽量化）
function generateCityscape(color, windowColor, scale, speed, hasWindows) {
    const oc = document.createElement('canvas');
    oc.width = 1200; oc.height = 800; // 仮想サイズ
    const octx = oc.getContext('2d');
    
    // ビルのシルエットを描画
    let x = 0;
    while (x < oc.width) {
        const bWidth = 40 + Math.random() * 80;
        const bHeight = 100 + Math.random() * 400 * scale;
        octx.fillStyle = color;
        octx.fillRect(x, oc.height - bHeight, bWidth, bHeight);
        
        // 窓の描画（重い処理だが、初回1回しか走らないので問題なし）
        if (hasWindows) {
            octx.fillStyle = windowColor;
            for (let wy = oc.height - bHeight + 10; wy < oc.height - 10; wy += 20) {
                for (let wx = x + 10; wx < x + bWidth - 10; wx += 15) {
                    if (Math.random() > 0.3) { // 70%の確率で窓が点灯
                        octx.fillRect(wx, wy, 8, 12);
                    }
                }
            }
        }
        x += bWidth + 5;
    }
    return { canvas: oc, speed: speed, x: 0 };
}

// 遠景（暗い、遅い、窓なし）と 中景（明るい、速い、窓あり）を作成
const bgLayers = [
    generateCityscape('#1a1a2e', '', 1.2, 0.5, false),
    generateCityscape('#0f3460', '#e94560', 0.8, 1.5, true)
];


// ---- B. パーティクルシステム（shadowBlurを使わない発光） ----
let particles = [];
class Particle {
    constructor(x, y) {
        this.x = x; this.y = y;
        // 爆発のように四方八方に散らす
        const angle = Math.random() * Math.PI * 2;
        const speed = Math.random() * 10 + 5;
        this.vx = Math.cos(angle) * speed;
        this.vy = Math.sin(angle) * speed;
        this.life = 1.0;
        this.decay = Math.random() * 0.02 + 0.01;
        this.size = Math.random() * 4 + 2;
        
        // ランダムなネオンカラー
        const colors = ['#0ff', '#f0f', '#fa0'];
        this.color = colors[Math.floor(Math.random() * colors.length)];
    }
    update() {
        this.vy += 0.3; // 重力
        this.x += this.vx;
        this.y += this.vy;
        this.life -= this.decay;
    }
    draw(ctx) {
        // 残像のようなスピード線を引く（プロのテクニック）
        ctx.beginPath();
        ctx.moveTo(this.x - this.vx * 2, this.y - this.vy * 2);
        ctx.lineTo(this.x, this.y);
        ctx.strokeStyle = this.color;
        ctx.lineWidth = this.size * this.life;
        ctx.lineCap = 'round';
        ctx.globalAlpha = this.life; // 消えゆくフェードアウト
        ctx.stroke();
    }
}


// ---- C. インタラクション（DOMによるスコア表示） ----
function spawnHitEffect(x, y) {
    // 1. パーティクル発生
    for (let i = 0; i < 20; i++) particles.push(new Particle(x, y));
    
    // 2. スコア加算
    score += 100;
    scoreVal.innerText = score;

    // 3. フローティングテキスト（HTML/CSSによるクッキリしたアニメーション）
    const floatText = document.createElement('div');
    floatText.className = 'floating-text';
    floatText.innerText = '+100';
    floatText.style.left = (x - 20) + 'px';
    floatText.style.top = (y - 30) + 'px';
    uiLayer.appendChild(floatText);
    
    // アニメーション終了後にDOMから削除
    setTimeout(() => { floatText.remove(); }, 800);
}

// スマホ・PC両対応の入力イベント（デフォルト動作のキャンセル含む）
canvas.addEventListener('pointerdown', (e) => {
    e.preventDefault(); // スワイプ等の防止
    spawnHitEffect(e.clientX, e.clientY);
});


// ---- D. メインループ ----
function render() {
    // 1. 背景のクリア（深みのある夜空）
    ctx.globalCompositeOperation = 'source-over';
    ctx.globalAlpha = 1.0;
    const grad = ctx.createLinearGradient(0, 0, 0, height);
    grad.addColorStop(0, '#050510');
    grad.addColorStop(1, '#1a0b2e');
    ctx.fillStyle = grad;
    ctx.fillRect(0, 0, width, height);

    // 2. パララックス（視差）背景の描画
    bgLayers.forEach(layer => {
        layer.x -= layer.speed;
        if (layer.x <= -layer.canvas.width) layer.x = 0;
        
        // 高さの中央付近から描画する計算
        const yPos = height - layer.canvas.height * 0.8; 
        
        // 無限スクロールさせるために2枚並べて描画
        ctx.drawImage(layer.canvas, layer.x, yPos);
        ctx.drawImage(layer.canvas, layer.x + layer.canvas.width, yPos);
    });

    // 3. パーティクルの描画（加算合成による美しい発光）
    ctx.globalCompositeOperation = 'lighter'; // 光が重なると白く飛ぶ
    
    for (let i = particles.length - 1; i >= 0; i--) {
        const p = particles[i];
        p.update();
        p.draw(ctx);
        if (p.life <= 0 || p.y > height) {
            particles.splice(i, 1);
        }
    }

    requestAnimationFrame(render);
}
render();
</script>
</body>
</html>
