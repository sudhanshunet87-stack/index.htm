<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Snip the Strings!</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background: linear-gradient(to bottom, #87CEEB, #E0F7FA);
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            user-select: none;
        }

        #gameCanvas {
            border: 4px solid #333;
            background-color: rgba(255, 255, 255, 0.3);
            box-shadow: 0 10px 20px rgba(0,0,0,0.2);
            cursor: crosshair;
        }

        #ui {
            position: absolute;
            top: 20px;
            left: 20px;
            pointer-events: none;
        }

        h1 {
            margin: 0;
            color: #333;
            text-shadow: 2px 2px 0px white;
        }

        p {
            margin: 5px 0;
            font-size: 18px;
            color: #444;
            background: rgba(255,255,255,0.7);
            padding: 5px 10px;
            border-radius: 5px;
        }

        #gameOverScreen {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: white;
            padding: 40px;
            border-radius: 15px;
            text-align: center;
            box-shadow: 0 10px 30px rgba(0,0,0,0.3);
            display: none;
        }

        button {
            background: #ff4757;
            color: white;
            border: none;
            padding: 15px 30px;
            font-size: 20px;
            border-radius: 8px;
            cursor: pointer;
            transition: transform 0.1s;
        }

        button:hover {
            transform: scale(1.05);
            background: #ff6b81;
        }
    </style>
</head>
<body>

    <div id="ui">
        <h1>Snip the Strings</h1>
        <p>Score: <span id="score">0</span></p>
        <p>Drag mouse to cut strings!</p>
    </div>

    <canvas id="gameCanvas"></canvas>

    <div id="gameOverScreen">
        <h2 style="color: #ff4757; font-size: 40px; margin: 0 0 20px 0;">POP!</h2>
        <p>You popped a balloon.</p>
        <p>Final Score: <span id="finalScore">0</span></p>
        <br>
        <button onclick="resetGame()">Try Again</button>
    </div>

<script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const scoreEl = document.getElementById('score');
    const finalScoreEl = document.getElementById('finalScore');
    const gameOverScreen = document.getElementById('gameOverScreen');

    // Game State
    let width, height;
    let balloons = [];
    let score = 0;
    let isGameOver = false;
    let mouse = { x: 0, y: 0, isDown: false, prevX: 0, prevY: 0 };
    let spawnTimer = 0;

    // Colors
    const balloonColors = ['#ff4757', '#2ed573', '#1e90ff', '#ffa502', '#a55eea'];

    // Resize handling
    function resize() {
        width = window.innerWidth * 0.8;
        height = window.innerHeight * 0.8;
        canvas.width = width;
        canvas.height = height;
    }
    window.addEventListener('resize', resize);
    resize();

    // Mouse Events
    canvas.addEventListener('mousedown', (e) => {
        const rect = canvas.getBoundingClientRect();
        mouse.isDown = true;
        mouse.x = e.clientX - rect.left;
        mouse.y = e.clientY - rect.top;
        mouse.prevX = mouse.x;
        mouse.prevY = mouse.y;
    });

    canvas.addEventListener('mousemove', (e) => {
        const rect = canvas.getBoundingClientRect();
        mouse.prevX = mouse.x;
        mouse.prevY = mouse.y;
        mouse.x = e.clientX - rect.left;
        mouse.y = e.clientY - rect.top;
    });

    canvas.addEventListener('mouseup', () => {
        mouse.isDown = false;
    });

    // Classes
    class Balloon {
        constructor() {
            this.radius = 20 + Math.random() * 15;
            this.x = Math.random() * (width - 100) + 50;
            this.y = height + this.radius;
            this.speed = 1 + Math.random() * 2;
            this.color = balloonColors[Math.floor(Math.random() * balloonColors.length)];
            this.stringLength = 100 + Math.random() * 100;
            this.isSnipped = false;
            this.floatOffset = Math.random() * Math.PI * 2;
        }

        update() {
            if (this.isSnipped) {
                this.y -= this.speed * 2; // Float up faster
                this.radius *= 0.98; // Shrink slightly
            } else {
                this.y -= this.speed;
            }
        }

        draw() {
            // Draw String
            if (!this.isSnipped) {
                ctx.beginPath();
                ctx.moveTo(this.x, this.y + this.radius);
                ctx.lineTo(this.x, this.y + this.radius + this.stringLength);
                ctx.strokeStyle = '#555';
                ctx.lineWidth = 2;
                ctx.stroke();
            }

            // Draw Balloon
            ctx.beginPath();
            ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
            ctx.fillStyle = this.color;
            ctx.fill();
            
            // Shine effect
            ctx.beginPath();
            ctx.arc(this.x - this.radius*0.3, this.y - this.radius*0.3, this.radius*0.2, 0, Math.PI * 2);
            ctx.fillStyle = 'rgba(255,255,255,0.4)';
            ctx.fill();

            // Knot
            ctx.beginPath();
            ctx.moveTo(this.x, this.y + this.radius);
            ctx.lineTo(this.x - 3, this.y + this.radius + 5);
            ctx.lineTo(this.x + 3, this.y + this.radius + 5);
            ctx.fillStyle = this.color;
            ctx.fill();
        }
    }

    // Game Logic
    function checkCollision() {
        if (!mouse.isDown) return;

        // Create a line segment from previous mouse pos to current
        const lineStart = { x: mouse.prevX, y: mouse.prevY };
        const lineEnd = { x: mouse.x, y: mouse.y };

        balloons.forEach(balloon => {
            if (balloon.isSnipped) return;

            // Calculate string endpoints
            const stringTop = { x: balloon.x, y: balloon.y + balloon.radius };
            const stringBottom = { x: balloon.x, y: balloon.y + balloon.radius + balloon.stringLength };

            // Check intersection between mouse line and string line
            if (lineIntersect(lineStart, lineEnd, stringTop, stringBottom)) {
                balloon.isSnipped = true;
                score++;
                scoreEl.innerText = score;
                createParticles(balloon.x, balloon.y + balloon.radius, balloon.color);
            }
        });
    }

    // Math helper for line intersection
    function lineIntersect(p1, p2, p3, p4) {
        const det = (p2.x - p1.x) * (p4.y - p3.y) - (p4.x - p3.x) * (p2.y - p1.y);
        if (det === 0) return false;
        const lambda = ((p4.y - p3.y) * (p4.x - p1.x) + (p3.x - p4.x) * (p4.y - p1.y)) / det;
        const gamma = ((p1.y - p2.y) * (p4.x - p1.x) + (p2.x - p1.x) * (p4.y - p1.y)) / det;
        return (0 < lambda && lambda < 1) && (0 < gamma && gamma < 1);
    }

    function createParticles(x, y, color) {
        // Simple particle effect could go here, keeping it simple for now
    }

    function drawScissors() {
        if (mouse.isDown) {
            ctx.beginPath();
            ctx.moveTo(mouse.prevX, mouse.prevY);
            ctx.lineTo(mouse.x, mouse.y);
            ctx.strokeStyle = '#333';
            ctx.lineWidth = 3;
            ctx.stroke();

            // Draw little circles at the ends of the scissors
            ctx.fillStyle = '#333';
            ctx.beginPath();
            ctx.arc(mouse.prevX, mouse.prevY, 4, 0, Math.PI*2);
            ctx.fill();
            ctx.beginPath();
            ctx.arc(mouse.x, mouse.y, 4, 0, Math.PI*2);
            ctx.fill();
        }
    }

    function update() {
        if (isGameOver) return;

        ctx.clearRect(0, 0, width, height);

        // Spawn Balloons
        spawnTimer++;
        if (spawnTimer > 60) { // Spawn every 60 frames (approx 1 sec)
            balloons.push(new Balloon());
            spawnTimer = 0;
        }

        // Update and Draw Balloons
        for (let i = balloons.length - 1; i >= 0; i--) {
            let b = balloons[i];
            b.update();
            b.draw();

            // Remove if off screen
            if (b.y < -50) {
                balloons.splice(i, 1);
            }
            // Game Over if balloon hits bottom (and wasn't snipped)
            else if (b.y > height && !b.isSnipped) {
                gameOver();
            }
        }

        checkCollision();
        drawScissors();

        requestAnimationFrame(update);
    }

    function gameOver() {
        isGameOver = true;
        finalScoreEl.innerText = score;
        gameOverScreen.style.display = 'block';
    }

    function resetGame() {
        balloons = [];
        score = 0;
        scoreEl.innerText = '0';
        isGameOver = false;
        gameOverScreen.style.display = 'none';
        update();
    }

    // Start
    update();

</script>
</body>
</html>
