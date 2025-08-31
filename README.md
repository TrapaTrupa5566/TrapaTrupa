<!doctype html>
<html lang="ka">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Stealth Ops — Simple Top-Down</title>
<style>
  html,body{height:100%;margin:0;font-family:Inter,Arial;background:#0e2b18;color:#e6f0ea;display:flex;align-items:center;justify-content:center}
  canvas{background:linear-gradient(#6db46e,#3b7f47);border-radius:10px;box-shadow:0 8px 30px rgba(0,0,0,.6)}
  #ui{position:fixed;left:12px;top:12px;color:#fff;font-weight:600}
  button{padding:8px 12px;border-radius:8px;border:none;background:#1f6feb;color:#fff}
  .touch{position:fixed;right:12px;bottom:12px;display:flex;gap:8px}
  .touch button{width:64px;height:64px;border-radius:12px;font-weight:800}
  #hint{position:fixed;left:12px;bottom:12px;color:#dbeadf}
</style>
</head>
<body>
<div id="ui">
  Score: <span id="score">0</span> &nbsp;|&nbsp; Lives: <span id="lives">1</span>
  &nbsp;|&nbsp;<button id="restart">Restart</button>
</div>
<canvas id="c" width="960" height="640"></canvas>

<div id="hint">Move: WASD / Arrow keys or touch. Kill stealthily: E / Tap "KILL". If enemy sees you → instant death.</div>
<div class="touch" id="touchControls" style="display:none;">
  <button id="btnUp">▲</button>
  <button id="btnLeft">◀</button>
  <button id="btnRight">▶</button>
  <button id="btnDown">▼</button>
  <button id="btnKill" style="background:#e03b3b">KILL</button>
</div>

<script>
/* ====== Stealth Ops — Top-Down Simple Game ======
   Mechanics:
   - Top-down movement (WASD / arrows / touch)
   - 4 enemies with patrol waypoints and vision cones
   - If player enters cone & has clear LOS -> instant death (game over)
   - If player is behind enemy (close + angle) -> press E / KILL to stealth kill
   - Simple houses (blocking vision) & grass environment
   - AKM drawn on player's back
*/
const canvas = document.getElementById('c'), ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;
let keys = {};
let scoreEl = document.getElementById('score');
let livesEl = document.getElementById('lives');
const restartBtn = document.getElementById('restart');
const touchControls = document.getElementById('touchControls');

addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);
restartBtn.onclick = () => init();

function showTouchIfMobile(){
  if (/Android|iPhone|iPad|Mobile/i.test(navigator.userAgent)) {
    touchControls.style.display = 'flex';
  }
}
showTouchIfMobile();

/* Touch buttons */
['Up','Left','Right','Down'].forEach(d=>{
  const b = document.getElementById('btn'+d);
  b.addEventListener('touchstart', ()=> keys[d.toLowerCase()] = true);
  b.addEventListener('touchend', ()=> keys[d.toLowerCase()] = false);
});
const btnKill = document.getElementById('btnKill');
btnKill.addEventListener('touchstart', ()=> keys['e'] = true);
btnKill.addEventListener('touchend', ()=> keys['e'] = false);

/* Utility */
function clamp(v,a,b){ return Math.max(a,Math.min(b,v)); }
function length(x,y){ return Math.hypot(x,y); }
function angleBetween(ax,ay,bx,by){ return Math.atan2(by-ay,bx-ax); }
function deg(d){ return d * 180/Math.PI; }
function rad(d){ return d * Math.PI/180; }

/* World objects */
let world;
function createWorld(){
  const w = {
    player: { x:80, y: H/2, r:12, speed:160, facing:0, alive:true },
    enemies: [],
    walls: [],
    score:0,
    lives:1
  };
  // Houses (walls) - rectangles that block vision & movement
  w.walls.push({x:300,y:120,w:120,h:120});
  w.walls.push({x:520,y:360,w:140,h:90});
  w.walls.push({x:720,y:150,w:100,h:160});
  w.walls.push({x:360,y:420,w:120,h:100});

  // Enemy factory: each enemy has x,y,dir,coneAngle,range,waypoints,wpIdx,alive
  const en = (x,y,waypoints, coneDeg=60, range=220, speed=40) => ({
    x,y, r:14, waypoints, wpIdx:0, speed, dir:0, cone:rad(coneDeg), range, alive:true, lastSeen:false
  });

  // place 4 enemies with simple patrol paths
  w.enemies.push(en(420,80, [{x:420,y:80},{x:420,y:200}], 70, 260, 30));
  w.enemies.push(en(640,280, [{x:640,y:280},{x:760,y:280}], 70, 240, 25));
  w.enemies.push(en(200,440, [{x:200,y:440},{x:340,y:440}], 60, 220, 30));
  w.enemies.push(en(820,440, [{x:820,y:440},{x:820,y:300}], 80, 260, 20));

  return w;
}

/* Line of sight check (segment vs walls) */
function segmentIntersectsRect(x1,y1,x2,y2, rect){
  // Liang-Barsky or simple check: check intersection with 4 edges
  const lines = [
    [rect.x,rect.y, rect.x+rect.w, rect.y],
    [rect.x+rect.w,rect.y, rect.x+rect.w, rect.y+rect.h],
    [rect.x+rect.w,rect.y+rect.h, rect.x, rect.y+rect.h],
    [rect.x,rect.y+rect.h, rect.x, rect.y]
  ];
  for (let l of lines){
    if (segSegIntersect(x1,y1,x2,y2, l[0],l[1],l[2],l[3])) return true;
  }
  return false;
}
function segSegIntersect(x1,y1,x2,y2,x3,y3,x4,y4){
  // quick segment intersection
  const denom = (y4-y3)*(x2-x1) - (x4-x3)*(y2-y1);
  if (Math.abs(denom) < 1e-6) return false;
  const ua = ((x4-x3)*(y1-y3) - (y4-y3)*(x1-x3)) / denom;
  const ub = ((x2-x1)*(y1-y3) - (y2-y1)*(x1-x3)) / denom;
  return ua>=0 && ua<=1 && ub>=0 && ub<=1;
}

/* Update & AI */
function update(dt){
  if (!world.player.alive) return;
  // player movement
  let mvx=0,mvy=0;
  if (keys['a']||keys['arrowleft']) mvx -= 1;
  if (keys['d']||keys['arrowright']) mvx += 1;
  if (keys['w']||keys['arrowup']) mvy -= 1;
  if (keys['s']||keys['arrowdown']) mvy += 1;
  const len = Math.hypot(mvx,mvy);
  if (len>0){ mvx/=len; mvy/=len; world.player.facing = Math.atan2(mvy,mvx); }
  world.player.x += mvx * world.player.speed * dt;
  world.player.y += mvy * world.player.speed * dt;
  // clamp inside canvas and don't walk through walls (simple pushback)
  world.player.x = clamp(world.player.x, 20, W-20);
  world.player.y = clamp(world.player.y, 20, H-20);
  for (let wall of world.walls){
    // circle vs rect pushout
    const px = clamp(world.player.x, wall.x, wall.x+wall.w);
    const py = clamp(world.player.y, wall.y, wall.y+wall.h);
    const dx = world.player.x - px, dy = world.player.y - py;
    const d2 = dx*dx + dy*dy;
    if (d2 < world.player.r*world.player.r){
      const d = Math.sqrt(d2) || 0.001;
      const overlap = world.player.r - d;
      world.player.x += (dx/d) * overlap;
      world.player.y += (dy/d) * overlap;
    }
  }

  // enemies update: patrol and detection
  for (let en of world.enemies){
    if (!en.alive) continue;
    const wp = en.waypoints[en.wpIdx];
    const dx = wp.x - en.x, dy = wp.y - en.y;
    const dist = Math.hypot(dx,dy);
    if (dist < 4) en.wpIdx = (en.wpIdx+1) % en.waypoints.length;
    else {
      en.x += (dx/dist) * en.speed * dt;
      en.y += (dy/dist) * en.speed * dt;
    }
    // direction facing toward next waypoint (or toward player if recently saw)
    en.dir = Math.atan2(dy,dx);

    // Detection: check if player inside cone and within range and clear LOS
    const toPlayerX = world.player.x - en.x, toPlayerY = world.player.y - en.y;
    const dPlayer = Math.hypot(toPlayerX,toPlayerY);
    const angToPlayer = Math.atan2(toPlayerY,toPlayerX);
    let angDiff = Math.abs(((angToPlayer - en.dir + Math.PI) % (2*Math.PI)) - Math.PI);
    const inCone = angDiff <= en.cone/2 && dPlayer <= en.range;
    // check line of sight (no wall in between)
    let los = true;
    if (inCone){
      for (let wall of world.walls){
        if (segmentIntersectsRect(en.x,en.y, world.player.x, world.player.y, wall)){
          los = false; break;
        }
      }
    } else los = false;

    if (los){
      // enemy sees player -> instant death
      world.player.alive = false;
      playDeathSound();
      setTimeout(()=> alert('ჩაამჩნია! მოკვდი — Game Over'), 50);
    }
  }

  // stealth kill (E) - must be close and be behind enemy (angle condition)
  if (keys['e']){
    for (let en of world.enemies){
      if (!en.alive) continue;
      const dx = en.x - world.player.x, dy = en.y - world.player.y;
      const dist = Math.hypot(dx,dy);
      if (dist <= 36){
        // determine if player is behind enemy: angle between enemy's facing and vector from enemy to player > 140deg
        const angEnemyFacing = en.dir;
        const angFromEnemyToPlayer = Math.atan2(world.player.y - en.y, world.player.x - en.x);
        let diff = Math.abs(((angFromEnemyToPlayer - angEnemyFacing + Math.PI) % (2*Math.PI)) - Math.PI);
        // if diff is near PI (i.e., player is roughly behind)
        if (diff > rad(120)){
          // also ensure player is NOT in cone (so stealth approach works)
          // basic LOS check: no walls between player and enemy
          let blocked = false;
          for (let w of world.walls){
            if (segmentIntersectsRect(world.player.x,world.player.y,en.x,en.y,w)){ blocked = true; break; }
          }
          if (!blocked){
            en.alive = false;
            world.score += 1;
            scoreEl.innerText = world.score;
            playStealthKillSound();
            // small "push" to show action
            world.player.x -= Math.cos(world.player.facing)*6;
            world.player.y -= Math.sin(world.player.facing)*6;
          }
        }
      }
    }
    // avoid holding key to spam kills: simple cooldown
    keys['e'] = false;
  }
}

/* Simple sounds */
function playStealthKillSound(){
  try{
    const ctxAudio = new (window.AudioContext || window.webkitAudioContext)();
    const o = ctxAudio.createOscillator();
    const g = ctxAudio.createGain();
    o.type = 'sine'; o.frequency.value = 880;
    o.connect(g); g.connect(ctxAudio.destination); g.gain.value = 0.02;
    o.start(); o.stop(ctxAudio.currentTime + 0.08);
  }catch(e){}
}
function playDeathSound(){
  try{
    const ctxAudio = new (window.AudioContext || window.webkitAudioContext)();
    const o = ctxAudio.createOscillator();
    const g = ctxAudio.createGain();
    o.type = 'sawtooth'; o.frequency.value = 120;
    o.connect(g); g.connect(ctxAudio.destination); g.gain.value = 0.08;
    o.start(); o.frequency.exponentialRampToValueAtTime(40, ctxAudio.currentTime + 0.25);
    o.stop(ctxAudio.currentTime + 0.25);
  }catch(e){}
}

/* Render */
function draw(){
  // background grass gradient
  ctx.clearRect(0,0,W,H);
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0,'#7cc66a'); g.addColorStop(1,'#3b7f47');
  ctx.fillStyle = g; ctx.fillRect(0,0,W,H);

  // draw houses (walls)
  for (let wall of world.walls){
    ctx.fillStyle = '#8b5e3c';
    ctx.fillRect(wall.x, wall.y, wall.w, wall.h);
    ctx.strokeStyle = '#5a3f2b';
    ctx.strokeRect(wall.x, wall.y, wall.w, wall.h);
  }

  // draw enemies vision cones FIRST (semi-transparent)
  for (let en of world.enemies){
    if (!en.alive) continue;
    // cone polygon
    const cx = en.x, cy = en.y;
    const left = en.dir - en.cone/2, right = en.dir + en.cone/2;
    const lx = cx + Math.cos(left)*en.range, ly = cy + Math.sin(left)*en.range;
    const rx = cx + Math.cos(right)*en.range, ry = cy + Math.sin(right)*en.range;
    ctx.beginPath();
    ctx.moveTo(cx,cy);
    ctx.lineTo(lx,ly);
    // draw arc (approx)
    const steps = 20;
    for (let i=0;i<=steps;i++){
      const a = left + (i/steps)*(right-left);
      ctx.lineTo(cx + Math.cos(a)*en.range, cy + Math.sin(a)*en.range);
    }
    ctx.closePath();
    ctx.fillStyle = 'rgba(200,40,40,0.12)';
    ctx.fill();
  }

  // draw enemies (front)
  for (let en of world.enemies){
    if (!en.alive) continue;
    // body
    ctx.save();
    ctx.translate(en.x,en.y);
    ctx.rotate(en.dir);
    ctx.fillStyle = '#d3d3d3';
    ctx.beginPath(); ctx.rect(-en.r,-en.r,en.r*2,en.r*2); ctx.fill();
    // head
    ctx.fillStyle = '#c57b7b'; ctx.beginPath(); ctx.arc(0,-en.r-6,6,0,Math.PI*2); ctx.fill();
    // gun as a small rectangle in front
    ctx.fillStyle = '#222';
    ctx.fillRect(en.r-6, -4, 18, 6);
    ctx.restore();
  }

  // draw player (on top)
  const p = world.player;
  if (p.alive){
    ctx.save();
    ctx.translate(p.x,p.y);
    // body
    ctx.fillStyle = '#4a8cff';
    ctx.beginPath(); ctx.arc(0,0,p.r,0,Math.PI*2); ctx.fill();
    // AKM on back (simple)
    ctx.save();
    ctx.rotate(p.facing);
    ctx.fillStyle = '#2b2b2b';
    ctx.fillRect(-p.r-8, -3, 22, 6); // stock+body
    ctx.fillRect(-p.r-8, -7, 6, 14); // handle/back
    ctx.restore();
    // direction marker
    ctx.fillStyle = '#fff';
    ctx.fillRect(Math.cos(p.facing)*(p.r+6)-3, Math.sin(p.facing)*(p.r+6)-3,6,6);
    ctx.restore();
  } else {
    // dead indicator
    ctx.fillStyle = 'rgba(0,0,0,0.6)';
    ctx.fillRect(0, H-36, W, 36);
    ctx.fillStyle = '#fff';
    ctx.font = '22px sans-serif';
    ctx.fillText('You were seen — Game Over. Press Restart.', 18, H-12);
  }

  // HUD overlay: info about enemies alive
  ctx.fillStyle = '#fff';
  ctx.font = '14px sans-serif';
  const aliveCount = world.enemies.filter(e=>e.alive).length;
  ctx.fillText('Enemies remaining: ' + aliveCount, 12, 22);
}

/* Main loop */
let last = performance.now();
function frame(now){
  const dt = Math.min(1/30, (now-last)/1000);
  last = now;
  update(dt);
  draw();
  requestAnimationFrame(frame);
}

/* init & start */
function init(){
  world = createWorld();
  scoreEl.innerText = world.score;
  livesEl.innerText = world.lives;
  world.player.alive = true;
  // reset keys
  keys['e'] = false;
}
init();
requestAnimationFrame(frame);

</script>
</body>
</html>
