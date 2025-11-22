<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Habul-Habulan — Filipino Chase</title>
<style>
  :root{--bg:#fdeecf;--panel:#fff9f4;--accent:#ff6b35;--muted:#6b6b6b}
  html,body{height:100%;margin:0;font-family:Inter, system-ui, -apple-system, 'Segoe UI', Roboto, Arial;background:var(--bg);color:#222}
  .wrap{max-width:980px;margin:18px auto;padding:14px}
  header{display:flex;justify-content:space-between;align-items:center}
  header h1{font-size:20px;margin:0}
  .controls{display:flex;gap:8px;align-items:center}
  button{background:var(--accent);color:white;border:0;padding:8px 12px;border-radius:8px;cursor:pointer;font-weight:700}
  button.secondary{background:#fff;border:1px solid #eee;color:#333}
  .game-area{margin-top:12px;background:var(--panel);padding:12px;border-radius:12px;box-shadow:0 8px 20px rgba(0,0,0,.06);display:grid;grid-template-columns:1fr 240px;gap:12px}
  canvas{background:linear-gradient(#fef6e6,#fff);width:100%;height:560px;border-radius:8px;display:block}
  .side{padding:8px}
  .stat{margin-bottom:10px}
  .muted{color:var(--muted);font-size:13px}
  .hud{display:flex;gap:8px;align-items:center}
  .hud .pill{background:#fff;padding:8px 10px;border-radius:10px;box-shadow:0 4px 12px rgba(0,0,0,.04);min-width:82px;text-align:center}
  .footer{margin-top:10px;text-align:center;color:var(--muted);font-size:13px}
  /* mobile controls */
  .touch-controls{display:none;gap:8px;margin-top:10px;justify-content:center}
  .touch-controls button{width:56px;height:56px;border-radius:10px;font-weight:800;font-size:18px}
  @media (max-width:900px){
    .game-area{grid-template-columns:1fr}
    canvas{height:520px}
    .touch-controls{display:flex}
  }
</style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>🇵🇭 Habul-Habulan</h1>
      <div class="controls">
        <div class="muted">Pinoy Mode</div>
        <button id="startBtn">Simulan</button>
        <button id="pauseBtn" class="secondary">Pahinto</button>
      </div>
    </header>

    <div class="game-area">
      <div>
        <canvas id="game" width="800" height="560"></canvas>

        <!-- touch controls for mobile -->
        <div class="touch-controls" id="touchControls">
          <button data-key="up">▲</button>
          <div style="display:flex;gap:8px;">
            <button data-key="left">◀</button>
            <button data-key="down">▼</button>
            <button data-key="right">▶</button>
          </div>
        </div>
      </div>

      <aside class="side">
        <div class="stat hud">
          <div class="pill"><div class="muted">Lives</div><div id="lives">3</div></div>
          <div class="pill"><div class="muted">Score</div><div id="score">0</div></div>
        </div>

        <div class="stat">
          <div class="muted">Panahon (Time)</div>
          <div id="time">0 s</div>
        </div>

        <div class="stat">
          <div class="muted">Mga Tip</div>
          <ul style="padding-left:16px;margin:6px 0">
            <li>Gamitin arrow keys o WASD</li>
            <li>Kolektahin ang coins para sa mataas na score</li>
            <li>Iwasan ang kalaban — mabilis siyang mag-habol</li>
          </ul>
        </div>

        <div class="stat">
          <div class="muted">Maikling Kuwento</div>
          <div style="font-size:13px">Ikaw ay tumatakbo sa kalsada ng baryo — habul-habulan kasama ang kaibigan! Kolektahin ang ₱-coins at tumagal hangga't kaya.</div>
        </div>

        <div style="margin-top:10px">
          <button id="muteBtn" class="secondary">Patunog</button>
          <button id="resetBtn" class="secondary">I-reset</button>
        </div>
      </aside>
    </div>

    <div class="footer">Made with ❤️ — Press <strong>Simulan</strong> to play. Mobile: use the on-screen arrows.</div>
  </div>

<script>
/* Habul-Habulan — simple chase game
   Controls: arrow keys / WASD / touch buttons
   Features:
   - Player moves; enemy homing AI chases player
   - Coins spawn; collecting increases score + short speed boost
   - Lives and timer
*/

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');

let W = canvas.width, H = canvas.height;
function resizeCanvas() {
  // keep logical size constant for simplicity, CSS may scale
  // update if needed
  W = canvas.width; H = canvas.height;
}
resizeCanvas();

let playing = false, paused = false;
let score = 0, lives = 3, elapsed = 0;
let lastTime = 0;

// player
const player = {
  x: W/2, y: H/2, r: 18,
  speed: 200, // px per second
  vx: 0, vy: 0
};

// enemy (chaser)
const enemy = {
  x: 80, y: 80, r: 22,
  speed: 120 // px/s base
};

// coins (collectibles)
let coins = [];

// input
const keys = {};
window.addEventListener('keydown', e => { keys[e.key.toLowerCase()] = true; if(e.key === ' ' && !playing) startGame(); });
window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

// touch buttons
document.querySelectorAll('#touchControls button').forEach(btn => {
  let key = btn.dataset.key;
  btn.addEventListener('touchstart', e => { e.preventDefault(); touchSet(key, true); });
  btn.addEventListener('touchend', e => { e.preventDefault(); touchSet(key, false); });
  btn.addEventListener('mousedown', ()=> touchSet(key, true));
  btn.addEventListener('mouseup', ()=> touchSet(key, false));
});
function touchSet(dir, on) {
  if(dir === 'up') keys['arrowup'] = on;
  if(dir === 'down') keys['arrowdown'] = on;
  if(dir === 'left') keys['arrowleft'] = on;
  if(dir === 'right') keys['arrowright'] = on;
}

// UI elements
const startBtn = document.getElementById('startBtn');
const pauseBtn = document.getElementById('pauseBtn');
const resetBtn = document.getElementById('resetBtn');
const muteBtn = document.getElementById('muteBtn');

const scoreEl = document.getElementById('score');
const livesEl = document.getElementById('lives');
const timeEl = document.getElementById('time');

startBtn.addEventListener('click', ()=>{ if(!playing) startGame(); else if(paused){ paused=false; lastTime = performance.now(); }});
pauseBtn.addEventListener('click', ()=>{ paused = !paused; pauseBtn.textContent = paused ? 'Magpatuloy' : 'Pahinto'; });
resetBtn.addEventListener('click', resetGame);
let muted = false;
muteBtn.addEventListener('click', ()=>{ muted = !muted; muteBtn.textContent = muted ? 'Patunog' : 'Patay'; });

// spawn helper
function rand(min,max){ return Math.random()*(max-min)+min; }
function dist(a,b){ return Math.hypot(a.x-b.x, a.y-b.y); }

// reset / start
function resetGame(){
  playing = false;
  paused = false;
  score = 0;
  lives = 3;
  elapsed = 0;
  coins = [];
  player.x = W/2; player.y = H/2; player.speed = 200;
  enemy.x = 80; enemy.y = 80; enemy.speed = 120;
  updateHUD();
  drawIntro();
}
function startGame(){
  playing = true;
  paused = false;
  score = 0; lives = 3; elapsed = 0;
  player.x = W/2; player.y = H/2;
  enemy.x = rand(40,W-40); enemy.y = rand(40,H-40);
  coins = [];
  spawnCoin();
  lastTime = performance.now();
  requestAnimationFrame(loop);
  updateHUD();
}

// spawn coin
function spawnCoin(){
  // limit coins
  if(coins.length >= 4) return;
  const margin = 30;
  for(let i=0;i<6;i++){
    const c = { x: rand(margin, W-margin), y: rand(margin, H-margin), r: 10 };
    // avoid spawning too close to player/enemy
    if(Math.hypot(c.x-player.x, c.y-player.y) < 80) continue;
    if(Math.hypot(c.x-enemy.x, c.y-enemy.y) < 80) continue;
    coins.push(c);
    break;
  }
}

// game loop
function loop(now){
  if(!playing) return;
  if(paused){ lastTime = now; requestAnimationFrame(loop); return; }
  const dt = Math.min(0.05, (now - lastTime) / 1000);
  lastTime = now;
  elapsed += dt;

  update(dt);
  render();

  // spawn coins occasionally
  if(Math.random() < dt * 0.5) spawnCoin();

  if(playing) requestAnimationFrame(loop);
}

// update physics & AI
function update(dt){
  // player input
  let ax = 0, ay = 0;
  if(keys['arrowup'] || keys['w']) ay -= 1;
  if(keys['arrowdown'] || keys['s']) ay += 1;
  if(keys['arrowleft'] || keys['a']) ax -= 1;
  if(keys['arrowright'] || keys['d']) ax += 1;

  // normalize
  if(ax !== 0 || ay !== 0){
    const len = Math.hypot(ax, ay);
    ax /= len; ay /= len;
  }

  player.vx = ax * player.speed;
  player.vy = ay * player.speed;
  player.x += player.vx * dt;
  player.y += player.vy * dt;

  // keep inside bounds
  player.x = Math.max(player.r, Math.min(W - player.r, player.x));
  player.y = Math.max(player.r, Math.min(H - player.r, player.y));

  // enemy AI: homing with slight randomness
  const dx = player.x - enemy.x;
  const dy = player.y - enemy.y;
  const ang = Math.atan2(dy, dx);
  const pursueSpeed = enemy.speed + Math.min(1.2, elapsed/30) * 40; // gets slowly faster
  enemy.x += Math.cos(ang) * pursueSpeed * dt;
  enemy.y += Math.sin(ang) * pursueSpeed * dt;

  // enemy boundary clamp
  enemy.x = Math.max(enemy.r, Math.min(W - enemy.r, enemy.x));
  enemy.y = Math.max(enemy.r, Math.min(H - enemy.r, enemy.y));

  // check collisions: player <> coins
  for(let i = coins.length - 1; i >= 0; i--){
    const c = coins[i];
    if(Math.hypot(player.x - c.x, player.y - c.y) < player.r + c.r){
      coins.splice(i,1);
      onCollectCoin();
    }
  }

  // check collision: player <> enemy
  if(Math.hypot(player.x - enemy.x, player.y - enemy.y) < player.r + enemy.r - 2){
    onPlayerHit();
  }

  updateHUD();
}

// coin collected
function onCollectCoin(){
  score += 10;
  // small speed boost
  player.speed = Math.min(360, player.speed + 30);
  // reduce boost over time
  setTimeout(()=>{ player.speed = Math.max(200, player.speed - 30); }, 1800);
  if(!muted) playBeep();
}

// player hit
function onPlayerHit(){
  lives -= 1;
  if(!muted) playHit();
  if(lives <= 0){
    // game over
    playing = false;
    showGameOver();
  } else {
    // respawn player to center, enemy to random corner
    player.x = W/2; player.y = H/2;
    enemy.x = rand(40, W-40); enemy.y = rand(40, H-40);
    // small temporary invulnerability could be added
  }
}

// HUD update
function updateHUD(){
  scoreEl.textContent = score;
  livesEl.textContent = lives;
  timeEl.textContent = Math.floor(elapsed) + ' s';
}

// drawing
function render(){
  // clear
  ctx.clearRect(0, 0, W, H);

  // background road / barrio vibes
  drawBackground();

  // coins
  for(const c of coins){
    drawCoin(c.x, c.y, c.r);
  }

  // draw player (jeepney-ish color)
  drawPlayer(player.x, player.y, player.r);

  // draw enemy (chaser)
  drawEnemy(enemy.x, enemy.y, enemy.r);

  // top-left HUD small
  ctx.fillStyle = 'rgba(0,0,0,0.06)';
  ctx.fillRect(12, 12, 160, 40);
}

// drawing helpers
function drawBackground(){
  // simple ground and lanes
  ctx.fillStyle = '#fff9f0';
  ctx.fillRect(0,0,W,H);
  // subtle road stripes
  ctx.strokeStyle = '#fde6c9';
  ctx.lineWidth = 2;
  for(let y=80; y < H; y += 120){
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(W, y);
    ctx.stroke();
  }
}

function drawPlayer(x,y,r){
  // body
  ctx.beginPath();
  ctx.fillStyle = '#2d6a4f'; // green jeepney style
  roundRect(ctx, x - r, y - r, r*2, r*1.4, 6, true, false);
  // wheel
  ctx.fillStyle = '#333';
  ctx.beginPath(); ctx.arc(x - r*0.6, y + r*0.65, r*0.28, 0, Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.arc(x + r*0.6, y + r*0.65, r*0.28, 0, Math.PI*2); ctx.fill();
  // head
  ctx.fillStyle = '#fff';
  ctx.beginPath(); ctx.arc(x + r*0.6, y - r*0.25, r*0.28, 0, Math.PI*2); ctx.fill();
}

function drawEnemy(x,y,r){
  ctx.beginPath();
  ctx.fillStyle = '#b02e0c'; // red chaser
  roundRect(ctx, x - r, y - r, r*2, r*2, 12, true, false);
  // eyes
  ctx.fillStyle = '#fff';
  ctx.beginPath(); ctx.arc(x - r*0.35, y - r*0.15, r*0.25, 0, Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.arc(x + r*0.35, y - r*0.15, r*0.25, 0, Math.PI*2); ctx.fill();
  ctx.fillStyle = '#111';
  ctx.beginPath(); ctx.arc(x - r*0.35, y - r*0.15, r*0.1, 0, Math.PI*2); ctx.fill();
  ctx.beginPath(); ctx.arc(x + r*0.35, y - r*0.15, r*0.1, 0, Math.PI*2); ctx.fill();
}

function drawCoin(x,y,r){
  ctx.save();
  ctx.beginPath();
  ctx.fillStyle = '#ffd166';
  ctx.arc(x,y,r,0,Math.PI*2);
  ctx.fill();
  ctx.fillStyle = '#b8860b';
  ctx.font = (r) + 'px sans-serif';
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.fillText('₱', x, y - 1);
  ctx.restore();
}

// rounded rect helper
function roundRect(ctx, x, y, width, height, radius, fill, stroke) {
  if (typeof stroke === 'undefined') stroke = true;
  if (typeof radius === 'undefined') radius = 5;
  if (typeof radius === 'number') radius = {tl: radius, tr: radius, br: radius, bl: radius};
  ctx.beginPath();
  ctx.moveTo(x + radius.tl, y);
  ctx.lineTo(x + width - radius.tr, y);
  ctx.quadraticCurveTo(x + width, y, x + width, y + radius.tr);
  ctx.lineTo(x + width, y + height - radius.br);
  ctx.quadraticCurveTo(x + width, y + height, x + width - radius.br, y + height);
  ctx.lineTo(x + radius.bl, y + height);
  ctx.quadraticCurveTo(x, y + height, x, y + height - radius.bl);
  ctx.lineTo(x, y + radius.tl);
  ctx.quadraticCurveTo(x, y, x + radius.tl, y);
  ctx.closePath();
  if (fill) ctx.fill();
  if (stroke) ctx.stroke();
}

// game over
function showGameOver(){
  // simple overlay
  ctx.fillStyle = 'rgba(0,0,0,0.45)';
  ctx.fillRect(0,0,W,H);
  ctx.fillStyle = '#fff';
  ctx.font = '28px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText('Game Over — Habul-habulan natapos', W/2, H/2 - 20);
  ctx.font = '20px sans-serif';
  ctx.fillText('Score: ' + score, W/2, H/2 + 14);
  // small restart hint
  ctx.font = '14px sans-serif';
  ctx.fillText('Pindutin ang Simulan para maglaro ulit', W/2, H/2 + 48);
}

// small intro draw
function drawIntro(){
  ctx.clearRect(0,0,W,H);
  drawBackground();
  ctx.fillStyle = '#222';
  ctx.font = '24px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText('Habul-Habulan', W/2, H/2 - 10);
  ctx.font = '14px sans-serif';
  ctx.fillText('Press Simulan para maglaro — arrow keys o touch controls sa mobile', W/2, H/2 + 18);
}

// audio (very small beeps using WebAudio)
let audioCtx = null;
function ensureAudio(){
  if(audioCtx) return audioCtx;
  try{ audioCtx = new (window.AudioContext || window.webkitAudioContext)(); } catch(e){ audioCtx = null; }
  return audioCtx;
}
function playBeep(){
  if(muted) return;
  const a = ensureAudio();
  if(!a) return;
  const o = a.createOscillator();
  const g = a.createGain();
  o.type = 'sine'; o.frequency.value = 880;
  g.gain.value = 0.08;
  o.connect(g); g.connect(a.destination);
  o.start();
  setTimeout(()=>{ o.stop(); }, 90);
}
function playHit(){
  if(muted) return;
  const a = ensureAudio(); if(!a) return;
  const o = a.createOscillator(); const g = a.createGain();
  o.type = 'square'; o.frequency.setValueAtTime(220, a.currentTime);
  g.gain.value = 0.12;
  o.connect(g); g.connect(a.destination);
  o.start();
  setTimeout(()=>{ o.stop(); }, 220);
}

// initial
resetGame();

// resize handler (if user resizes window, canvas style may scale but keep logical size)
window.addEventListener('resize', ()=> { /* could implement responsive scaling */ });

// keyboard accessibility (prevent scrolling when using arrows)
window.addEventListener('keydown', function(e) {
  if(['ArrowUp','ArrowDown','ArrowLeft','ArrowRight','Space'].includes(e.code)) e.preventDefault();
});
</script>
</body>
</html> Mema
