<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Torombolitos 2D - Captura la Bandera</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; user-select: none; font-family: sans-serif; }
        body, html { width: 100%; height: 100%; overflow: hidden; background: #1a1c23; color: #fff; }
        #gameCanvas { display: block; background: #222831; }
        
        /* UI Overlay */
        .ui-panel {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            display: flex; flex-direction: column; justify-content: center; align-items: center;
            background: rgba(18, 20, 28, 0.9); z-index: 10;
        }
        .card {
            background: #2d3748; padding: 25px; border-radius: 15px; text-align: center;
            box-shadow: 0 10px 25px rgba(0,0,0,0.5); max-width: 400px; width: 90%;
        }
        h1 { color: #4DEEEA; margin-bottom: 10px; font-size: 1.8rem; }
        p { color: #cbd5e0; margin-bottom: 20px; font-size: 0.95rem; }
        .btn {
            background: #4DEEEA; color: #111; border: none; padding: 12px 24px;
            font-size: 1rem; font-weight: bold; border-radius: 25px; cursor: pointer;
            width: 100%; margin: 8px 0; transition: transform 0.1s;
        }
        .btn:active { transform: scale(0.98); }
        .btn-red { background: #ff4757; color: #fff; }

        /* HUD */
        #hud {
            position: absolute; top: 15px; left: 50%; transform: translateX(-50%);
            display: none; gap: 20px; background: rgba(0,0,0,0.6); padding: 8px 20px;
            border-radius: 20px; font-size: 1.2rem; font-weight: bold; pointer-events: none;
        }
        .score-red { color: #ff4757; }
        .score-blue { color: #2ed573; }
        .timer { color: #eccc68; }

        /* Controles móviles */
        #mobile-controls {
            position: absolute; bottom: 20px; width: 100%; display: none;
            justify-content: space-between; padding: 0 30px; pointer-events: none;
        }
        #dash-btn {
            width: 70px; height: 70px; border-radius: 50%; background: #eccc68;
            border: none; font-weight: bold; pointer-events: auto; font-size: 0.9rem;
        }
    </style>
</head>
<body>

    <canvas id="gameCanvas"></canvas>

    <!-- HUD del Juego -->
    <div id="hud">
        <span class="score-red" id="scoreR">0</span> - <span class="score-blue" id="scoreB">0</span>
        <span class="timer" id="timerText">06:00</span>
    </div>

    <!-- Menú Principal -->
    <div id="menu" class="ui-panel">
        <div class="card">
            <h1>TOROMBOLITOS</h1>
            <p>Captura la Bandera 2D</p>
            <button class="btn" onclick="startGame()">🎮 JUGAR PRACTICA (1v1 IA)</button>
        </div>
    </div>

    <!-- Controles táctiles -->
    <div id="mobile-controls">
        <button id="dash-btn" onclick="triggerDash()">DASH</button>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;

        let gameRunning = false;
        let timeLeft = 360;
        let scoreRed = 0, scoreBlue = 0;

        // Teclas
        const keys = {};
        window.addEventListener('keydown', e => keys[e.key] = true);
        window.addEventListener('keyup', e => keys[e.key] = false);

        // Personaje Jugador (Rojo)
        const player = {
            x: 100, y: canvas.height / 2, radius: 24,
            speed: 3.5, color: '#ff4757', team: 'red',
            vx: 0, vy: 0, isFrozen: false, freezeTimer: 0,
            hasFlag: false, dashCooldown: 0
        };

        // Bot IA (Azul)
        const bot = {
            x: canvas.width - 100, y: canvas.height / 2, radius: 24,
            speed: 2.8, color: '#2ed573', team: 'blue',
            isFrozen: false, hasFlag: false
        };

        // Banderas
        const flagRed = { x: 80, y: canvas.height / 2, homeX: 80, homeY: canvas.height / 2, carrier: null, color: '#ff4757' };
        const flagBlue = { x: canvas.width - 80, y: canvas.height / 2, homeX: canvas.width - 80, homeY: canvas.height / 2, carrier: null, color: '#2ed573' };

        // Obstáculos de Retraso
        const slowZones = [
            { x: canvas.width/2 - 50, y: 100, w: 100, h: 150 },
            { x: canvas.width/2 - 50, y: canvas.height - 250, w: 100, h: 150 }
        ];

        const walls = [
            { x: canvas.width/3, y: canvas.height/2 - 80, w: 20, h: 160 },
            { x: (canvas.width/3)*2, y: canvas.height/2 - 80, w: 20, h: 160 }
        ];

        function drawTorombolito(p) {
            ctx.save();
            ctx.translate(p.x, p.y);

            // Si está congelado
            if(p.isFrozen) {
                ctx.fillStyle = 'rgba(77, 238, 234, 0.5)';
                ctx.beginPath();
                ctx.arc(0, 0, p.radius + 6, 0, Math.PI * 2);
                ctx.fill();
            }

            // Cuerpo
            ctx.fillStyle = p.color;
            ctx.beginPath();
            ctx.arc(0, 0, p.radius, 0, Math.PI * 2);
            ctx.fill();

            // Orejitas
            ctx.fillStyle = p.color;
            ctx.beginPath();
            ctx.moveTo(-12, -18); ctx.lineTo(-18, -30); ctx.lineTo(-4, -22);
            ctx.moveTo(12, -18); ctx.lineTo(18, -30); ctx.lineTo(4, -22);
            ctx.fill();

            // Antenitas
            ctx.strokeStyle = p.color;
            ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.moveTo(-4, -22); ctx.quadraticCurveTo(-8, -32, -4, -35);
            ctx.moveTo(4, -22); ctx.quadraticCurveTo(8, -32, 4, -35);
            ctx.stroke();

            // Ojos Gigantes Blancos
            ctx.fillStyle = '#fff';
            ctx.beginPath();
            ctx.arc(-8, -2, 9, 0, Math.PI * 2);
            ctx.arc(8, -2, 9, 0, Math.PI * 2);
            ctx.fill();

            // Pupilas Oscuras
            ctx.fillStyle = '#111';
            ctx.beginPath();
            ctx.arc(-7, -2, 4, 0, Math.PI * 2);
            ctx.arc(9, -2, 4, 0, Math.PI * 2);
            ctx.fill();

            // Nariz
            ctx.fillStyle = '#111';
            ctx.beginPath();
            ctx.moveTo(-2, 7); ctx.lineTo(2, 7); ctx.lineTo(0, 10);
            ctx.fill();

            ctx.restore();
        }

        function drawFlags() {
            [flagRed, flagBlue].forEach(f => {
                ctx.fillStyle = f.color;
                ctx.fillRect(f.x - 5, f.y - 25, 20, 15);
                ctx.strokeStyle = '#fff';
                ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.moveTo(f.x - 5, f.y - 25);
                ctx.lineTo(f.x - 5, f.y + 15);
                ctx.stroke();
            });
        }

        function triggerDash() {
            if(player.dashCooldown <= 0 && !player.isFrozen) {
                player.speed = 8;
                player.dashCooldown = 100;
                setTimeout(() => player.speed = 3.5, 300);
            }
        }

        function update() {
            if(!gameRunning) return;

            // Enfriamiento de Dash
            if(player.dashCooldown > 0) player.dashCooldown--;

            // Movimiento del Jugador
            let currentSpeed = player.speed;

            // Reducción por Zona de Lodo / Ciénaga
            slowZones.forEach(z => {
                if(player.x > z.x && player.x < z.x + z.w && player.y > z.y && player.y < z.y + z.h) {
                    currentSpeed *= 0.4;
                }
            });

            if(!player.isFrozen) {
                if(keys['w'] || keys['ArrowUp']) player.y -= currentSpeed;
                if(keys['s'] || keys['ArrowDown']) player.y += currentSpeed;
                if(keys['a'] || keys['ArrowLeft']) player.x -= currentSpeed;
                if(keys['d'] || keys['ArrowRight']) player.x += currentSpeed;
            }

            // IA simple del Bot
            if(!bot.isFrozen) {
                let targetX = flagRed.x, targetY = flagRed.y;
                if(bot.hasFlag) { targetX = flagBlue.homeX; targetY = flagBlue.homeY; }
                
                let dx = targetX - bot.x;
                let dy = targetY - bot.y;
                let dist = Math.hypot(dx, dy);
                if(dist > 0) {
                    bot.x += (dx / dist) * bot.speed;
                    bot.y += (dy / dist) * bot.speed;
                }
            }

            // Lógica de Banderas
            if(Math.hypot(player.x - flagBlue.x, player.y - flagBlue.y) < 30 && !player.hasFlag) {
                player.hasFlag = true;
                flagBlue.carrier = player;
            }
            if(player.hasFlag) {
                flagBlue.x = player.x; flagBlue.y = player.y - 20;
                // Anotar punto
                if(Math.hypot(player.x - flagRed.homeX, player.y - flagRed.homeY) < 40) {
                    scoreRed++;
                    player.hasFlag = false;
                    flagBlue.x = flagBlue.homeX; flagBlue.y = flagBlue.homeY;
                    document.getElementById('scoreR').innerText = scoreRed;
                }
            }

            if(Math.hypot(bot.x - flagRed.x, bot.y - flagRed.y) < 30 && !bot.hasFlag) {
                bot.hasFlag = true;
                flagRed.carrier = bot;
            }
            if(bot.hasFlag) {
                flagRed.x = bot.x; flagRed.y = bot.y - 20;
                if(Math.hypot(bot.x - flagBlue.homeX, bot.y - flagBlue.homeY) < 40) {
                    scoreBlue++;
                    bot.hasFlag = false;
                    flagRed.x = flagRed.homeX; flagRed.y = flagRed.homeY;
                    document.getElementById('scoreB').innerText = scoreBlue;
                }
            }
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Zonas de Lodo (Obstáculos)
            ctx.fillStyle = 'rgba(61, 35, 20, 0.6)';
            slowZones.forEach(z => ctx.fillRect(z.x, z.y, z.w, z.h));

            // Muros
            ctx.fillStyle = '#4a5568';
            walls.forEach(w => ctx.fillRect(w.x, w.y, w.w, w.h));

            drawFlags();
            drawTorombolito(player);
            drawTorombolito(bot);

            update();
            requestAnimationFrame(draw);
        }

        function startGame() {
            document.getElementById('menu').style.display = 'none';
            document.getElementById('hud').style.display = 'flex';
            if('ontouchstart' in window) document.getElementById('mobile-controls').style.display = 'flex';
            gameRunning = true;
            
            setInterval(() => {
                if(timeLeft > 0 && gameRunning) {
                    timeLeft--;
                    let m = Math.floor(timeLeft/60).toString().padStart(2, '0');
                    let s = (timeLeft%60).toString().padStart(2, '0');
                    document.getElementById('timerText').innerText = `${m}:${s}`;
                }
            }, 1000);

            draw();
        }
    </script>
</body>
</html>
