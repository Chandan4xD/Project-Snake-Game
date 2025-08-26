<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Snake Game</title>
    <style>
        :root {
            --bg: #0f172a;
            --board: #111827;
            --snake: #22c55e;
            --snake-head: #16a34a;
            --food: #ef4444;
            --grid: #1f2937;
            --text: #e5e7eb;
        }

        * {
            box-sizing: border-box;
        }

        html, body {
            height: 100%;
            margin: 0;
            background: var(--bg);
            color: var(--text);
            font-family: 'Segoe UI', 'Roboto', 'Helvetica', 'Arial', sans-serif;
        }

        .wrap {
            min-height: 100%;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 24px;
        }

        .container {
            width: 100%;
            max-width: 520px;
            display: grid;
            gap: 12px;
        }

        .header {
            display: flex;
            align-items: baseline;
            justify-content: space-between;
        }

        .title {
            font-weight: 700;
            font-size: 20px;
        }

        .subtitle {
            opacity: 0.8;
            font-size: 13px;
        }

        .hud {
            display: flex;
            gap: 10px;
            align-items: center;
            justify-content: space-between;
        }

        .badge, .btn {
            background: #0b1020;
            border: 1px solid #1f2937;
            color: var(--text);
            padding: 8px 12px;
            border-radius: 10px;
            font-size: 13px;
        }

        .btn {
            cursor: pointer;
            font-weight: 600;
        }

        .btn:hover {
            background: #1e293b;
        }

        .board {
            border-radius: 16px;
            overflow: hidden;
            background: linear-gradient(180deg, #0b1020, var(--board));
            aspect-ratio: 1;
            box-shadow: inset 0 0 1px rgba(255,255,255,0.05);
            position: relative;
        }

        canvas {
            width: 100%;
            height: 100%;
            display: block;
        }

        .kbd {
            display: inline-block;
            padding: 2px 6px;
            border-radius: 6px;
            border: 1px solid #334155;
            background: #0b1020;
            font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
            font-size: 12px;
            margin: 0 2px;
        }

        .controls {
            text-align: center;
            opacity: 0.7;
            font-size: 14px;
            margin-top: 12px;
        }

        .game-over {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.9);
            color: white;
            padding: 20px;
            border-radius: 10px;
            text-align: center;
            display: none;
        }

        .game-over h2 {
            margin: 0 0 10px 0;
            color: #ef4444;
        }

        .game-over button {
            background: #22c55e;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 5px;
            cursor: pointer;
            font-size: 14px;
            margin-top: 10px;
        }

        .game-over button:hover {
            background: #16a34a;
        }
    </style>
</head>
<body>
    <div class="wrap">
        <div class="container">
            <div class="header">
                <div>
                    <div class="title">Snake Game</div>
                    <div class="subtitle">Walls wrap • Dies on self-collision • Space to pause/restart</div>
                </div>
            </div>

            <div class="hud">
                <div class="badge">Score: <strong id="scoreEl">0</strong></div>
                <div class="badge">High: <strong id="highEl">0</strong></div>
                <div class="badge">Speed: <strong id="speedEl">1</strong></div>
                <button class="btn" onclick="resetGame()">Reset</button>
            </div>

            <div class="board">
                <canvas id="game" width="400" height="400"></canvas>
                <div class="game-over" id="gameOver">
                    <h2>Game Over!</h2>
                    <p>Final Score: <span id="finalScore">0</span></p>
                    <button onclick="resetGame()">Play Again</button>
                </div>
            </div>

            <div class="controls">
                Use <span class="kbd">↑</span><span class="kbd">↓</span><span class="kbd">←</span><span class="kbd">→</span> or <span class="kbd">W</span><span class="kbd">A</span><span class="kbd">S</span><span class="kbd">D</span> to move • <span class="kbd">Space</span> to pause
            </div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('game');
        const ctx = canvas.getContext('2d');
        const scoreEl = document.getElementById('scoreEl');
        const highEl = document.getElementById('highEl');
        const speedEl = document.getElementById('speedEl');
        const gameOverEl = document.getElementById('gameOver');
        const finalScoreEl = document.getElementById('finalScore');

        // Game settings
        const GRID_SIZE = 20;
        const TILE_COUNT = canvas.width / GRID_SIZE;

        let snake = [{ x: 10, y: 10 }];
        let dx = 0;
        let dy = 0;
        let food = { x: 15, y: 15 };
        let score = 0;
        let highScore = localStorage.getItem('snakeHighScore') || 0;
        let gameRunning = false;
        let gamePaused = false;
        let speed = 150;
        let speedLevel = 1;

        highEl.textContent = highScore;

        function drawGame() {
            clearCanvas();
            moveSnake();
            drawFood();
            drawSnake();
            
            if (checkGameOver()) {
                gameOver();
                return;
            }

            if (checkFoodCollision()) {
                eatFood();
            }

            updateSpeed();
        }

        function clearCanvas() {
            ctx.fillStyle = '#111827';
            ctx.fillRect(0, 0, canvas.width, canvas.height);
        }

        function drawSnake() {
            snake.forEach((segment, index) => {
                if (index === 0) {
                    // Head
                    ctx.fillStyle = '#16a34a';
                } else {
                    // Body
                    ctx.fillStyle = '#22c55e';
                }
                ctx.fillRect(segment.x * GRID_SIZE, segment.y * GRID_SIZE, GRID_SIZE - 2, GRID_SIZE - 2);
            });
        }

        function drawFood() {
            ctx.fillStyle = '#ef4444';
            ctx.fillRect(food.x * GRID_SIZE, food.y * GRID_SIZE, GRID_SIZE - 2, GRID_SIZE - 2);
        }

        function moveSnake() {
            if (dx === 0 && dy === 0) return;

            const head = { x: snake[0].x + dx, y: snake[0].y + dy };

            // Wall wrapping
            if (head.x < 0) head.x = TILE_COUNT - 1;
            if (head.x >= TILE_COUNT) head.x = 0;
            if (head.y < 0) head.y = TILE_COUNT - 1;
            if (head.y >= TILE_COUNT) head.y = 0;

            snake.unshift(head);

            // Remove tail unless food was eaten
            if (!checkFoodCollision()) {
                snake.pop();
            }
        }

        function checkFoodCollision() {
            return snake[0].x === food.x && snake[0].y === food.y;
        }

        function checkGameOver() {
            // Check self collision (skip head)
            for (let i = 1; i < snake.length; i++) {
                if (snake[0].x === snake[i].x && snake[0].y === snake[i].y) {
                    return true;
                }
            }
            return false;
        }

        function eatFood() {
            score += 10;
            scoreEl.textContent = score;

            if (score > highScore) {
                highScore = score;
                highEl.textContent = highScore;
                localStorage.setItem('snakeHighScore', highScore);
            }

            generateFood();
        }

        function generateFood() {
            food = {
                x: Math.floor(Math.random() * TILE_COUNT),
                y: Math.floor(Math.random() * TILE_COUNT)
            };

            // Make sure food doesn't spawn on snake
            while (snake.some(segment => segment.x === food.x && segment.y === food.y)) {
                food = {
                    x: Math.floor(Math.random() * TILE_COUNT),
                    y: Math.floor(Math.random() * TILE_COUNT)
                };
            }
        }

        function updateSpeed() {
            const newSpeedLevel = Math.floor(score / 50) + 1;
            if (newSpeedLevel !== speedLevel) {
                speedLevel = newSpeedLevel;
                speedEl.textContent = speedLevel;
                speed = Math.max(80, 150 - (speedLevel - 1) * 10);
            }
        }

        function gameOver() {
            gameRunning = false;
            finalScoreEl.textContent = score;
            gameOverEl.style.display = 'block';
        }

        function resetGame() {
            snake = [{ x: 10, y: 10 }];
            dx = 0;
            dy = 0;
            score = 0;
            speedLevel = 1;
            speed = 150;
            scoreEl.textContent = '0';
            speedEl.textContent = '1';
            generateFood();
            gameOverEl.style.display = 'none';
            gameRunning = false;
            gamePaused = false;
        }

        function startGame() {
            if (!gameRunning && !gamePaused) {
                gameRunning = true;
                gameLoop();
            }
        }

        function pauseGame() {
            gamePaused = !gamePaused;
            if (!gamePaused && gameRunning) {
                gameLoop();
            }
        }

        function gameLoop() {
            if (!gameRunning || gamePaused) return;
            
            drawGame();
            
            if (gameRunning) {
                setTimeout(gameLoop, speed);
            }
        }

        // Controls
        document.addEventListener('keydown', (e) => {
            if (!gameRunning && !gamePaused && (e.key === 'ArrowUp' || e.key === 'ArrowDown' || e.key === 'ArrowLeft' || e.key === 'ArrowRight' || e.key.toLowerCase() === 'w' || e.key.toLowerCase() === 'a' || e.key.toLowerCase() === 's' || e.key.toLowerCase() === 'd')) {
                startGame();
            }

            // Pause/Resume
            if (e.key === ' ') {
                e.preventDefault();
                if (!gameRunning && !gamePaused) {
                    resetGame();
                    startGame();
                } else {
                    pauseGame();
                }
                return;
            }

            // Movement
            switch (e.key) {
                case 'ArrowUp':
                case 'w':
                case 'W':
                    if (dy === 0) { dx = 0; dy = -1; }
                    break;
                case 'ArrowDown':
                case 's':
                case 'S':
                    if (dy === 0) { dx = 0; dy = 1; }
                    break;
                case 'ArrowLeft':
                case 'a':
                case 'A':
                    if (dx === 0) { dx = -1; dy = 0; }
                    break;
                case 'ArrowRight':
                case 'd':
                case 'D':
                    if (dx === 0) { dx = 1; dy = 0; }
                    break;
            }
        });

        // Initialize
        resetGame();
        drawGame();
    </script>
</body>
</html>
