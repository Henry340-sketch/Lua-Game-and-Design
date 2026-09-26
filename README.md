<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>Soldier vs Zombies - Real Pics</title>
<style>
*{margin:0;padding:0;box-sizing:border-box;font-family:sans-serif;touch-action:none;}
body{background:#0d1117;display:flex;justify-content:center;align-items:center;height:100vh;overflow:hidden;}
#gameContainer{position:relative;width:1000px;max-width:100vw;height:600px;max-height:100vh;border:3px solid #2ecc71;border-radius:15px;overflow:hidden;background:#162016;}
canvas{display:block;width:100%;height:100%;}
#ui{position:absolute;top:0;left:0;width:100%;padding:12px;display:flex;justify-content:space-between;color:white;z-index:5;}
.hud{background:rgba(0,0,0,0.7);padding:6px 14px;border-radius:20px;font-weight:bold;font-size:13px;margin-right:6px;border:1px solid #2a5a3a;}
.btn{pointer-events:auto;background:#ff3b30;border:none;color:white;padding:7px 16px;border-radius:20px;font-weight:bold;cursor:pointer;}
#controls{position:absolute;bottom:15px;left:15px;z-index:5;display:flex;gap:8px;flex-wrap:wrap;width:130px;}
.cbtn{width:55px;height:55px;background:rgba(255,255,255,0.15);border:2px solid rgba(255,255,255,0.3);border-radius:12px;color:white;font-size:20px;font-weight:bold;backdrop-filter:blur(5px);}
.cbtn:active{background:rgba(46,204,113,0.5);}
#overlay{position:absolute;inset:0;background:rgba(0,0,0,0.88);display:flex;flex-direction:column;justify-content:center;align-items:center;color:white;z-index:20;text-align:center;}
#overlay h1{font-size:42px;color:#2ecc71;text-shadow:0 0 15px #2ecc71;}
.menuBtn{margin-top:15px;padding:12px 35px;font-size:17px;border-radius:30px;border:none;cursor:pointer;font-weight:bold;background:#2ecc71;color:#000;}
#healthBar{width:100px;height:10px;background:#333;border-radius:10px;overflow:hidden;display:inline-block;vertical-align:middle;margin-left:6px;}
#healthFill{height:100%;width:100%;background:#2ecc71;transition:width 0.2s;}
</style>
</head>
<body>
<div id="gameContainer">
<canvas id="game" width="1000" height="600"></canvas>
<div id="ui">
  <div>
    <span class="hud">SCORE: <span id="score">0</span></span>
    <span class="hud">LVL: <span id="level">1</span></span>
    <span class="hud">HP: <div id="healthBar"><div id="healthFill"></div></div></span>
  </div>
  <div><span class="hud">🧟 <span id="zCount">0</span></span> <button class="btn" onclick="exitGame()">EXIT</button></div>
</div>
<div id="controls">
  <button class="cbtn" style="margin-left:32px;" ontouchstart="keys['w']=true" ontouchend="keys['w']=false" onmousedown="keys['w']=true" onmouseup="keys['w']=false">↑</button>
  <button class="cbtn" ontouchstart="keys['a']=true" ontouchend="keys['a']=false" onmousedown="keys['a']=true" onmouseup="keys['a']=false">←</button>
  <button class="cbtn" ontouchstart="keys['s']=true" ontouchend="keys['s']=false" onmousedown="keys['s']=true" onmouseup="keys['s']=false">↓</button>
  <button class="cbtn" ontouchstart="keys['d']=true" ontouchend="keys['d']=false" onmousedown="keys['d']=true" onmouseup="keys['d']=false">→</button>
  <button class="cbtn" style="background:rgba(255,59,48,0.5); width:120px;" onclick="tryShoot()">🔫 SHOOT</button>
</div>
<div id="overlay">
  <h1 id="oTitle">SOLDIER VS ZOMBIES</h1>
  <p id="oSub" style="color:#aaa;margin:10px;">Move with buttons / WASD<br>Tap a ZOMBIE to auto-aim & shoot</p>
  <button class="menuBtn" id="startBtn" onclick="startGame(1)">START BATTLE</button>
  <p id="finalScore" style="margin-top:15px;color:#f1c40f;font-size:18px;"></p>
</div>
</div>

<script>
const canvas=document.getElementById('game'), ctx=canvas.getContext('2d');
const W=1000,H=600;
let player,zombies=[],bullets=[],particles=[];
let score=0,level=1,zombiesToSpawn=0,gameState='menu';
let keys={},mouse={x:W/2,y:H/2},shootCooldown=0,spawnTimer=0;

// REAL IMAGES - preloaded
const soldierImg=new Image(); soldierImg.src='https://cdn-icons-png.flaticon.com/512/1995/1995539.png'; // soldier top view
const zombieImg=new Image(); zombieImg.src='https://cdn-icons-png.flaticon.com/512/4908/4908117.png'; // zombie face
const zombieImg2=new Image(); zombieImg2.src='https://cdn-icons-png.flaticon.com/512/3460/3460333.png';

class Player{
 constructor(){this.x=W/2;this.y=H/2;this.speed=3.5;this.radius=22;this.angle=0;this.health=100;}
 update(){
  if(keys['w']||keys['arrowup']) this.y-=this.speed;
  if(keys['s']||keys['arrowdown']) this.y+=this.speed;
  if(keys['a']||keys['arrowleft']) this.x-=this.speed;
  if(keys['d']||keys['arrowright']) this.x+=this.speed;
  this.x=Math.max(25,Math.min(W-25,this.x)); this.y=Math.max(25,Math.min(H-25,this.y));
  this.angle=Math.atan2(mouse.y-this.y, mouse.x-this.x);
  if(shootCooldown>0) shootCooldown-=0.016;
 }
 draw(){
  ctx.save(); ctx.translate(this.x,this.y); ctx.rotate(this.angle);
  if(soldierImg.complete){ ctx.drawImage(soldierImg,-24,-24,48,48); }
  else{ ctx.fillStyle='#2ecc71'; ctx.fillRect(-18,-12,36,24); }
  ctx.restore();
  // shadow
  ctx.fillStyle='rgba(0,0,0,0.3)'; ctx.beginPath(); ctx.ellipse(this.x,this.y+20,18,6,0,0,Math.PI*2); ctx.fill();
 }
}
class Zombie{
 constructor(x,y,lvl){
  this.x=x;this.y=y;this.speed=0.7+Math.random()*0.9+(lvl*0.2); this.radius=24; this.img=Math.random()>0.5?zombieImg:zombieImg2;
 }
 update(t){ let a=Math.atan2(t.y-this.y,t.x-this.x); this.x+=Math.cos(a)*this.speed; this.y+=Math.sin(a)*this.speed; }
 draw(){
  ctx.save(); ctx.translate(this.x,this.y);
  // glow
  ctx.shadowColor='#ff0000'; ctx.shadowBlur=10;
  if(this.img.complete) ctx.drawImage(this.img,-22,-22,44,44);
  else { ctx.fillStyle='#7ab648'; ctx.beginPath(); ctx.arc(0,0,this.radius,0,Math.PI*2); ctx.fill(); }
  ctx.restore();
 }
}
class Bullet{
 constructor(x,y,a){this.x=x;this.y=y;this.angle=a;this.speed=10;}
 update(){this.x+=Math.cos(this.angle)*this.speed; this.y+=Math.sin(this.angle)*this.speed;}
 draw(){ctx.fillStyle='#ffeb3b'; ctx.shadowColor='#ffeb3b'; ctx.shadowBlur=10; ctx.beginPath(); ctx.arc(this.x,this.y,5,0,Math.PI*2); ctx.fill(); ctx.shadowBlur=0;}
}

function tryShoot(targetX=null,targetY=null){
 if(gameState!=='playing' || shootCooldown>0) return;
 let angle=player.angle;
 if(targetX!==null){ angle=Math.atan2(targetY-player.y, targetX-player.x); player.angle=angle; }
 bullets.push(new Bullet(player.x+Math.cos(angle)*25, player.y+Math.sin(angle)*25, angle));
 shootCooldown=0.25; // 0.25 sec cooldown - FIXED!
 // muzzle flash
 particles.push({x:player.x+Math.cos(angle)*30,y:player.y+Math.sin(angle)*30,vx:0,vy:0,life:0.15,color:'#ffeb3b'});
}

function spawnZombie(){
 if(zombiesToSpawn<=0) return;
 let s=Math.floor(Math.random()*4),x,y;
 if(s==0){x=-40;y=Math.random()*H}else if(s==1){x=W+40;y=Math.random()*H}else if(s==2){x=Math.random()*W;y=-40}else{x=Math.random()*W;y=H+40}
 zombies.push(new Zombie(x,y,level)); zombiesToSpawn--;
}

function gameLoop(){
 requestAnimationFrame(gameLoop);
 if(gameState!=='playing'){ return; }
 ctx.fillStyle='#1e2f23'; ctx.fillRect(0,0,W,H);
 // grid
 ctx.strokeStyle='rgba(255,255,255,0.03)'; for(let i=0;i<W;i+=50){ctx.beginPath();ctx.moveTo(i,0);ctx.lineTo(i,H);ctx.stroke();} for(let i=0;i<H;i+=50){ctx.beginPath();ctx.moveTo(0,i);ctx.lineTo(W,i);ctx.stroke();}

 player.update();
 spawnTimer-=0.016; if(spawnTimer<=0 && zombies.length<25){ spawnZombie(); spawnTimer=Math.max(0.2,1.6-level*0.12); }

 zombies.forEach(z=>z.update(player)); bullets.forEach(b=>b.update());

 // Bullet vs Zombie - when you PRESS the zombie, it dies
 for(let i=bullets.length-1;i>=0;i--){
  for(let j=zombies.length-1;j>=0;j--){
   let dx=bullets[i].x-zombies[j].x, dy=bullets[i].y-zombies[j].y;
   if(Math.sqrt(dx*dx+dy*dy)<28){
    for(let k=0;k<12;k++) particles.push({x:zombies[j].x,y:zombies[j].y,vx:(Math.random()-0.5)*7,vy:(Math.random()-0.5)*7,life:0.6,color:'#ff3b30'});
    zombies.splice(j,1); bullets.splice(i,1); score+=150; break;
   }
  }
 }
 bullets=bullets.filter(b=>b.x>-30&&b.x<W+30&&b.y>-30&&b.y<H+30);

 zombies.forEach(z=>{
  let dx=player.x-z.x, dy=player.y-z.y;
  if(Math.sqrt(dx*dx+dy*dy)<32){ player.health-=0.5; if(player.health<=0) setState('gameOver'); }
 });

 player.draw(); zombies.forEach(z=>z.draw()); bullets.forEach(b=>b.draw());
 particles.forEach(p=>{p.x+=p.vx;p.y+=p.vy;p.life-=0.02; ctx.fillStyle=p.color; ctx.globalAlpha=p.life; ctx.fillRect(p.x,p.y,4,4); ctx.globalAlpha=1;});
 particles=particles.filter(p=>p.life>0);

 document.getElementById('score').innerText=score;
 document.getElementById('level').innerText=level;
 document.getElementById('zCount').innerText=zombies.length+zombiesToSpawn;
 document.getElementById('healthFill').style.width=player.health+'%';

 if(zombiesToSpawn==0 && zombies.length==0) setState('levelComplete');
}

function setState(s){
 gameState=s; const ov=document.getElementById('overlay'), t=document.getElementById('oTitle'), sub=document.getElementById('oSub'), btn=document.getElementById('startBtn'), fs=document.getElementById('finalScore');
 if(s=='playing'){ov.style.display='none';return;}
 ov.style.display='flex';
 if(s=='menu'){t.innerText='SOLDIER VS ZOMBIES';t.style.color='#2ecc71';sub.innerHTML='Move with WASD / Buttons<br><b>TAP ANY ZOMBIE 🧟 to shoot him</b>';btn.innerText='START BATTLE';btn.onclick=()=>startGame(1);fs.innerText='';}
 if(s=='paused'){t.innerText='PAUSED';sub.innerText='Game Paused';btn.innerText='RESUME';btn.onclick=()=>setState('playing');fs.innerText='Score: '+score;}
 if(s=='levelComplete'){t.innerText=`LEVEL ${level} CLEARED!`;sub.innerText=`Level ${level+1} incoming - Faster Zombies!`;btn.innerText=`GO TO LEVEL ${level+1}`;btn.onclick=()=>startGame(level+1);fs.innerText=`Score: ${score}`;}
 if(s=='gameOver'){t.innerText='MISSION FAILED';t.style.color='#ff3b30';sub.innerText='You were overrun';btn.innerText='RESTART';btn.onclick=()=>{score=0;startGame(1);};fs.innerText=`Final Score: ${score}`;}
}

function startGame(lvl){
 level=lvl; player=new Player(); zombies=[]; bullets=[]; particles=[]; zombiesToSpawn=6+(level*4); if(lvl==1)score=0; document.getElementById('oTitle').style.color='#2ecc71'; setState('playing');
}
function exitGame(){ if(confirm('Exit to menu?')) setState('menu'); }

// INPUTS - FIXED SHOOTING LOGIC
window.addEventListener('keydown',e=>{
 keys[e.key.toLowerCase()]=true;
 if(e.code=='Space') tryShoot();
 if(e.key=='Escape'){ if(gameState=='playing') setState('paused'); else if(gameState=='paused') setState('playing'); }
});
window.addEventListener('keyup',e=>keys[e.key.toLowerCase()]=false);

canvas.addEventListener('mousemove',e=>{const r=canvas.getBoundingClientRect(); mouse.x=(e.clientX-r.left)*(W/r.width); mouse.y=(e.clientY-r.top)*(H/r.height);});
canvas.addEventListener('mousedown',e=>{
 const r=canvas.getBoundingClientRect(); const x=(e.clientX-r.left)*(W/r.width); const y=(e.clientY-r.top)*(H/r.height);
 // Check if clicked on zombie
 let hitZombie=null; zombies.forEach(z=>{ if(Math.sqrt((x-z.x)**2+(y-z.y)**2)<35) hitZombie=z; });
 if(hitZombie){ tryShoot(hitZombie.x, hitZombie.y); } else { tryShoot(x,y); }
});
canvas.addEventListener('touchstart',e=>{
 const t=e.touches[0]; const r=canvas.getBoundingClientRect(); const x=(t.clientX-r.left)*(W/r.width); const y=(t.clientY-r.top)*(H/r.height);
 let hitZombie=null; zombies.forEach(z=>{ if(Math.sqrt((x-z.x)**2+(y-z.y)**2)<50) hitZombie=z; });
 if(hitZombie){ tryShoot(hitZombie.x, hitZombie.y); }
}, {passive:false});

gameLoop();
</script>
</body>
</html>
