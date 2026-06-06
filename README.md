# explosivesnake  <!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Snake — Power Chests</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      background: #0a110b;
      display: flex;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      font-family: monospace;
      color: #fff;
    }
    #game-wrap { display: flex; flex-direction: column; align-items: center; gap: 12px; }
    h1 { font-size: 22px; font-weight: 700; letter-spacing: 0.2em; color: #5DCAA5; text-transform: uppercase; }
    #hud { display: flex; gap: 1.5rem; font-size: 12px; letter-spacing: 0.08em; text-transform: uppercase; color: #6b7c6d; }
    #hud span { font-weight: 700; color: #c8e6cb; }
    canvas { border: 1px solid #1f3322; border-radius: 8px; background: #0e1a10; display: block; }

    #powerbar {
      width: 400px; min-height: 40px;
      background: #0e1a10;
      border: 1px solid #1f3322;
      border-radius: 8px;
      display: flex; align-items: center;
      padding: 0 14px; gap: 12px;
      font-size: 12px; color: #6b7c6d;
      letter-spacing: 0.06em;
    }
    #powerbar .pw-icon { font-size: 20px; }
    #powerbar .pw-name { font-weight: 700; color: #fff; text-transform: uppercase; letter-spacing: 0.1em; font-size: 11px; }
    #powerbar .pw-desc { color: #6b7c6d; font-size: 11px; }
    #powerbar .pw-timer-wrap { flex: 1; height: 4px; background: #1a2e1c; border-radius: 2px; overflow: hidden; }
    #powerbar .pw-timer-fill { height: 100%; border-radius: 2px; transition: width 0.1s linear; }
    #powerbar .pw-cancel { font-size: 11px; color: #6b7c6d; white-space: nowrap; }
    #powerbar .pw-cancel kbd { background: #1a2e1c; border: 1px solid #2e4d32; border-radius: 4px; padding: 1px 5px; color: #5DCAA5; }

    #msg { font-size: 12px; color: #6b7c6d; letter-spacing: 0.05em; min-height: 16px; }
    #start-btn {
      padding: 8px 28px; font-size: 12px; font-family: monospace;
      letter-spacing: 0.1em; text-transform: uppercase;
      background: transparent; border: 1px solid #2e4d32;
      border-radius: 8px; color: #5DCAA5; cursor: pointer;
      transition: background 0.15s, border-color 0.15s;
    }
    #start-btn:hover { background: #1a2e1c; border-color: #5DCAA5; }

    #dpad { display: none; grid-template-columns: repeat(3, 52px); grid-template-rows: repeat(2, 52px); gap: 6px; }
    #dpad button {
      width: 52px; height: 52px; font-size: 20px;
      background: #1a2e1c; border: 1px solid #2e4d32;
      border-radius: 8px; color: #5DCAA5; cursor: pointer;
      display: flex; align-items: center; justify-content: center;
    }
    #dpad button:active { background: #2a4a2e; }
    #dpad .empty { visibility: hidden; }
    @media (pointer: coarse) { #dpad { display: grid; } }
  </style>
</head>
<body>
<div id="game-wrap">
  <h1>Snake</h1>
  <div id="hud">
    score <span id="score-val">0</span>&nbsp;&nbsp;|&nbsp;&nbsp;length <span id="len-val">1</span>
  </div>
  <canvas id="c" width="400" height="400"></canvas>
  <div id="powerbar"><span style="color:#2e4d32;font-size:11px;">no active power — open a chest to get one</span></div>
  <div id="msg">press start or any arrow key</div>
  <button id="start-btn">Start</button>
  <div id="dpad">
    <div class="empty"></div><button id="btn-up">↑</button><div class="empty"></div>
    <button id="btn-left">←</button><button id="btn-down">↓</button><button id="btn-right">→</button>
  </div>
</div>

<script>
const C   = document.getElementById('c');
const ctx = C.getContext('2d');
const CELL = 20, COLS = 20, ROWS = 20;
const scoreEl  = document.getElementById('score-val');
const lenEl    = document.getElementById('len-val');
const msgEl    = document.getElementById('msg');
const btn      = document.getElementById('start-btn');
const powerbar = document.getElementById('powerbar');

// ── Power definitions ──────────────────────────────────────────
const POWERS = {
  speed:   { name: 'Turbo',      icon: '⚡', desc: 'Double speed',        color: '#EF9F27', duration: 8000  },
  slow:    { name: 'Slow-Mo',    icon: '🐢', desc: 'Half speed',          color: '#5DCAA5', duration: 10000 },
  ghost:   { name: 'Ghost',      icon: '👻', desc: 'Pass through walls',  color: '#AFA9EC', duration: 8000  },
  shrink:  { name: 'Shrink',     icon: '✂️',  desc: 'Cuts length in half', color: '#F09595', duration: 0     },
  shield:  { name: 'Shield',     icon: '🛡️',  desc: '1 free death',       color: '#85B7EB', duration: 15000 },
  magnet:  { name: 'Magnet',     icon: '🧲', desc: 'Apple flies to you',  color: '#F4C0D1', duration: 8000  },
};

const BASE_SPEED = 120;
let snake, dir, nextDir, apple, score, running, exploding;
let particles = [], chests = [];
let activePower = null, powerTimer = 0, powerDuration = 0, powerStart = 0;
let shieldActive = false;
let gameLoopInterval, appleInterval, chestInterval, animFrame;

function rnd(n)      { return Math.floor(Math.random() * n); }
function rndF(a, b)  { return a + Math.random() * (b - a); }

function freeCell() {
  let pos, tries = 0;
  do {
    pos = { x: rnd(COLS), y: rnd(ROWS) };
    tries++;
    if (tries > 200) return pos;
  } while (
    snake.some(s => s.x === pos.x && s.y === pos.y) ||
    (apple && apple.x === pos.x && apple.y === pos.y) ||
    chests.some(ch => ch.x === pos.x && ch.y === pos.y)
  );
  return pos;
}

// ── Chest spawn ────────────────────────────────────────────────
function spawnChest() {
  if (!running) return;
  if (chests.length >= 2) return;
  const pos  = freeCell();
  const keys = Object.keys(POWERS);
  const power = keys[rnd(keys.length)];
  chests.push({ x: pos.x, y: pos.y, power, bob: 0 });
}

// ── Power activation ──────────────────────────────────────────
function activatePower(power) {
  // cancel existing
  if (activePower) deactivatePower(true);

  activePower = power;
  const def = POWERS[power];

  if (power === 'shrink') {
    const half = Math.max(1, Math.floor(snake.length / 2));
    snake = snake.slice(0, half);
    lenEl.textContent = snake.length;
    activePower = null;
    updatePowerBar();
    return;
  }

  if (power === 'shield') shieldActive = true;

  if (def.duration > 0) {
    powerDuration = def.duration;
    powerStart    = Date.now();
    clearInterval(gameLoopInterval);
    const spd = power === 'speed' ? BASE_SPEED / 2 : power === 'slow' ? BASE_SPEED * 2 : BASE_SPEED;
    gameLoopInterval = setInterval(tick, spd);
  }

  updatePowerBar();
}

function deactivatePower(silent) {
  if (!activePower) return;
  const old = activePower;
  activePower = null;
  shieldActive = false;
  powerDuration = 0;

  // restore normal speed
  clearInterval(gameLoopInterval);
  if (running) gameLoopInterval = setInterval(tick, BASE_SPEED);

  updatePowerBar();
  if (!silent) msgEl.textContent = `${POWERS[old].icon} ${POWERS[old].name} ended`;
}

function updatePowerBar() {
  if (!activePower) {
    powerbar.innerHTML = '<span style="color:#2e4d32;font-size:11px;">no active power — open a chest to get one</span>';
    return;
  }
  const def = POWERS[activePower];
  const elapsed  = Date.now() - powerStart;
  const fraction = Math.max(0, 1 - elapsed / powerDuration);
  const secs     = Math.ceil((powerDuration - elapsed) / 1000);
  powerbar.innerHTML = `
    <span class="pw-icon">${def.icon}</span>
    <div style="display:flex;flex-direction:column;gap:2px;min-width:90px">
      <span class="pw-name">${def.name}</span>
      <span class="pw-desc">${def.desc}</span>
    </div>
    <div class="pw-timer-wrap">
      <div class="pw-timer-fill" style="width:${fraction*100}%;background:${def.color}"></div>
    </div>
    <span style="font-size:11px;color:${def.color};min-width:24px;text-align:right">${secs}s</span>
    <span class="pw-cancel">cancel <kbd>E</kbd></span>
  `;
}

// ── Particles ────────────────────────────────────────────────
function spawnExplosion(snakeCopy) {
  particles = [];
  snakeCopy.forEach((seg, i) => {
    const cx = seg.x * CELL + CELL/2, cy = seg.y * CELL + CELL/2;
    const count = i === 0 ? 18 : 5;
    for (let p = 0; p < count; p++) {
      const angle = rndF(0, Math.PI*2), speed = rndF(1.5, i===0?7:4);
      particles.push({
        x:cx, y:cy, vx:Math.cos(angle)*speed, vy:Math.sin(angle)*speed,
        size:rndF(2, i===0?7:5), alpha:1, decay:rndF(0.018,0.04),
        color:`hsl(${rndF(100,160)},80%,60%)`, trail:[],
      });
    }
    if (i===0) for (let p=0;p<10;p++) {
      const angle = rndF(0,Math.PI*2);
      particles.push({
        x:cx, y:cy, vx:Math.cos(angle)*rndF(3,9), vy:Math.sin(angle)*rndF(3,9),
        size:rndF(1.5,3.5), alpha:1, decay:rndF(0.03,0.06),
        color:`hsl(${rndF(20,50)},100%,70%)`, trail:[],
      });
    }
  });
}
function spawnChestBurst(x, y, color) {
  for (let i=0;i<20;i++) {
    const angle=rndF(0,Math.PI*2), speed=rndF(2,6);
    particles.push({
      x:x*CELL+CELL/2, y:y*CELL+CELL/2,
      vx:Math.cos(angle)*speed, vy:Math.sin(angle)*speed,
      size:rndF(2,5), alpha:1, decay:rndF(0.025,0.05),
      color, trail:[],
    });
  }
}
function updateParticles() {
  particles.forEach(p=>{
    p.trail.push({x:p.x,y:p.y});
    if(p.trail.length>4) p.trail.shift();
    p.x+=p.vx; p.y+=p.vy; p.vy+=0.12; p.vx*=0.97;
    p.alpha-=p.decay; p.size*=0.985;
  });
  particles=particles.filter(p=>p.alpha>0);
}
function drawParticles() {
  particles.forEach(p=>{
    if(p.trail.length>1){
      ctx.beginPath(); ctx.moveTo(p.trail[0].x,p.trail[0].y);
      p.trail.forEach(t=>ctx.lineTo(t.x,t.y));
      ctx.strokeStyle=p.color; ctx.globalAlpha=p.alpha*0.3;
      ctx.lineWidth=p.size*0.5; ctx.stroke(); ctx.globalAlpha=1;
    }
    ctx.globalAlpha=p.alpha; ctx.fillStyle=p.color;
    ctx.beginPath(); ctx.arc(p.x,p.y,Math.max(0.1,p.size),0,Math.PI*2); ctx.fill();
    ctx.globalAlpha=1;
  });
}

// ── Chest drawing ──────────────────────────────────────────
function drawChest(ch, t) {
  const px = ch.x*CELL, py = ch.y*CELL;
  const bob = Math.sin(t * 0.003 + ch.x) * 2;
  const def = POWERS[ch.power];
  ctx.save(); ctx.translate(px + CELL/2, py + CELL/2 + bob);

  // glow ring
  ctx.strokeStyle = def.color; ctx.globalAlpha = 0.25 + 0.15*Math.sin(t*0.005);
  ctx.lineWidth = 3;
  ctx.beginPath(); ctx.arc(0, 0, CELL/2 + 3, 0, Math.PI*2); ctx.stroke();
  ctx.globalAlpha = 1;

  // chest body
  ctx.fillStyle = '#7b5c2a';
  ctx.beginPath(); ctx.roundRect(-CELL/2+1, -CELL/2+3, CELL-2, CELL-4, 3); ctx.fill();

  // chest lid
  ctx.fillStyle = '#a07840';
  ctx.beginPath(); ctx.roundRect(-CELL/2+1, -CELL/2+1, CELL-2, CELL/2-1, [3,3,0,0]); ctx.fill();

  // metal band
  ctx.fillStyle = '#c8a84b';
  ctx.fillRect(-CELL/2+1, -1, CELL-2, 3);

  // lock
  ctx.fillStyle = '#c8a84b';
  ctx.beginPath(); ctx.arc(0, 0, 2.5, 0, Math.PI*2); ctx.fill();

  // icon above
  ctx.font = '9px monospace';
  ctx.fillStyle = def.color;
  ctx.textAlign = 'center';
  ctx.fillText(def.icon, 0, -CELL/2 - 4);

  ctx.restore();
}

// ── Core game ────────────────────────────────────────────────
function init() {
  snake     = [{x:10,y:10}];
  dir       = {x:1,y:0}; nextDir = {x:1,y:0};
  apple     = freeCell();
  score     = 0; running = true; exploding = false;
  particles = []; chests   = [];
  activePower = null; shieldActive = false;
  powerDuration = 0;

  scoreEl.textContent = 0; lenEl.textContent = 1;
  msgEl.textContent   = 'use arrow keys';
  btn.textContent     = 'Restart';
  updatePowerBar();

  clearInterval(gameLoopInterval);
  clearInterval(appleInterval);
  clearInterval(chestInterval);
  cancelAnimationFrame(animFrame);

  gameLoopInterval = setInterval(tick, BASE_SPEED);
  appleInterval    = setInterval(()=>{ if(running && apple===null) apple=freeCell(); }, 2000);
  chestInterval    = setInterval(spawnChest, 20000);

  mainLoop();
}

function tick() {
  // power timer check
  if (activePower && powerDuration > 0) {
    const elapsed = Date.now() - powerStart;
    if (elapsed >= powerDuration) { deactivatePower(false); return; }
    updatePowerBar();
  }

  dir = nextDir;
  let nx = snake[0].x + dir.x;
  let ny = snake[0].y + dir.y;

  // ghost wrapping
  if (activePower === 'ghost') {
    nx = (nx + COLS) % COLS;
    ny = (ny + ROWS) % ROWS;
  }

  const head = {x:nx, y:ny};
  const wallHit  = head.x<0||head.x>=COLS||head.y<0||head.y>=ROWS;
  const selfHit  = snake.some(s=>s.x===head.x&&s.y===head.y);

  if (wallHit || selfHit) {
    if (shieldActive) {
      shieldActive = false;
      deactivatePower(true);
      msgEl.textContent = '🛡️ Shield absorbed the hit!';
      return;
    }
    triggerExplosion(); return;
  }

  snake.unshift(head);

  // magnet: move apple closer
  if (activePower === 'magnet' && apple) {
    const dx = Math.sign(head.x - apple.x);
    const dy = Math.sign(head.y - apple.y);
    const nx2 = apple.x + dx, ny2 = apple.y + dy;
    if (!snake.some(s=>s.x===nx2&&s.y===apple.y)) apple.x = nx2;
    if (!snake.some(s=>s.x===apple.x&&s.y===ny2)) apple.y = ny2;
  }

  // eat apple
  if (apple && head.x===apple.x && head.y===apple.y) {
    score++; scoreEl.textContent = score; apple = null;
  } else {
    snake.pop();
  }

  // open chest
  const chestIdx = chests.findIndex(ch=>ch.x===head.x&&ch.y===head.y);
  if (chestIdx !== -1) {
    const ch = chests.splice(chestIdx,1)[0];
    spawnChestBurst(ch.x, ch.y, POWERS[ch.power].color);
    activatePower(ch.power);
    msgEl.textContent = `${POWERS[ch.power].icon} ${POWERS[ch.power].name}! ${POWERS[ch.power].desc}`;
  }

  lenEl.textContent = snake.length;
}

let frameT = 0;
function mainLoop(t=0) {
  frameT = t;
  draw(t);
  if (!exploding) animFrame = requestAnimationFrame(mainLoop);
}

function draw(t=0) {
  ctx.clearRect(0,0,C.width,C.height);
  ctx.fillStyle='#0e1a10'; ctx.fillRect(0,0,C.width,C.height);

  // grid
  ctx.fillStyle='#112214';
  for(let x=0;x<COLS;x++) for(let y=0;y<ROWS;y++)
    if((x+y)%2===0) ctx.fillRect(x*CELL,y*CELL,CELL,CELL);

  // chests
  chests.forEach(ch=>drawChest(ch,t));

  // apple
  if (apple) {
    const ax=apple.x*CELL+CELL/2, ay=apple.y*CELL+CELL/2;
    ctx.fillStyle='#E24B4A'; ctx.beginPath(); ctx.arc(ax,ay,CELL/2-2,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#5DCAA5'; ctx.fillRect(ax-1,ay-CELL/2+1,2,5);
    ctx.fillStyle='#3B6D11'; ctx.beginPath(); ctx.ellipse(ax+3,ay-CELL/2+4,4,2,-0.5,0,Math.PI*2); ctx.fill();
  }

  // snake
  snake.forEach((s,i)=>{
    const frac = i/Math.max(snake.length,1);
    let color;
    if (activePower === 'ghost')  color = i===0?'rgba(175,169,236,0.9)':`rgba(175,169,236,${0.5-frac*0.3})`;
    else if (activePower==='shield') color = i===0?'#85B7EB':`hsl(210,60%,${45-frac*15}%)`;
    else if (activePower==='speed')  color = i===0?'#EF9F27':`hsl(${40-frac*10},80%,${50-frac*15}%)`;
    else if (activePower==='magnet') color = i===0?'#F4C0D1':`hsl(${340-frac*10},70%,${55-frac*15}%)`;
    else color = i===0?'#5DCAA5':`hsl(${140-frac*30},60%,${45-frac*15}%)`;

    ctx.fillStyle = color;
    ctx.beginPath();
    ctx.roundRect(s.x*CELL+1,s.y*CELL+1,CELL-2,CELL-2,4);
    ctx.fill();

    // shield aura
    if (activePower==='shield' && i===0) {
      ctx.strokeStyle='rgba(133,183,235,0.5)';
      ctx.lineWidth=2; ctx.beginPath();
      ctx.arc(s.x*CELL+CELL/2,s.y*CELL+CELL/2,CELL/2+3,0,Math.PI*2); ctx.stroke();
    }

    if (i===0) {
      ctx.fillStyle='#0e1a10';
      const ex=dir.x,ey=dir.y,cx=s.x*CELL+CELL/2,cy=s.y*CELL+CELL/2;
      ctx.beginPath(); ctx.arc(cx+ex*4+ey*3,cy+ey*4+ex*3,2,0,Math.PI*2); ctx.fill();
      ctx.beginPath(); ctx.arc(cx+ex*4-ey*3,cy+ey*4-ex*3,2,0,Math.PI*2); ctx.fill();
    }
  });

  drawParticles();
  updateParticles();
}

function triggerExplosion() {
  running=false; exploding=true;
  clearInterval(gameLoopInterval); clearInterval(appleInterval); clearInterval(chestInterval);
  cancelAnimationFrame(animFrame);
  activePower=null; shieldActive=false; updatePowerBar();
  msgEl.textContent='game over — press restart';
  spawnExplosion([...snake]); snake=[];

  function animExp() {
    draw(frameT);
    updateParticles(); drawParticles();
    frameT+=16;
    if(particles.length>0) requestAnimationFrame(animExp);
    else { exploding=false; drawGameOver(); }
  }
  requestAnimationFrame(animExp);
}

function drawGameOver() {
  ctx.fillStyle='rgba(0,0,0,0.62)'; ctx.fillRect(0,0,C.width,C.height);
  ctx.fillStyle='#fff'; ctx.font='700 24px monospace'; ctx.textAlign='center';
  ctx.fillText('GAME OVER',C.width/2,C.height/2-14);
  ctx.font='400 14px monospace'; ctx.fillStyle='#6b7c6d';
  ctx.fillText('score: '+score,C.width/2,C.height/2+14);
  ctx.textAlign='left';
}

// ── Controls ────────────────────────────────────────────────
btn.addEventListener('click',init);

document.addEventListener('keydown',e=>{
  const map={ArrowUp:{x:0,y:-1},ArrowDown:{x:0,y:1},ArrowLeft:{x:-1,y:0},ArrowRight:{x:1,y:0}};
  if(map[e.key]){
    e.preventDefault();
    const d=map[e.key];
    if(d.x!==-dir.x||d.y!==-dir.y) nextDir=d;
    if(!running&&!exploding) init();
  }
  if(e.key==='e'||e.key==='E') {
    e.preventDefault();
    if(activePower) deactivatePower(false);
  }
});

document.getElementById('btn-up').onclick    = ()=>{ if(dir.y!== 1) nextDir={x:0,y:-1};  if(!running&&!exploding) init(); };
document.getElementById('btn-down').onclick  = ()=>{ if(dir.y!==-1) nextDir={x:0,y: 1};  if(!running&&!exploding) init(); };
document.getElementById('btn-left').onclick  = ()=>{ if(dir.x!== 1) nextDir={x:-1,y:0};  if(!running&&!exploding) init(); };
document.getElementById('btn-right').onclick = ()=>{ if(dir.x!==-1) nextDir={x: 1,y:0};  if(!running&&!exploding) init(); };

draw(0);
</script>
</body>
</html>
