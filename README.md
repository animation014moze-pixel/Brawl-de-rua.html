<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>FutBrawl Rua - V3.0</title>
    <style>
        :root { --blue: #3498db; --red: #e74c3c; --yellow: #f1c40f; --green: #2ecc71; --wall: #d35400; }
        body { margin: 0; overflow: hidden; background: #2c3e50; font-family: 'Arial Black', sans-serif; touch-action: none; user-select: none; -webkit-user-select: none; }
        
        /* TELAS */
        .screen { position: absolute; inset: 0; display: none; flex-direction: column; align-items: center; justify-content: center; z-index: 1000; background: rgba(0,0,0,0.9); }
        .active { display: flex; }

        /* LOBBY */
        #screen-lobby { background: linear-gradient(135deg, #2980b9, #2c3e50); }
        .title { color: var(--yellow); font-size: 40px; text-shadow: 4px 4px 0 #000; margin-bottom: 20px; text-align: center; }
        .btn-play { background: var(--yellow); color: black; padding: 20px 60px; border-radius: 15px; font-size: 24px; border: 4px solid #f39c12; cursor: pointer; box-shadow: 0 10px 0 #d35400; transition: transform 0.1s; }
        .btn-play:active { transform: translateY(10px); box-shadow: none; }

        /* HUD JOGO */
        #game-ui { pointer-events: none; z-index: 500; }
        .top-bar { position: absolute; top: 10px; width: 100%; display: flex; justify-content: center; gap: 20px; font-size: 24px; color: white; text-shadow: 2px 2px 0 #000; }
        
        /* JOYSTICKS */
        .joystick-container { position: absolute; bottom: 40px; width: 100%; display: flex; justify-content: space-between; padding: 0 40px; box-sizing: border-box; }
        .joy-base { width: 100px; height: 100px; background: rgba(0,0,0,0.3); border: 2px solid rgba(255,255,255,0.3); border-radius: 50%; position: relative; pointer-events: auto; }
        .joy-stick { width: 40px; height: 40px; background: white; border-radius: 50%; position: absolute; top: 30px; left: 30px; pointer-events: none; box-shadow: 0 5px 15px rgba(0,0,0,0.5); }
        
        /* BOTÃO SUPER */
        #btn-super { 
            position: absolute; bottom: 160px; right: 40px; 
            width: 70px; height: 70px; 
            background: radial-gradient(circle, #f1c40f, #e67e22); 
            border: 3px solid white; border-radius: 50%; 
            display: none; align-items: center; justify-content: center; 
            font-size: 30px; color: black; pointer-events: auto; 
            animation: pulse 0.5s infinite alternate;
            box-shadow: 0 0 20px var(--yellow);
        }
        @keyframes pulse { from { transform: scale(1); } to { transform: scale(1.1); } }
        .super-active { border-color: red !important; background: radial-gradient(circle, #fff, #f1c40f) !important; }

        canvas { display: block; position: absolute; top: 0; left: 0; z-index: 10; }
    </style>
</head>
<body>

<div id="screen-lobby" class="screen active">
    <div class="title">FUTBRAWL RUA</div>
    <div style="color:white; margin-bottom:30px; text-align:center; font-size:14px;">OBJETIVO: LEVE A BOLA AO GOL ADVERSÁRIO<br>USE O SUPER PARA QUEBRAR PAREDES!</div>
    <button class="btn-play" id="startBtn">BORA JOGAR</button>
</div>

<div id="game-ui" class="screen">
    <div class="top-bar">
        <span style="color:var(--blue)">AZUL: <span id="scoreA">0</span></span>
        <span style="color:white">|</span>
        <span style="color:var(--red)">VERMELHO: <span id="scoreB">0</span></span>
    </div>
    <div id="btn-super" onclick="toggleSuper()">☠️</div>
    <div class="joystick-container">
        <div class="joy-base" id="joyMoveBase"><div class="joy-stick" id="joyMoveStick"></div></div>
        <div class="joy-base" id="joyShootBase"><div class="joy-stick" id="joyShootStick"></div></div>
    </div>
</div>

<canvas id="gameCanvas"></canvas>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
// Ajuste para celular: ocupa tudo
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;

// --- CONFIGURAÇÃO DO MAPA (0=Chão, 1=Caixa(Quebra), 2=Parede(Fixo), 3=Moita, 9=Gol) ---
// Baseado na imagem: Gol em cima e embaixo, obstáculos no meio
const TILE_SIZE = 40;
const MAP_COLS = 13;
const MAP_ROWS = 20; // Campo vertical
// Mapa simplificado da imagem
const LEVEL_DESIGN = [
    [2,2,2,2,2,9,9,9,2,2,2,2,2], // Gol Vermelho
    [2,1,1,0,0,0,0,0,0,0,1,1,2],
    [2,1,0,0,0,2,2,2,0,0,0,1,2],
    [2,0,0,3,3,0,0,0,3,3,0,0,2],
    [2,0,1,1,0,0,0,0,0,1,1,0,2],
    [2,0,1,0,0,0,0,0,0,0,1,0,2],
    [2,0,0,0,2,0,0,0,2,0,0,0,2], // Meio campo
    [2,0,0,0,2,0,0,0,2,0,0,0,2],
    [2,0,0,0,0,0,0,0,0,0,0,0,2],
    [2,0,0,3,3,0,5,0,3,3,0,0,2], // 5 = Bola Spawn (Centro aproximado)
    [2,0,0,0,0,0,0,0,0,0,0,0,2],
    [2,0,0,0,2,0,0,0,2,0,0,0,2],
    [2,0,1,0,0,0,0,0,0,0,1,0,2],
    [2,0,1,1,0,0,0,0,0,1,1,0,2],
    [2,0,0,3,3,0,0,0,3,3,0,0,2],
    [2,1,0,0,0,2,2,2,0,0,0,1,2],
    [2,1,1,0,0,0,0,0,0,0,1,1,2],
    [2,2,2,2,2,9,9,9,2,2,2,2,2]  // Gol Azul
];

let map = []; // Será preenchido no init
let camera = {x:0, y:0};
let gameActive = false;

// --- ENTIDADES ---
let player = { x:0, y:0, r:18, speed:5, color:'#3498db', hp:100, maxHp:100, ammo:3, maxAmmo:3, reload:0, super:0, hasBall:false };
let ball = { x:0, y:0, vx:0, vy:0, owner: null, r: 12 };
let enemies = [];
let bullets = [];
let particles = [];
let scores = { a: 0, b: 0 };
let superReady = false; // Se o botão foi clicado

// --- JOYSTICKS ---
const joyMove = { active:false, x:0, y:0, id:null };
const joyShoot = { active:false, x:0, y:0, id:null };

// --- INICIALIZAÇÃO ---
document.getElementById('startBtn').addEventListener('click', startGame);

function startGame() {
    document.getElementById('screen-lobby').classList.remove('active');
    document.getElementById('game-ui').classList.add('active');
    gameActive = true;
    resetRound();
    requestAnimationFrame(gameLoop);
}

function resetRound() {
    // Clonar o mapa original para o jogo (para poder quebrar paredes)
    map = JSON.parse(JSON.stringify(LEVEL_DESIGN));
    
    // Achar posições iniciais
    let centerX = (MAP_COLS * TILE_SIZE) / 2;
    let centerY = (MAP_ROWS * TILE_SIZE) / 2;
    
    player.x = centerX; 
    player.y = (MAP_ROWS - 3) * TILE_SIZE; // Perto do gol azul
    player.hasBall = false;
    player.hp = 100;
    
    ball.x = centerX; 
    ball.y = centerY;
    ball.vx = 0; ball.vy = 0; ball.owner = null;

    enemies = [];
    // Spawnar 3 inimigos (Time Vermelho)
    for(let i=0; i<3; i++) {
        enemies.push({
            x: centerX + (i-1)*60,
            y: 3 * TILE_SIZE,
            hp: 100, r: 18, color: '#e74c3c'
        });
    }
}

// --- CONTROLES E SUPER ---
function toggleSuper() {
    if(player.super >= 100) {
        superReady = !superReady;
        document.getElementById('btn-super').classList.toggle('super-active');
    }
}

// Lógica de Touch (A mesma da V2.2 mas ajustada)
function handleTouch(e) {
    if(!gameActive) return;
    e.preventDefault();
    const touches = e.changedTouches;
    for(let i=0; i<touches.length; i++) {
        const t = touches[i];
        const isRight = t.clientX > window.innerWidth/2;
        const joy = isRight ? joyShoot : joyMove;
        const base = document.getElementById(isRight ? 'joyShootBase' : 'joyMoveBase');
        const stick = document.getElementById(isRight ? 'joyShootStick' : 'joyMoveStick');
        
        if(e.type === 'touchstart' && !joy.active) {
            joy.active = true; joy.id = t.identifier;
        }
        if(joy.active && joy.id === t.identifier) {
            if(e.type === 'touchmove') {
                const rect = base.getBoundingClientRect();
                let dx = t.clientX - (rect.left + rect.width/2);
                let dy = t.clientY - (rect.top + rect.height/2);
                let dist = Math.sqrt(dx*dx+dy*dy);
                let max = 40;
                if(dist>max) { dx *= max/dist; dy *= max/dist; }
                stick.style.transform = `translate(${dx}px, ${dy}px)`;
                joy.x = dx/max; joy.y = dy/max;
                
                // Atirar ou Chutar
                if(isRight && dist > 20) {
                    if(player.hasBall) kickBall();
                    else fireBullet();
                }
            }
            if(e.type === 'touchend') {
                joy.active = false; joy.id = null; joy.x =0; joy.y=0;
                stick.style.transform = `translate(0,0)`;
                // Se soltou o joystick de tiro, reseta o estado do super se não atirou (opcional, mas bom manter ativo)
            }
        }
    }
}
document.addEventListener('touchstart', handleTouch, {passive:false});
document.addEventListener('touchmove', handleTouch, {passive:false});
document.addEventListener('touchend', handleTouch, {passive:false});

function fireBullet() {
    if(Date.now() - (player.lastShot||0) < 300 || player.ammo <= 0) return;
    
    player.ammo--;
    player.lastShot = Date.now();
    
    let isSuperShot = superReady;
    
    bullets.push({
        x: player.x, y: player.y,
        vx: joyShoot.x * (isSuperShot?12:10), vy: joyShoot.y * (isSuperShot?12:10),
        r: isSuperShot ? 15 : 6,
        isSuper: isSuperShot,
        life: 50,
        team: 'blue' // Player é sempre blue aqui
    });

    if(isSuperShot) {
        player.super = 0;
        superReady = false;
        document.getElementById('btn-super').style.display = 'none';
        document.getElementById('btn-super').classList.remove('super-active');
    }
}

function kickBall() {
    if(Date.now() - (player.lastKick||0) < 500) return;
    player.lastKick = Date.now();
    player.hasBall = false;
    ball.owner = null;
    ball.vx = joyShoot.x * 12; // Chute forte
    ball.vy = joyShoot.y * 12;
    // Consome uma munição pra chutar (estilo Brawl)
    if(player.ammo > 0) player.ammo--; 
}

// --- FÍSICA E LÓGICA ---
function gameLoop() {
    if(!gameActive) return;
    update();
    draw();
    requestAnimationFrame(gameLoop);
}

function update() {
    // 1. Player Movimento
    let nx = player.x + joyMove.x * player.speed;
    let ny = player.y + joyMove.y * player.speed;
    if(!checkWallCollision(nx, player.y)) player.x = nx;
    if(!checkWallCollision(player.x, ny)) player.y = ny;

    // Recarga
    if(player.ammo < player.maxAmmo) {
        player.reload++;
        if(player.reload > 50) { player.ammo++; player.reload = 0; }
    }

    // 2. Bola Física
    if(!ball.owner) {
        ball.x += ball.vx; ball.y += ball.vy;
        ball.vx *= 0.94; ball.vy *= 0.94; // Atrito
        
        // Colisão Bola-Parede
        if(checkWallCollision(ball.x+ball.vx, ball.y)) ball.vx *= -0.8;
        if(checkWallCollision(ball.x, ball.y+ball.vy)) ball.vy *= -0.8;

        // Pegar Bola (Player)
        let dx = player.x - ball.x, dy = player.y - ball.y;
        if(Math.sqrt(dx*dx+dy*dy) < player.r + ball.r) {
            player.hasBall = true; ball.owner = player;
        }
    } else {
        ball.x = ball.owner.x + (joyMove.x*10); // Carrega a bola um pouco a frente
        ball.y = ball.owner.y + (joyMove.y*10);
        ball.vx = 0; ball.vy = 0;
    }

    // 3. GOLS
    let goalTopY = TILE_SIZE * 1.5;
    let goalBotY = TILE_SIZE * (MAP_ROWS - 1.5);
    
    if(ball.y < goalTopY && ball.x > 5*TILE_SIZE && ball.x < 8*TILE_SIZE) {
        scores.a++; document.getElementById('scoreA').innerText = scores.a;
        resetRound();
        return;
    }
    if(ball.y > goalBotY && ball.x > 5*TILE_SIZE && ball.x < 8*TILE_SIZE) {
        scores.b++; document.getElementById('scoreB').innerText = scores.b;
        resetRound();
        return;
    }

    // 4. Balas e Super
    for(let i=bullets.length-1; i>=0; i--) {
        let b = bullets[i];
        b.x += b.vx; b.y += b.vy; b.life--;
        
        // Colisão com Paredes
        let c = Math.floor(b.x / TILE_SIZE);
        let r = Math.floor(b.y / TILE_SIZE);
        
        // Verifica limites do array antes de acessar map[r][c]
        if(r>=0 && r<MAP_ROWS && c>=0 && c<MAP_COLS) {
            let tile = map[r][c];
            if(tile === 1 || tile === 2) { // Bateu parede
                if(b.isSuper && tile === 1) {
                    // SUPER QUEBRA CAIXA!
                    map[r][c] = 0; // Vira chão
                    createParticles(b.x, b.y, '#d35400');
                }
                bullets.splice(i, 1);
                createParticles(b.x, b.y, 'white');
                continue;
            }
        }
        
        // Colisão com Inimigos
        enemies.forEach(en => {
            let dist = Math.hypot(b.x - en.x, b.y - en.y);
            if(dist < en.r + b.r) {
                en.hp -= b.isSuper ? 100 : 35; // Super dá hit kill ou muito dano
                bullets.splice(i, 1);
                
                // Carrega Super
                if(!b.isSuper && player.super < 100) {
                    player.super += 20; // 5 tiros carrega
                    if(player.super >= 100) {
                        player.super = 100;
                        document.getElementById('btn-super').style.display = 'flex';
                    }
                }
            }
        });
        
        if(b.life <= 0) bullets.splice(i, 1);
    }

    // 5. Atualizar Inimigos (IA burra que segue a bola)
    enemies.forEach((en, i) => {
        if(en.hp <= 0) { enemies.splice(i,1); return; }
        let target = ball.owner ? ball.owner : ball;
        let angle = Math.atan2(target.y - en.y, target.x - en.x);
        let ex = en.x + Math.cos(angle) * 2;
        let ey = en.y + Math.sin(angle) * 2;
        if(!checkWallCollision(ex, ey)) { en.x = ex; en.y = ey; }
    });

    // Câmera
    camera.x = player.x - canvas.width/2;
    camera.y = player.y - canvas.height/2;
    
    // Partículas
    particles.forEach((p,i) => { p.life--; if(p.life<=0) particles.splice(i,1); });
}

function checkWallCollision(x, y) {
    let c = Math.floor(x / TILE_SIZE);
    let r = Math.floor(y / TILE_SIZE);
    if(r < 0 || r >= MAP_ROWS || c < 0 || c >= MAP_COLS) return true;
    let t = map[r][c];
    return (t === 1 || t === 2); // 1 e 2 bloqueiam, 3 (moita) não
}

function createParticles(x, y, color) {
    for(let i=0; i<5; i++) {
        particles.push({
            x:x, y:y, 
            vx:(Math.random()-0.5)*5, vy:(Math.random()-0.5)*5,
            life: 20, color: color
        });
    }
}

// --- DESENHO ---
function draw() {
    ctx.fillStyle = '#2c3e50'; // Fundo fora do mapa
    ctx.fillRect(0,0, canvas.width, canvas.height);
    
    ctx.save();
    ctx.translate(-camera.x, -camera.y);

    // Desenhar Mapa
    for(let r=0; r<MAP_ROWS; r++) {
        for(let c=0; c<MAP_COLS; c++) {
            let tile = map[r][c];
            let x = c*TILE_SIZE, y = r*TILE_SIZE;
            
            if(tile === 0) { ctx.fillStyle = '#27ae60'; ctx.fillRect(x,y,TILE_SIZE,TILE_SIZE); } // Grama
            if(tile === 1) { ctx.fillStyle = '#d35400'; ctx.fillRect(x+2,y+2,TILE_SIZE-4,TILE_SIZE-4); ctx.strokeStyle='#e67e22'; ctx.strokeRect(x+5,y+5,TILE_SIZE-10,TILE_SIZE-10); } // Caixa (X)
            if(tile === 2) { ctx.fillStyle = '#7f8c8d'; ctx.fillRect(x,y,TILE_SIZE,TILE_SIZE); } // Parede Indestrutível
            if(tile === 3) { ctx.fillStyle = '#2ecc71'; ctx.fillRect(x,y,TILE_SIZE,TILE_SIZE); } // Moita (Renderiza chão primeiro, moita por cima dps)
            if(tile === 9) { ctx.fillStyle = 'rgba(0,0,0,0.3)'; ctx.fillRect(x,y,TILE_SIZE,TILE_SIZE); } // Gol
        }
    }

    // Linhas do Gol
    ctx.fillStyle = 'white';
    ctx.fillRect(5*TILE_SIZE, 1.5*TILE_SIZE, 3*TILE_SIZE, 2); // Linha Gol Cima
    ctx.fillRect(5*TILE_SIZE, (MAP_ROWS-1.5)*TILE_SIZE, 3*TILE_SIZE, 2); // Linha Gol Baixo

    // Bola
    ctx.fillStyle = 'white'; 
    ctx.beginPath(); ctx.arc(ball.x, ball.y, ball.r, 0, Math.PI*2); ctx.fill();
    ctx.strokeStyle = 'black'; ctx.stroke();

    // Player
    // Se estiver na moita (tile 3), fica meio transparente
    let pc = Math.floor(player.x/TILE_SIZE), pr = Math.floor(player.y/TILE_SIZE);
    let inBush = (pr>=0 && pr<MAP_ROWS && map[pr][pc]===3);
    
    ctx.globalAlpha = inBush ? 0.6 : 1.0;
    ctx.fillStyle = player.color;
    ctx.beginPath(); ctx.arc(player.x, player.y, player.r, 0, Math.PI*2); ctx.fill();
    
    // Anel do Super no Player
    if(player.super >= 100) {
        ctx.strokeStyle = '#f1c40f'; ctx.lineWidth = 3;
        ctx.beginPath(); ctx.arc(player.x, player.y, player.r+5, 0, Math.PI*2); ctx.stroke();
    }
    
    // Nome e Munição
    ctx.globalAlpha = 1.0;
    ctx.fillStyle = 'white'; ctx.textAlign = 'center'; ctx.font = 'bold 12px Arial';
    ctx.fillText("VOCÊ", player.x, player.y - 30);
    ctx.fillStyle = '#f39c12'; ctx.fillRect(player.x-15, player.y-25, player.ammo*10, 4);

    // Inimigos
    enemies.forEach(en => {
        let ec = Math.floor(en.x/TILE_SIZE), er = Math.floor(en.y/TILE_SIZE);
        if(map[er] && map[er][ec] === 3 && !inBush) return; // Se inimigo na moita e vc fora, não desenha (invisível)
        
        ctx.fillStyle = en.color;
        ctx.beginPath(); ctx.arc(en.x, en.y, en.r, 0, Math.PI*2); ctx.fill();
        ctx.fillStyle = 'red'; ctx.fillRect(en.x-15, en.y-25, (en.hp/100)*30, 4);
    });

    // Balas
    bullets.forEach(b => {
        ctx.fillStyle = b.isSuper ? '#f1c40f' : '#f1c40f';
        ctx.beginPath(); ctx.arc(b.x, b.y, b.r, 0, Math.PI*2); ctx.fill();
        if(b.isSuper) { ctx.strokeStyle='red'; ctx.lineWidth=2; ctx.stroke(); }
    });

    // Partículas
    particles.forEach(p => {
        ctx.fillStyle = p.color; ctx.fillRect(p.x, p.y, 4, 4);
    });

    // Moitas (Camada superior para cobrir pés)
    for(let r=0; r<MAP_ROWS; r++) {
        for(let c=0; c<MAP_COLS; c++) {
            if(map[r][c] === 3) {
                ctx.fillStyle = 'rgba(46, 204, 113, 0.5)'; // Moita semitransparente por cima
                ctx.beginPath(); ctx.arc(c*TILE_SIZE+20, r*TILE_SIZE+20, 20, 0, Math.PI*2); ctx.fill();
            }
        }
    }

    ctx.restore();
}

window.addEventListener('resize', () => { canvas.width = window.innerWidth; canvas.height = window.innerHeight; });
</script>
</body>
</html>
