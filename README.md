# zero_snake-
<!DOCTYPE html>
<html lang="ar">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Zero Snake — Chemsou Zero</title>
<style>
html,body{margin:0;padding:0;overflow:hidden;background:#07101a;}
canvas{display:block;background:linear-gradient(180deg,#07101a 0%,#071827 100%);}
#hud{position:absolute;top:10px;left:10px;color:white;font-family:sans-serif;z-index:10;}
#hud div{margin-bottom:5px;}
#joystick{position:absolute;bottom:20px;left:20px;width:120px;height:120px;border-radius:50%;background:rgba(255,255,255,0.08);touch-action:none;}
#joystickInner{position:absolute;width:60px;height:60px;border-radius:50%;background:rgba(255,255,255,0.3);top:30px;left:30px;}
#rButton{position:absolute;bottom:30px;right:30px;width:70px;height:70px;border-radius:50%;background:#ff6b35;color:white;font-weight:bold;font-size:20px;text-align:center;line-height:70px;user-select:none;}
#footerText{position:absolute;bottom:5px;width:100%;text-align:center;font-family:sans-serif;color:black;font-size:24px;}
#startButton{
    position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);
    background:#28a745;color:white;padding:20px 40px;font-size:24px;border-radius:12px;cursor:pointer;z-index:20;
}
#infoButton{
    position:absolute; bottom:120px; right:30px;
    background:white; color:black; padding:10px 15px;
    font-size:16px; border-radius:8px; cursor:pointer; z-index:20;
}
#infoBox{
    position:absolute; bottom:160px; right:30px;
    background:rgba(0,0,0,0.8); color:white; padding:15px;
    border-radius:10px; font-family:sans-serif; display:none; width:250px; z-index:20;
}
</style>
</head>
<body>
<canvas id="game"></canvas>
<div id="hud">
  <div>Score: <span id="score">0</span></div>
  <div>Level: <span id="level">1</span></div>
  <div>HP: <span id="hp">3</span></div>
</div>
<div id="joystick"><div id="joystickInner"></div></div>
<div id="rButton">R</div>
<div id="footerText">zero snake</div>
<div id="startButton">ابدأ اللعبة</div>

<!-- زر المعلومات -->
<div id="infoButton">Information</div>
<div id="infoBox">
    <strong>Zero Snake</strong><br>
    لعبة الثعبان الكلاسيكية مع ألوان ديناميكية.<br>
    مطور اللعبة: شمسو Zero
</div>

<script>
// ====== Zero Snake Game ======
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");
let W=window.innerWidth,H=window.innerHeight;
canvas.width=W;canvas.height=H;

const scoreEl=document.getElementById("score");
const levelEl=document.getElementById("level");
const hpEl=document.getElementById("hp");

const joystick=document.getElementById("joystick");
const inner=document.getElementById("joystickInner");
let joystickActive=false;
let joystickDir={x:0,y:0};
let joyStart={x:0,y:0};

joystick.addEventListener("touchstart",(e)=>{
    joystickActive=true;
    const t=e.touches[0];
    joyStart={x:t.clientX,y:t.clientY};
});
joystick.addEventListener("touchmove",(e)=>{
    if(!joystickActive)return;
    const t=e.touches[0];
    let dx=t.clientX-joyStart.x;
    let dy=t.clientY-joyStart.y;
    const maxDist=50;
    const mag=Math.hypot(dx,dy);
    if(mag>maxDist){dx=dx/mag*maxDist;dy=dy/mag*maxDist;}
    inner.style.transform=`translate(${dx}px,${dy}px)`;
    joystickDir.x=dx/maxDist;
    joystickDir.y=dy/maxDist;
});
// ====== تعديل: يبقى يتحرك بعد رفع الصبع ======
joystick.addEventListener("touchend",(e)=>{
    joystickActive=false;
    inner.style.transform="translate(0px,0px)";
    // joystickDir يبقى آخر اتجاه
});

const rButton=document.getElementById("rButton");
let rPressed=false;
rButton.addEventListener("touchstart",(e)=>{rPressed=true;});
rButton.addEventListener("touchend",(e)=>{rPressed=false;});

class Segment{constructor(x,y){this.pos={x,y};this.r=10;this.color="#61d0ff";}}
class Apple{constructor(){this.respawn();}respawn(){this.x=Math.random()*(W-60)+30;this.y=Math.random()*(H-60)+30;this.r=12;}draw(){ctx.beginPath();ctx.fillStyle="#ff4b4b";ctx.arc(this.x,this.y,this.r,0,Math.PI*2);ctx.fill();}}

class Obstacle{
    constructor(x,y,w,h){
        this.x=x; this.y=y; this.w=w; this.h=h;
        this.vx=(Math.random()*40-20);
        this.vy=(Math.random()*40-20);
    }
    draw(dt){
        this.x += this.vx*dt;
        this.y += this.vy*dt;
        if(this.x<0 || this.x+this.w>W) this.vx*=-1;
        if(this.y<0 || this.y+this.h>H) this.vy*=-1;
        ctx.fillStyle="rgba(200,50,50,0.4)";
        ctx.fillRect(this.x,this.y,this.w,this.h);
    }
    collide(px,py,radius){
        return px+radius>this.x&&px-radius<this.x+this.w&&py+radius>this.y&&py-radius<this.y+this.h;
    }
}

class Snake{
constructor(){this.head={x:W/2,y:H/2,vx:0,vy:0,speed:200};this.segs=[];for(let i=0;i<20;i++)this.segs.push(new Segment(this.head.x-i*8,this.head.y));this.maxSegs=200;this.hp=3;this.alive=true;}
update(dt){let tx=joystickDir.x,ty=joystickDir.y;let mag=Math.hypot(tx,ty);if(mag>0.01){tx/=mag;ty/=mag;}let sp=this.head.speed*(rPressed?1.8:1.0);this.head.vx=this.head.vx*(1-0.2)+tx*sp*0.2;this.head.vy=this.head.vy*(1-0.2)+ty*sp*0.2;this.head.x+=this.head.vx*dt;this.head.y+=this.head.vy*dt;this.head.x=Math.max(10,Math.min(W-10,this.head.x));this.head.y=Math.max(10,Math.min(H-10,this.head.y));let prev={x:this.head.x,y:this.head.y};for(let i=0;i<this.segs.length;i++){let s=this.segs[i];let dx=prev.x-s.pos.x,dy=prev.y-s.pos.y;let d=Math.hypot(dx,dy)||0.0001;let targetDist=10+Math.sin(performance.now()/200+i*0.5)*2;let t=Math.min(1,(d-targetDist)/d);s.pos.x+=dx*t;s.pos.y+=dy*t;prev=s.pos;let hue=(i*5+(performance.now()/30))%360;let light=50+10*Math.sin(performance.now()/500+i/2);s.color=`hsl(${hue},90%,${light}%)`;}if(rPressed&&this.segs.length>6)this.segs.pop();}
grow(n){for(let i=0;i<n;i++){let last=this.segs[this.segs.length-1];this.segs.push(new Segment(last.pos.x,last.pos.y));if(this.segs.length>this.maxSegs)this.segs.shift();}}
draw(){for(let i=this.segs.length-1;i>=0;i--){let s=this.segs[i];ctx.beginPath();ctx.fillStyle=s.color;ctx.arc(s.pos.x,s.pos.y,10,0,Math.PI*2);ctx.fill();}ctx.beginPath();ctx.fillStyle="white";ctx.arc(this.head.x,this.head.y,12,0,Math.PI*2);ctx.fill();}
checkSelfCollision(){for(let i=5;i<this.segs.length;i++){if(Math.hypot(this.head.x-this.segs[i].pos.x,this.head.y-this.segs[i].pos.y)<10){this.alive=false;audioDie.play();break;}}}
}

let snake, apple, obstacles, score, level;
const audioEat=new Audio("https://actions.google.com/sounds/v1/cartoon/clang_and_wobble.ogg");
const audioDie=new Audio("https://actions.google.com/sounds/v1/cartoon/metal_thud_and_bounce.ogg");
const audioLevel=new Audio("https://actions.google.com/sounds/v1/cartoon/wood_plank_flicks.ogg");

let startTime = null;

function initGame() {
    snake = new Snake();
    apple = new Apple();
    obstacles = [];
    for(let i=0;i<6;i++) obstacles.push(new Obstacle(Math.random()*(W-100), Math.random()*(H-100),40,20));
    score = 0;
    level = 1;
    updateUI();
}

function updateUI(){scoreEl.textContent=score;levelEl.textContent=level;hpEl.textContent=snake.hp;}

let last;
let gameRunning = false;

function loop(now){
    if(!gameRunning) return;
    if(!last) last = now;
    if(!startTime) startTime = now; 
    let dt=(now-last)/1000;last=now;

    ctx.clearRect(0,0,W,H);

    if(snake.alive){
        let elapsed = (now - startTime)/1000;

        // ====== الحركة فقط إذا اللاعب بدأ يتحرك الجويستيك ======
        if(elapsed > 3 || Math.hypot(joystickDir.x,joystickDir.y)>0.01){ 
            snake.update(dt);
            snake.checkSelfCollision();
        }

        for(let ob of obstacles){
            if(ob.collide(snake.head.x,snake.head.y,12)){snake.alive=false;audioDie.play();}
            ob.draw(dt);
        }

        let d=Math.hypot(snake.head.x-apple.x,snake.head.y-apple.y);
        if(d<12+apple.r && (elapsed>3 || Math.hypot(joystickDir.x,joystickDir.y)>0.01)){
            score+=10;
            snake.grow(3);
            audioEat.play();
            apple.respawn();
            if(level<250 && score >= level*10){
                level++;
                snake.head.speed+=5;
                obstacles.push(new Obstacle(Math.random()*(W-100), Math.random()*(H-100), 40, 20));
                audioLevel.play();
            }
            updateUI();
        }
        apple.draw();
        snake.draw();
    } else {
        ctx.fillStyle="rgba(0,0,0,0.5)";
        ctx.fillRect(0,0,W,H);
        ctx.fillStyle="red";
        ctx.font="60px sans-serif";
        ctx.textAlign="center";
        ctx.fillText("GAME OVER",W/2,H/2);
        ctx.font="30px sans-serif";
        ctx.fillText(`Score: ${score}`,W/2,H/2+50);
        gameRunning = false;
        document.getElementById("startButton").style.display="block";
    }
    requestAnimationFrame(loop);
}

const startButton = document.getElementById("startButton");
startButton.addEventListener("click", () => {
    startButton.style.display = "none";
    last = null;
    startTime = null;
    initGame();
    gameRunning = true;
    loop(performance.now());
});

window.addEventListener("resize",()=>{W=window.innerWidth;H=window.innerHeight;canvas.width=W;canvas.height=H;});

const infoButton = document.getElementById("infoButton");
const infoBox = document.getElementById("infoBox");
infoButton.addEventListener("click", () => {
    infoBox.style.display = (infoBox.style.display === "none") ? "block" : "none";
});
</script>
</body>
</html>
