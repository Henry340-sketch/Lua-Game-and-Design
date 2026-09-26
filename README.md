<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>SOLDIER VS ZOMBIES - Pro Edition</title>
<style>
  *{margin:0;padding:0;box-sizing:border-box;font-family: 'Segoe UI', sans-serif;}
  body{background:#0a0a0f;overflow:hidden;display:flex;justify-content:center;align-items:center;height:100vh;}
  #gameContainer{position:relative; width:1000px; height:600px; box-shadow:0 0 50px rgba(0,255,100,0.2); border:2px solid #1f3d2b; border-radius:12px; overflow:hidden;}
  canvas{display:block;background: radial-gradient(#1a2a1f, #0e1410);}
  #ui{position:absolute;top:0;left:0;width:100%;padding:15px 20px;display:flex;justify-content:space-between;color:white;pointer-events:none;z-index:10;}
  #ui div{pointer-events:auto;}
 .hud{background:rgba(0,0,0,0.6); padding:8px 15px; border-radius:20px; border:1px solid #2a5a3a; font-weight:bold; font-size:14px; margin-right:10px; display:inline-block;}
 .btn{background:#ff3b30; border:none; color:white; padding:8px 18px; border-radius:20px; cursor:pointer; font-weight:bold; transition:0.2s;}
 .btn:hover{background:#ff1a0e; transform:scale(1.05);}
 .btn-green{background:#2ecc71;}.btn-green:hover{background:#27ae60;}
  #overlay{position:absolute; inset:0; background:rgba(0,0,0,0.85); display:flex; flex-direction:column; justify-content:center; align-items:center; color:white; z-index:20; text-align:center;}
  #overlay h1{font-size:48px; letter-spacing:3px; color:#2ecc71; text-shadow:0 0 20px #2ecc71;}
  #overlay p{margin:10px 0; color:#aaa;}
 .menuBtn{margin-top:15px; padding:12px 30px; font-size:16px; border-radius:30px; border:none; cursor:pointer; font-weight:bold;}
  #healthBar{width:120px; height:10px; background:#333; border-radius:10px; overflow:hidden; display:inline-block; vertical-align:middle; margin-left:8px;}
  #healthFill{height:100%; width:100%; background:linear-gradient(90deg,#2ecc71,#f1c40f,#e74c3c); transition:width 0.2s;}
</style>
</head>
<body>
<div id="gameContainer">
  <canvas id="game" width="1000" height="600"></canvas>
  <div id="ui">
    <div>
      <span class="hud">SCORE: <span id="score">0</span></span>
      <span class="hud">LEVEL: <span id="level">1</span></span>
      <span class="hud">HEALTH: <div id="healthBar"><div id="healthFill"></div></div></span>
    </div>
    <div>
      <span class="hud">ZOMBIES: <span id="zCount">0</span></span>
      <button class="btn" onclick="exitGame()">EXIT</button>
    </div>
  </div>
  <div id="overlay">
    <h1 id="oTitle">SOLDIER VS ZOMBIES</h1>
    <p id="oSub">Professional Survival Shooter</p>
    <p style="max-width:500px; font-size:13px; line-height:1.6;">WASD to Move | Mouse to Aim | Left Click / SPACE to Shoot | ESC to Pause</p>
    <button class="menuBtn btn-green" id="startBtn" onclick="startGame(1)">START GAME</button>
    <p id="finalScore" style="margin-top:20px; font-size:20px; color:#f1c40f;"></p>
  </div>
</div>

<script>
// ===== PROFESSIONAL GAME ENGINE =====
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const W = canvas.width, H = canvas.height;

let player, zombies = [], bullets = [], particles = [];
let score = 0, level = 1, zombiesToSpawn = 0, gameState = 'menu'; // menu, playing, paused, levelComplete, gameOver
let keys = {}, mouse = {x:W/2, y:H/2, down:false}, shootCooldown = 0;

class Player{
  constructor(){ this.x=W/2; this.y=H/2; this.speed=3.2; this.radius=18; this.angle=0; this.health=100; }
  update(){
    if(keys['w']||keys['ArrowUp']) this.y-=this.speed;
    if(keys['s']||keys['ArrowDown']) this.y+=this.speed;
    if(keys['a']||keys['ArrowLeft']) this.x-=this.speed;
    if(keys['d']||keys['ArrowRight']) this.x+=this.speed;
    this.x=Math.max(20,Math.min(W-20,this.x));
    this.y=Math.max(20,Math.min(H-20,this.y));
    this.angle=Math.atan2(mouse.y-this.y, mouse.x-this.x);
    if(shootCooldown>0) shootCooldown-=0.016;
  }
  draw(){
    ctx.save(); ctx.translate(this.x,this.y); ctx.rotate(this.angle);
    ctx.fillStyle='#2ecc71'; ctx.beginPath(); ctx.roundRect(-18,-12,36,24,5); ctx.fill();
    ctx.fillStyle='#111'; ctx.fillRect(10,-3,28,6);
    ctx.fillStyle='#f5d6a0'; ctx.beginPath(); ctx.arc( -8,0,8,0,Math.PI*2); ctx.fill();
    ctx.restore();
  }
}
class Zombie{
  constructor(x,y,lvl){
    this.x=x; this.y=y;
    this.speed=0.6 + Math.random()*0.8 + (lvl*0.15);
    this.radius=16; this.health=30;
  }
  update(target){ let a=Math.atan2(target.y-this.y, target.x-this.x); this.x+=Math.cos(a)*this.speed; this.y+=Math.sin(a)*this.speed; }
  draw(){
    ctx.fillStyle='#7ab648'; ctx.beginPath(); ctx.arc(this.x,this.y,this.radius,0,Math.PI*2); ctx.fill();
    ctx.fillStyle='#000'; ctx.beginPath(); ctx.arc(this.x+4,this.y-3,3,0,Math.PI*2); ctx.fill();
  }
}
class Bullet{ constructor(x,y,a){this.x=x;this.y=y;this.angle=a; this.speed=9;} update(){this.x+=Math.cos(this.angle)*this.speed; this.y+=Math.sin(this.angle)*this.speed;} draw(){ctx.fillStyle='#ffeb3b'; ctx.beginPath(); ctx.arc(this.x,this.y,4,0,Math.PI*2); ctx.fill();}}

function spawnZombie(){
  if(zombiesToSpawn<=0) return;
  let side=Math.floor(Math.random()*4), x,y;
  if(side==0){x=-30; y=Math.random()*H} else if(side==1){x=W+30; y=Math.random()*H}
  else if(side==2){x=Math.random()*W; y=-30} else {x=Math.random()*W; y=H+30}
  zombies.push(new Zombie(x,y,level)); zombiesToSpawn--;
}

let spawnTimer=0;
function gameLoop(){
  requestAnimationFrame(gameLoop);
  if(gameState!=='playing') return;

  // Clear with slight trail effect
  ctx.fillStyle='rgba(14,20,16,0.3)'; ctx.fillRect(0,0,W,H);

  player.update();

  spawnTimer-=0.016;
  if(spawnTimer<=0 && zombies.length<20){ spawnZombie(); spawnTimer = Math.max(0.2, 1.5 - level*0.1); }

  zombies.forEach(z=>z.update(player));
  bullets.forEach(b=>b.update());

  // Collisions
  for(let i=bullets.length-1;i>=0;i--){
    for(let j=zombies.length-1;j>=0;j--){
      let dx=bullets[i].x-zombies[j].x, dy=bullets[i].y-zombies[j].y;
      if(Math.sqrt(dx*dx+dy*dy) < zombies[j].radius+4){
        // Particle effect
        for(let k=0;k<8;k++) particles.push({x:zombies[j].x,y:zombies[j].y,vx:(Math.random()-0.5)*6, vy:(Math.random()-0.5)*6, life:0.5});
        zombies.splice(j,1); bullets.splice(i,1); score+=100; break;
      }
    }
  }

  // Player hit
  zombies.forEach(z=>{
    let dx=player.x-z.x, dy=player.y-z.y;
    if(Math.sqrt(dx*dx+dy*dy) < 28){ player.health-=0.4; if(player.health<=0) setGameState('gameOver'); }
  });

  bullets = bullets.filter(b=> b.x>-20 && b.x<W+20 && b.y>-20 && b.y<H+20);

  player.draw(); zombies.forEach(z=>z.draw()); bullets.forEach(b=>b.draw());

  // Particles
  particles.forEach((p,i)=>{ p.x+=p.vx; p.y+=p.vy; p.life-=0.02; ctx.fillStyle=`rgba(255,100,50,${p.life})`; ctx.fillRect(p.x,p.y,3,3); });
  particles = particles.filter(p=>p.life>0);

  // UI Update
  document.getElementById('score').innerText=score;
  document.getElementById('level').innerText=level;
  document.getElementById('zCount').innerText=zombies.length + zombiesToSpawn;
  document.getElementById('healthFill').style.width=player.health+'%';

  // Level Complete
  if(zombiesToSpawn==0 && zombies.length==0){ setGameState('levelComplete'); }
}

function setGameState(state){
  gameState=state;
  const overlay=document.getElementById('overlay');
  const title=document.getElementById('oTitle');
  const sub=document.getElementById('oSub');
  const btn=document.getElementById('startBtn');
  const final=document.getElementById('finalScore');

  if(state=='playing'){ overlay.style.display='none'; return; }

  overlay.style.display='flex';
  if(state=='menu'){ title.innerText='SOLDIER VS ZOMBIES'; sub.innerText='Professional Survival Shooter'; btn.innerText='START GAME'; btn.onclick=()=>startGame(1); final.innerText='';}
  if(state=='paused'){ title.innerText='PAUSED'; sub.innerText='Game Paused'; btn.innerText='RESUME'; btn.onclick=()=>setGameState('playing'); final.innerText='Score: '+score; }
  if(state=='levelComplete'){ title.innerText=`LEVEL ${level} CLEARED!`; sub.innerText=`Get ready for Level ${level+1} - Harder Zombies`; btn.innerText=`GO TO LEVEL ${level+1}`; btn.onclick=()=>startGame(level+1); btn.className='menuBtn btn-green'; final.innerText=`Score: ${score}`; }
  if(state=='gameOver'){ title.innerText='YOU DIED'; title.style.color='#ff3b30'; sub.innerText='Zombies got you'; btn.innerText='RESTART'; btn.onclick=()=>{score=0; startGame(1);}; final.innerText=`Final Score: ${score}`; }
}

function startGame(lvl){
  level=lvl; player=new Player(); zombies=[]; bullets=[]; particles=[];
  zombiesToSpawn = 8 + (level * 5);
  if(lvl==1) score=0;
  document.getElementById('oTitle').style.color='#2ecc71';
  setGameState('playing');
}

function exitGame(){
  if(confirm('Exit Game? Your progress will be lost.')){ setGameState('menu'); score=0; }
}

// Inputs
window.addEventListener('keydown', e=>{
  keys[e.key.toLowerCase()]=true;
  if(e.code=='Space' && gameState=='playing' && shootCooldown<=0){ bullets.push(new Bullet(player.x, player.y, player.angle)); shootCooldown=12; }
  if(e.key=='Escape'){ if(gameState=='playing') setGameState('paused'); else if(gameState=='paused') setGameState('playing'); }
});
window.addEventListener('keyup', e=> keys[e.key.toLowerCase()]=false);
canvas.addEventListener('mousemove', e=>{ const r=canvas.getBoundingClientRect(); mouse.x=e.clientX-r.left; mouse.y=e.clientY-r.top; });
canvas.addEventListener('mousedown', e=>{ if(gameState=='playing' && shootCooldown<=0){ bullets.push(new Bullet(player.x, player.y, player.angle)); shootCooldown=12; } });

gameLoop();
</script>
</body>
</html>
