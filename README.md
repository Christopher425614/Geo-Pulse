<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Geo-Pulse: Precision</title>
    <style>
        body { margin: 0; background: #0a0a0c; color: #fff; font-family: 'Courier New', Courier, monospace; overflow: hidden; touch-action: none; }
        canvas { display: block; }
        #ui { position: absolute; top: 20px; width: 100%; text-align: center; pointer-events: none; font-size: 24px; text-shadow: 0 0 10px #00d4ff; }
        #overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; justify-content: center; align-items: center; background: rgba(0,0,0,0.8); display: none; }
        button { padding: 10px 20px; font-size: 18px; background: none; border: 2px solid #00d4ff; color: #00d4ff; cursor: pointer; margin-top: 20px; }
        button:hover { background: #00d4ff; color: #000; }
    </style>
</head>
<body>
    <div id="ui">SCORE: <span id="score">0</span></div>
    <div id="overlay">
        <h1>CRITICAL FAILURE</h1>
        <p id="finalScore"></p>
        <button onclick="resetGame()">REBOOT SYSTEM</button>
    </div>
    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreEl = document.getElementById('score');
        const overlay = document.getElementById('overlay');
        const finalScoreEl = document.getElementById('finalScore');

        let score, gameActive, player, enemies, bullets, particles, mouse;

        function init() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
            score = 0;
            gameActive = true;
            enemies = [];
            bullets = [];
            particles = [];
            mouse = { x: canvas.width/2, y: canvas.height/2 };
            player = { x: canvas.width/2, y: canvas.height/2, radius: 15, angle: 0 };
            overlay.style.display = 'none';
        }

        // Input Tracking
        const updateMouse = (e) => {
            const clientX = e.touches ? e.touches[0].clientX : e.clientX;
            const clientY = e.touches ? e.touches[0].clientY : e.clientY;
            mouse.x = clientX;
            mouse.y = clientY;
        };
        window.addEventListener('mousemove', updateMouse);
        window.addEventListener('touchmove', updateMouse);
        window.addEventListener('mousedown', () => { if(!gameActive) return; });

        function spawnEnemy() {
            if (!gameActive) return;
            const size = 15 + Math.random() * 15;
            let x, y;
            if (Math.random() < 0.5) {
                x = Math.random() < 0.5 ? -size : canvas.width + size;
                y = Math.random() * canvas.height;
            } else {
                x = Math.random() * canvas.width;
                y = Math.random() < 0.5 ? -size : canvas.height + size;
            }
            enemies.push({ x, y, size, speed: 1 + (score/200), color: '#ff3300' });
            setTimeout(spawnEnemy, Math.max(300, 1200 - (score * 2)));
        }

        function createExplosion(x, y, color, count = 10) {
            for (let i = 0; i < count; i++) {
                particles.push({
                    x, y, 
                    vx: (Math.random() - 0.5) * 8,
                    vy: (Math.random() - 0.5) * 8,
                    alpha: 1,
                    color
                });
            }
        }

        function resetGame() {
            init();
            spawnEnemy();
            animate();
        }

        function animate() {
            if (!gameActive) return;
            ctx.fillStyle = 'rgba(10, 10, 12, 0.15)';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Move Player toward Mouse (Lagged for smoothness)
            player.x += (mouse.x - player.x) * 0.1;
            player.y += (mouse.y - player.y) * 0.1;
            player.angle = Math.atan2(mouse.y - player.y, mouse.x - player.x);

            // Auto-shoot Logic
            if (Date.now() % 15 < 2) { // Rapid fire interval
                bullets.push({
                    x: player.x, y: player.y,
                    vx: Math.cos(player.angle) * 10,
                    vy: Math.sin(player.angle) * 10
                });
            }

            // Particles
            particles.forEach((p, i) => {
                p.x += p.vx; p.y += p.vy;
                p.alpha -= 0.02;
                if (p.alpha <= 0) particles.splice(i, 1);
                ctx.globalAlpha = p.alpha;
                ctx.fillStyle = p.color;
                ctx.fillRect(p.x, p.y, 2, 2);
            });
            ctx.globalAlpha = 1;

            // Bullets
            bullets.forEach((b, i) => {
                b.x += b.vx; b.y += b.vy;
                if (b.x < 0 || b.x > canvas.width || b.y < 0 || b.y > canvas.height) bullets.splice(i, 1);
                ctx.fillStyle = '#00d4ff';
                ctx.beginPath();
                ctx.arc(b.x, b.y, 3, 0, Math.PI * 2);
                ctx.fill();
            });

            // Enemies
            enemies.forEach((en, i) => {
                const dx = player.x - en.x;
                const dy = player.y - en.y;
                const dist = Math.hypot(dx, dy);
                en.x += (dx / dist) * en.speed;
                en.y += (dy / dist) * en.speed;

                // Player Collision
                if (dist < player.radius + en.size/2) {
                    gameActive = false;
                    createExplosion(player.x, player.y, '#fff', 30);
                    finalScoreEl.innerText = "FINAL SCORE: " + score;
                    overlay.style.display = 'flex';
                }

                // Bullet Collision
                bullets.forEach((b, bi) => {
                    if (Math.hypot(b.x - en.x, b.y - en.y) < en.size) {
                        createExplosion(en.x, en.y, en.color);
                        enemies.splice(i, 1);
                        bullets.splice(bi, 1);
                        score += 10;
                        scoreEl.innerText = score;
                    }
                });

                ctx.strokeStyle = en.color;
                ctx.lineWidth = 2;
                ctx.strokeRect(en.x - en.size/2, en.y - en.size/2, en.size, en.size);
            });

            // Draw Player
            ctx.save();
            ctx.translate(player.x, player.y);
            ctx.rotate(player.angle);
            ctx.strokeStyle = '#00d4ff';
            ctx.shadowBlur = 10; ctx.shadowColor = '#00d4ff';
            ctx.beginPath();
            ctx.moveTo(15, 0);
            ctx.lineTo(-10, -10);
            ctx.lineTo(-10, 10);
            ctx.closePath();
            ctx.stroke();
            ctx.restore();

            requestAnimationFrame(animate);
        }

        window.addEventListener('resize', () => {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        });

        resetGame();
    </script>
</body>
</html>
