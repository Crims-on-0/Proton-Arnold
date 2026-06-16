# Proton-Arnold
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, orientation=landscape">
<title>PROTON ARNOLD</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; }
  body {
    background:#000;
    overflow:hidden;
    touch-action:none;
    user-select:none;
    -webkit-user-select:none;
    font-family: 'Courier New', monospace;
    display:flex;
    flex-direction:column;
    height:100vh;
    width:100vw;
  }
  #gameCanvas {
    display:block;
    flex-shrink:0;
  }
  #controls {
    flex:1;
    background:#0a0a1a;
    border-top:2px solid #00ffff44;
    display:flex;
    align-items:center;
    justify-content:space-between;
    padding:4px 16px;
    min-height:110px;
    max-height:140px;
  }
  .dpad {
    position:relative;
    width:110px;
    height:110px;
    flex-shrink:0;
  }
  .dpad-btn {
    position:absolute;
    background:#111;
    border:2px solid #00ffff55;
    border-radius:6px;
    display:flex;
    align-items:center;
    justify-content:center;
    font-size:18px;
    color:#00ffff;
    cursor:pointer;
    -webkit-tap-highlight-color:transparent;
    transition:background 0.05s;
    width:34px;
    height:34px;
  }
  .dpad-btn:active, .dpad-btn.pressed {
    background:#00ffff33;
    border-color:#00ffff;
  }
  .dpad-up    { top:0;    left:38px; }
  .dpad-down  { bottom:0; left:38px; }
  .dpad-left  { top:38px; left:0; }
  .dpad-right { top:38px; right:0; }
  .dpad-center {
    position:absolute;
    top:38px; left:38px;
    width:34px; height:34px;
    background:#1a1a1a;
    border:2px solid #333;
    border-radius:6px;
  }
  .action-btns {
    display:flex;
    flex-direction:column;
    gap:10px;
    align-items:center;
  }
  .btn-fire {
    width:70px; height:70px;
    border-radius:50%;
    background:radial-gradient(circle, #ff4400, #aa2200);
    border:3px solid #ff6600;
    color:#fff;
    font-size:11px;
    font-weight:bold;
    letter-spacing:1px;
    cursor:pointer;
    -webkit-tap-highlight-color:transparent;
    display:flex;
    align-items:center;
    justify-content:center;
    flex-direction:column;
    text-align:center;
  }
  .btn-fire:active { background:radial-gradient(circle, #ff6600, #cc3300); transform:scale(0.95); }
  .score-display {
    color:#00ffff;
    font-size:10px;
    text-align:center;
    letter-spacing:1px;
    line-height:1.6;
  }
  .score-display .val { font-size:13px; color:#fff; }
</style>
</head>
<body>

<canvas id="gameCanvas"></canvas>

<div id="controls">
  <!-- D-PAD LEFT -->
  <div class="dpad" id="dpad">
    <div class="dpad-btn dpad-up"    data-dir="up">▲</div>
    <div class="dpad-btn dpad-left"  data-dir="left">◀</div>
    <div class="dpad-center"></div>
    <div class="dpad-btn dpad-right" data-dir="right">▶</div>
    <div class="dpad-btn dpad-down"  data-dir="down">▼</div>
  </div>

  <!-- SCORE CENTER -->
  <div class="score-display">
    <div id="ui-p1">P1</div>
    <div class="val" id="ui-score">000000</div>
    <div style="color:#ffff00;margin-top:4px">HI-SCORE</div>
    <div class="val" id="ui-hi">1232339</div>
    <div style="color:#ff4466;margin-top:4px" id="ui-lives">♥ ♥ ♥</div>
    <div style="color:#444;margin-top:2px;font-size:9px" id="ui-wave">WAVE 1</div>
  </div>

  <!-- FIRE BUTTON RIGHT -->
  <div class="action-btns">
    <div class="btn-fire" id="btnFire">⚡<br>FIRE</div>
  </div>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resize() {
  const ctrlH = document.getElementById('controls').offsetHeight || 120;
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight - ctrlH;
}
resize();
window.addEventListener('resize', () => { resize(); });

// ---- STATE ----
const keys = { up:false, down:false, left:false, right:false, fire:false };
let firePressed = false;
let score = 0, hiScore = 1232339, lives = 3, wave = 1;
let gameState = 'title'; // title | playing | dead | gameover
let combo = 0, comboTimer = 0;
let bonusPopups = [];
let particles = [];

// ---- D-PAD TOUCH ----
const dpadBtns = document.querySelectorAll('.dpad-btn');
const activePointers = {};

function dirDown(dir) {
  if(dir==='up')    keys.up=true;
  if(dir==='down')  keys.down=true;
  if(dir==='left')  keys.left=true;
  if(dir==='right') keys.right=true;
}
function dirUp(dir) {
  if(dir==='up')    keys.up=false;
  if(dir==='down')  keys.down=false;
  if(dir==='left')  keys.left=false;
  if(dir==='right') keys.right=false;
}

dpadBtns.forEach(btn => {
  btn.addEventListener('touchstart', e => {
    e.preventDefault();
    [...e.changedTouches].forEach(t => { activePointers[t.identifier] = btn.dataset.dir; });
    dirDown(btn.dataset.dir);
    btn.classList.add('pressed');
  }, {passive:false});
  btn.addEventListener('touchend', e => {
    e.preventDefault();
    [...e.changedTouches].forEach(t => {
      dirUp(activePointers[t.identifier]);
      delete activePointers[t.identifier];
    });
    btn.classList.remove('pressed');
  }, {passive:false});
});

const btnFire = document.getElementById('btnFire');
btnFire.addEventListener('touchstart', e => { e.preventDefault(); firePressed=true; }, {passive:false});
btnFire.addEventListener('touchend',   e => { e.preventDefault(); firePressed=false; }, {passive:false});
btnFire.addEventListener('mousedown',  e => { firePressed=true; });
btnFire.addEventListener('mouseup',    e => { firePressed=false; });

// keyboard fallback
window.addEventListener('keydown', e => {
  if(e.key==='ArrowUp')    keys.up=true;
  if(e.key==='ArrowDown')  keys.down=true;
  if(e.key==='ArrowLeft')  keys.left=true;
  if(e.key==='ArrowRight') keys.right=true;
  if(e.key===' ') { firePressed=true; e.preventDefault(); }
  if(e.key==='Enter' && gameState!=='playing') startGame();
});
window.addEventListener('keyup', e => {
  if(e.key==='ArrowUp')    keys.up=false;
  if(e.key==='ArrowDown')  keys.down=false;
  if(e.key==='ArrowLeft')  keys.left=false;
  if(e.key==='ArrowRight') keys.right=false;
  if(e.key===' ') firePressed=false;
});

// ---- STARS ----
const starColors = ['#00ffff','#ffff00','#ff00ff','#00ff88','#ffffff'];
let stars = [];
function makeStars() {
  stars = [];
  for(let i=0;i<80;i++) {
    stars.push({
      x: Math.random()*canvas.width,
      y: Math.random()*canvas.height,
      r: Math.random()*1.5+0.5,
      c: starColors[Math.floor(Math.random()*starColors.length)],
      spd: Math.random()*0.3+0.1
    });
  }
}

// ---- PLAYER ----
let player;
function makePlayer() {
  player = {
    x: canvas.width*0.5,
    y: canvas.height*0.35,
    w: 22, h: 28,
    spd: 180,
    fireCooldown: 0,
    fireRate: 0.25,
    invincible: 0,
    shield: false,
    multishot: false,
    multishotTimer: 0
  };
}

// ---- BULLETS ----
let bullets = [];
function fireBullet() {
  if(player.fireCooldown > 0) return;
  player.fireCooldown = player.fireRate;
  const angles = player.multishot ? [-0.2, 0, 0.2] : [0];
  angles.forEach(a => {
    const spd = 420;
    bullets.push({ x:player.x, y:player.y-14, vx:Math.sin(a)*spd, vy:-Math.cos(a)*spd, r:4 });
  });
}

// ---- ENEMIES ----
let enemies = [];
let enemyBullets = [];

function spawnWave() {
  enemies = [];
  const count = 3 + wave * 2;
  for(let i=0;i<count;i++) {
    const type = Math.random() < 0.4 ? 'crab' : 'bug';
    const side = Math.floor(Math.random()*4);
    let ex, ey;
    if(side===0)      { ex=Math.random()*canvas.width; ey=-30; }
    else if(side===1) { ex=canvas.width+30; ey=Math.random()*canvas.height*0.7; }
    else if(side===2) { ex=-30; ey=Math.random()*canvas.height*0.7; }
    else              { ex=Math.random()*canvas.width; ey=-30; }

    enemies.push({
      x:ex, y:ey,
      type,
      hp: type==='crab' ? 3 : 2,
      spd: 55 + wave*8,
      angle: 0,
      shootTimer: 1.5 + Math.random()*2,
      wobble: Math.random()*Math.PI*2,
      wobbleSpd: 1+Math.random()*2,
      wobbleAmt: 20+Math.random()*30,
      points: type==='crab' ? 500 : 300,
      flashTimer: 0
    });
  }
}

// ---- POWERUPS ----
let powerups = [];
function spawnPowerup(x, y) {
  if(Math.random() > 0.3) return;
  const types = ['shield','multishot','extralife'];
  powerups.push({ x, y, type:types[Math.floor(Math.random()*types.length)], vy:40, life:6 });
}

// ---- PLANET ----
function drawPlanet() {
  const cx = canvas.width/2;
  const cy = canvas.height + canvas.height*0.15;
  const r  = canvas.height*0.55;
  const g = ctx.createRadialGradient(cx-r*0.2, cy-r*0.3, r*0.1, cx, cy, r);
  g.addColorStop(0,'#555');
  g.addColorStop(0.5,'#333');
  g.addColorStop(1,'#111');
  ctx.beginPath();
  ctx.arc(cx, cy, r, 0, Math.PI*2);
  ctx.fillStyle = g;
  ctx.fill();
  // craters
  [[cx-r*0.25, cy-r*0.15, r*0.08],[cx+r*0.1, cy-r*0.05, r*0.05],[cx-r*0.05, cy-r*0.3, r*0.04]].forEach(([x,y,cr])=>{
    ctx.beginPath();
    ctx.arc(x,y,cr,0,Math.PI*2);
    ctx.fillStyle='#222';
    ctx.fill();
  });
}

// ---- DRAW PLAYER ----
function drawPlayer() {
  if(player.invincible > 0 && Math.floor(player.invincible*10)%2===0) return;
  const {x,y} = player;
  // body suit - blue
  ctx.fillStyle = player.shield ? '#00ffff' : '#2255cc';
  ctx.fillRect(x-7, y-8, 14, 16);
  // yellow chest stripe
  ctx.fillStyle='#ffdd00';
  ctx.fillRect(x-4, y-4, 8, 5);
  // head
  ctx.fillStyle='#ffccaa';
  ctx.fillRect(x-5, y-17, 10, 11);
  // red hair
  ctx.fillStyle='#cc2200';
  ctx.fillRect(x-6, y-20, 12, 5);
  ctx.fillRect(x-7, y-18, 3, 4);
  // legs
  ctx.fillStyle='#1144aa';
  ctx.fillRect(x-6, y+8, 5, 7);
  ctx.fillRect(x+1, y+8, 5, 7);
  // boots
  ctx.fillStyle='#333';
  ctx.fillRect(x-7, y+14, 6, 3);
  ctx.fillRect(x+1, y+14, 6, 3);
  // shield ring
  if(player.shield) {
    ctx.beginPath();
    ctx.arc(x, y, 22, 0, Math.PI*2);
    ctx.strokeStyle='#00ffff88';
    ctx.lineWidth=3;
    ctx.stroke();
    ctx.lineWidth=1;
  }
}

// ---- DRAW BUG ENEMY ----
function drawBug(e) {
  const {x,y,flashTimer} = e;
  const c = flashTimer>0 ? '#ffffff' : '#aa22ff';
  const c2 = flashTimer>0 ? '#ffffff' : '#7700cc';
  // wings X shape
  ctx.fillStyle=c;
  ctx.save();
  ctx.translate(x,y);
  ctx.rotate(Math.PI/4);
  ctx.fillRect(-12,-4,24,8);
  ctx.rotate(-Math.PI/2);
  ctx.fillRect(-12,-4,24,8);
  ctx.restore();
  // body center
  ctx.fillStyle=c2;
  ctx.fillRect(x-5,y-7,10,14);
  // eyes
  ctx.fillStyle='#ff0000';
  ctx.fillRect(x-4,y-5,3,3);
  ctx.fillRect(x+1,y-5,3,3);
}

// ---- DRAW CRAB ENEMY ----
function drawCrab(e) {
  const {x,y,flashTimer} = e;
  const c = flashTimer>0 ? '#ffffff' : '#ff44aa';
  // body
  ctx.fillStyle=c;
  ctx.fillRect(x-10,y-7,20,13);
  // eyes
  ctx.fillStyle='#ffff00';
  ctx.fillRect(x-7,y-5,4,4);
  ctx.fillRect(x+3,y-5,4,4);
  ctx.fillStyle='#000';
  ctx.fillRect(x-6,y-4,2,2);
  ctx.fillRect(x+4,y-4,2,2);
  // claws
  ctx.fillStyle=c;
  ctx.fillRect(x-16,y-3,7,5);
  ctx.fillRect(x+9,y-3,7,5);
  // legs
  for(let i=0;i<3;i++) {
    ctx.fillRect(x-10+i*5, y+6, 3, 6);
    ctx.fillRect(x+3+i*5,  y+6, 3, 6);
  }
}

// ---- DRAW POWERUP ----
function drawPowerup(p) {
  ctx.save();
  ctx.translate(p.x,p.y);
  if(p.type==='shield') {
    ctx.fillStyle='#00ffff';
    ctx.beginPath(); ctx.arc(0,0,9,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#000'; ctx.font='bold 10px monospace'; ctx.textAlign='center'; ctx.textBaseline='middle';
    ctx.fillText('S',0,1);
  } else if(p.type==='multishot') {
    ctx.fillStyle='#ffff00';
    ctx.beginPath(); ctx.arc(0,0,9,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#000'; ctx.font='bold 10px monospace'; ctx.textAlign='center'; ctx.textBaseline='middle';
    ctx.fillText('M',0,1);
  } else {
    ctx.fillStyle='#ff4466';
    ctx.beginPath(); ctx.arc(0,0,9,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#fff'; ctx.font='bold 10px monospace'; ctx.textAlign='center'; ctx.textBaseline='middle';
    ctx.fillText('♥',0,1);
  }
  ctx.restore();
}

// ---- PARTICLES ----
function spawnExplosion(x, y, color) {
  for(let i=0;i<18;i++) {
    const a = (Math.PI*2/18)*i;
    const spd = 60+Math.random()*100;
    particles.push({ x, y, vx:Math.cos(a)*spd, vy:Math.sin(a)*spd, r:2+Math.random()*3, life:0.7, maxLife:0.7, c:color });
  }
}

// ---- BONUS POPUP ----
function spawnBonus(x, y, pts) {
  bonusPopups.push({ x, y, pts, life:1.2, maxLife:1.2 });
  // sparkle diamonds for 5000
  if(pts>=5000) {
    for(let i=0;i<16;i++) {
      const a = (Math.PI*2/16)*i;
      const d = 20+Math.random()*25;
      particles.push({ x:x+Math.cos(a)*d, y:y+Math.sin(a)*d, vx:Math.cos(a)*40, vy:Math.sin(a)*40, r:4, life:1, maxLife:1, c:'#00ff88', diamond:true });
    }
  }
}

// ---- RECT COLLIDE ----
function rectsOverlap(ax,ay,aw,ah,bx,by,bw,bh) {
  return ax-aw/2<bx+bw/2 && ax+aw/2>bx-bw/2 && ay-ah/2<by+bh/2 && ay+ah/2>by-bh/2;
}

// ---- GAME LOGIC ----
let fireTapCooldown = 0;
let deadTimer = 0;
let waveClearing = false;
let waveClearTimer = 0;

function startGame() {
  score=0; lives=3; wave=1; combo=0;
  bullets=[]; enemies=[]; enemyBullets=[]; powerups=[]; particles=[]; bonusPopups=[];
  makePlayer();
  makeStars();
  spawnWave();
  gameState='playing';
  updateHUD();
}

function updateHUD() {
  document.getElementById('ui-score').textContent = String(score).padStart(6,'0');
  document.getElementById('ui-hi').textContent = String(Math.max(score,hiScore)).padStart(7,'0');
  document.getElementById('ui-lives').textContent = '♥ '.repeat(lives).trim() || '☆';
  document.getElementById('ui-wave').textContent = 'WAVE '+wave;
}

let lastTime = 0;
function loop(ts) {
  const dt = Math.min((ts-lastTime)/1000, 0.05);
  lastTime=ts;

  ctx.fillStyle='#000010';
  ctx.fillRect(0,0,canvas.width,canvas.height);

  // stars
  stars.forEach(s => {
    s.y += s.spd;
    if(s.y>canvas.height) { s.y=-2; s.x=Math.random()*canvas.width; }
    ctx.fillStyle=s.c;
    ctx.fillRect(s.x,s.y,s.r,s.r);
  });

  drawPlanet();

  if(gameState==='title') {
    drawTitle();
  } else if(gameState==='playing' || gameState==='dead') {
    updateGame(dt);
    drawGame();
  } else if(gameState==='gameover') {
    drawGameOver();
  }

  requestAnimationFrame(loop);
}

function updateGame(dt) {
  if(gameState==='dead') {
    deadTimer-=dt;
    if(deadTimer<=0) {
      if(lives>0) {
        makePlayer();
        gameState='playing';
        bullets=[]; enemyBullets=[];
      } else {
        hiScore=Math.max(score,hiScore);
        gameState='gameover';
      }
    }
    updateParticles(dt);
    return;
  }

  // PLAYER MOVE
  if(keys.up)    player.y -= player.spd*dt;
  if(keys.down)  player.y += player.spd*dt;
  if(keys.left)  player.x -= player.spd*dt;
  if(keys.right) player.x += player.spd*dt;
  player.x = Math.max(14, Math.min(canvas.width-14, player.x));
  player.y = Math.max(18, Math.min(canvas.height-20, player.y));

  // FIRE
  player.fireCooldown -= dt;
  fireTapCooldown -= dt;
  if(firePressed) fireBullet();

  if(player.invincible>0) player.invincible-=dt;
  if(player.multishotTimer>0) { player.multishotTimer-=dt; if(player.multishotTimer<=0) player.multishot=false; }

  // BULLETS
  bullets = bullets.filter(b => {
    b.x+=b.vx*dt; b.y+=b.vy*dt;
    return b.x>0&&b.x<canvas.width&&b.y>0&&b.y<canvas.height;
  });

  // COMBO timer
  if(comboTimer>0) { comboTimer-=dt; if(comboTimer<=0) combo=0; }

  // ENEMIES
  enemies.forEach(e => {
    // wobble toward player
    const dx = player.x-e.x, dy = player.y-e.y;
    const dist = Math.sqrt(dx*dx+dy*dy)||1;
    e.wobble += e.wobbleSpd*dt;
    const perpX = -dy/dist, perpY = dx/dist;
    e.x += (dx/dist*e.spd + perpX*Math.sin(e.wobble)*e.wobbleAmt)*dt;
    e.y += (dy/dist*e.spd + perpY*Math.sin(e.wobble)*e.wobbleAmt)*dt;
    if(e.flashTimer>0) e.flashTimer-=dt;

    // enemy shoot
    e.shootTimer-=dt;
    if(e.shootTimer<=0) {
      e.shootTimer=1.5+Math.random()*2 - wave*0.05;
      if(dist < canvas.width*0.8) {
        const spd=100+wave*15;
        enemyBullets.push({ x:e.x, y:e.y, vx:dx/dist*spd, vy:dy/dist*spd, r:5 });
      }
    }

    // bullet hits enemy
    bullets.forEach((b,bi) => {
      if(Math.abs(b.x-e.x)<16 && Math.abs(b.y-e.y)<16) {
        e.hp--;
        e.flashTimer=0.12;
        bullets.splice(bi,1);
        spawnExplosion(b.x,b.y,'#ff8800');
        if(e.hp<=0) {
          e.dead=true;
          combo++;
          comboTimer=2;
          const pts = e.points * (combo>=5?3:combo>=3?2:1);
          score+=pts;
          hiScore=Math.max(score,hiScore);
          if(combo>=5) spawnBonus(e.x,e.y,5000);
          else spawnBonus(e.x,e.y,pts);
          spawnExplosion(e.x,e.y, e.type==='crab'?'#ff44aa':'#aa22ff');
          spawnPowerup(e.x,e.y);
          updateHUD();
        }
      }
    });
  });
  enemies = enemies.filter(e=>!e.dead);

  // enemy bullets
  enemyBullets = enemyBullets.filter(b => {
    b.x+=b.vx*dt; b.y+=b.vy*dt;
    return b.x>-20&&b.x<canvas.width+20&&b.y>-20&&b.y<canvas.height+20;
  });

  // player hit by enemy bullet
  if(player.invincible<=0) {
    enemyBullets.forEach((b,i) => {
      if(Math.abs(b.x-player.x)<14 && Math.abs(b.y-player.y)<16) {
        if(player.shield) { player.shield=false; enemyBullets.splice(i,1); spawnExplosion(b.x,b.y,'#00ffff'); return; }
        enemyBullets.splice(i,1);
        playerHit();
      }
    });
    // player hit by enemy body
    enemies.forEach(e => {
      if(Math.abs(e.x-player.x)<18 && Math.abs(e.y-player.y)<20) {
        if(player.shield) { player.shield=false; e.dead=true; spawnExplosion(e.x,e.y,'#00ffff'); return; }
        playerHit();
      }
    });
  }

  // powerups
  powerups.forEach((p,i) => {
    p.y+=p.vy*dt;
    p.life-=dt;
    if(Math.abs(p.x-player.x)<18 && Math.abs(p.y-player.y)<18) {
      if(p.type==='shield')    { player.shield=true; }
      if(p.type==='multishot') { player.multishot=true; player.multishotTimer=10; }
      if(p.type==='extralife') { lives=Math.min(5,lives+1); updateHUD(); }
      spawnBonus(p.x,p.y,'BONUS!');
      powerups.splice(i,1);
    }
  });
  powerups=powerups.filter(p=>p.life>0&&p.y<canvas.height+30);

  // wave clear
  if(enemies.length===0 && !waveClearing) {
    waveClearing=true; waveClearTimer=2;
    score+=1000; updateHUD();
  }
  if(waveClearing) {
    waveClearTimer-=dt;
    if(waveClearTimer<=0) { wave++; waveClearing=false; spawnWave(); updateHUD(); }
  }

  updateParticles(dt);
  bonusPopups=bonusPopups.filter(p=>{p.life-=dt;return p.life>0;});
}

function playerHit() {
  if(player.invincible>0) return;
  lives--; updateHUD();
  spawnExplosion(player.x,player.y,'#ffff00');
  player.invincible=2.5;
  if(lives<=0) { gameState='dead'; deadTimer=2.5; }
}

function updateParticles(dt) {
  particles=particles.filter(p=>{ p.x+=p.vx*dt; p.y+=p.vy*dt; p.vx*=0.92; p.vy*=0.92; p.life-=dt; return p.life>0; });
}

function drawGame() {
  // bullets
  bullets.forEach(b => {
    ctx.fillStyle='#00ffff';
    ctx.beginPath(); ctx.arc(b.x,b.y,b.r,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#ffffff88';
    ctx.fillRect(b.x-1,b.y,2,12);
  });

  // enemy bullets
  enemyBullets.forEach(b => {
    ctx.fillStyle='#ff4400';
    ctx.beginPath(); ctx.arc(b.x,b.y,b.r,0,Math.PI*2); ctx.fill();
  });

  // enemies
  enemies.forEach(e => {
    if(e.type==='bug') drawBug(e);
    else drawCrab(e);
  });

  // powerups
  powerups.forEach(p=>drawPowerup(p));

  // particles
  particles.forEach(p => {
    const alpha = p.life/p.maxLife;
    ctx.globalAlpha=alpha;
    ctx.fillStyle=p.c;
    if(p.diamond) {
      ctx.save(); ctx.translate(p.x,p.y); ctx.rotate(Math.PI/4);
      ctx.fillRect(-p.r/2,-p.r/2,p.r,p.r);
      ctx.restore();
    } else {
      ctx.beginPath(); ctx.arc(p.x,p.y,p.r,0,Math.PI*2); ctx.fill();
    }
    ctx.globalAlpha=1;
  });

  // player
  if(gameState==='playing') drawPlayer();

  // bonus popups
  bonusPopups.forEach(p => {
    const alpha = p.life/p.maxLife;
    ctx.globalAlpha=alpha;
    ctx.fillStyle=p.pts>=5000?'#00ff88':'#ffff00';
    ctx.font=`bold ${p.pts>=5000?22:15}px 'Courier New'`;
    ctx.textAlign='center';
    ctx.fillText(typeof p.pts==='number'?'+'+p.pts:p.pts, p.x, p.y-(1-alpha)*30);
    ctx.globalAlpha=1;
  });

  // wave clear banner
  if(waveClearing && waveClearTimer>0.5) {
    ctx.fillStyle='#ffff0088';
    ctx.font="bold 20px 'Courier New'";
    ctx.textAlign='center';
    ctx.fillText('WAVE CLEAR! +1000', canvas.width/2, canvas.height/2);
  }
}

function drawTitle() {
  const cx=canvas.width/2, cy=canvas.height/2;
  // glow title
  ctx.shadowColor='#00ffff';
  ctx.shadowBlur=20;
  ctx.fillStyle='#00ffff';
  ctx.font=`bold ${Math.min(48,canvas.width/10)}px 'Courier New'`;
  ctx.textAlign='center';
  ctx.fillText('PROTON ARNOLD', cx, cy-30);
  ctx.shadowBlur=0;
  ctx.fillStyle='#ffffff66';
  ctx.font=`11px 'Courier New'`;
  ctx.fillText('AS SEEN ON  SCORPION', cx, cy-8);
  ctx.fillStyle='#ffff00';
  ctx.font=`13px 'Courier New'`;
  const blink = Math.floor(Date.now()/500)%2;
  if(blink) ctx.fillText('TAP FIRE TO START', cx, cy+20);
  ctx.fillStyle='#444';
  ctx.font=`10px 'Courier New'`;
  ctx.fillText('CREDIT 0', cx, canvas.height-10);

  if(firePressed && fireTapCooldown<=0) { fireTapCooldown=0.5; startGame(); }
}

function drawGameOver() {
  const cx=canvas.width/2, cy=canvas.height/2;
  ctx.fillStyle='#ff2200';
  ctx.font=`bold 32px 'Courier New'`;
  ctx.textAlign='center';
  ctx.fillText('GAME OVER', cx, cy-20);
  ctx.fillStyle='#ffffff';
  ctx.font=`13px 'Courier New'`;
  ctx.fillText('SCORE: '+String(score).padStart(6,'0'), cx, cy+5);
  ctx.fillStyle='#ffff00';
  ctx.fillText('HI: '+String(hiScore).padStart(7,'0'), cx, cy+24);
  const blink=Math.floor(Date.now()/600)%2;
  if(blink) { ctx.fillStyle='#00ffff'; ctx.font=`12px 'Courier New'`; ctx.fillText('TAP FIRE TO RETRY', cx, cy+48); }
  if(firePressed && fireTapCooldown<=0) { fireTapCooldown=0.5; startGame(); }
}

makeStars();
requestAnimationFrame(loop);
</script>
</body>
</html>
