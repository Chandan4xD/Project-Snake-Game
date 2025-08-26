<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Snake (Self-Death Only, No Overlay, Wall Wrap)</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
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
    * { box-sizing: border-box; }
    html, body { height: 100%; margin: 0; background: var(--bg); color: var(--text); font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial; }
    .wrap { min-height: 100%; display: flex; align-items: center; justify-content: center; padding: 24px; }
    .container { width: 100%; max-width: 520px; display: grid; gap: 12px; }
    .header { display: flex; align-items: baseline; justify-content: space-between; }
    .title { font-weight: 700; font-size: 20px; }
    .subtitle { opacity: 0.8; font-size: 13px; }
    .hud { display: flex; gap: 10px; align-items: center; justify-content: space-between; }
    .badge, .btn {
      background: #0b1020; border: 1px solid #1f2937; color: var(--text);
      padding: 8px 12px; border-radius: 10px; font-size: 13px;
    }
    .btn { cursor: pointer; font-weight: 600; }
    .board { border-radius: 16px; overflow: hidden; background: linear-gradient(180deg, #0b1020, var(--board)); aspect-ratio: 1; box-shadow: inset 0 0 0 1px rgba(255,255,255,0.05); }
    canvas { width: 100%; height: 100%; image-rendering: pixelated; display: block; }
    .kbd { display: inline-block; padding: 2px 6px; border-radius: 6px; border: 1px solid #334155; background: #0b1020; font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace; font-size: 12px; margin: 0 2px; }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="container">
      <div class="header">
        <div>
          <div class="title">Snake</div>
          <div class="subtitle">Walls wrap • Dies on self-collision • Space to pause/restart</div>
        </div>
        <button id="resetBtn" class="btn">Reset</button>
      </div>

      <div class="hud">
        <div class="badge">Score: <strong id="scoreEl" style="margin-left:6px">0</strong></div>
        <div class="badge">High: <strong id="highEl" style="margin-left:6px">0</strong></div>
        <div class="badge">Speed: <strong id="speedEl" style="margin-left:6px">10</strong> fps</div>
      </div>

      <div class="board">
        <canvas id="game" width="400" height="400"></canvas>
      </div>

      <div class="subtitle">
        Controls: <span class="kbd">↑</span><span class="kbd">↓</span><span class="kbd">←</span><span class="kbd">→</span>, <span class="kbd">Space</span> pause/resume or restart when dead.
      </div>
    </div>
  </div>

  <script>
    // Grid and timing
    const CELL = 20, COLS = 20, ROWS = 20;
    const START_LEN = 4;
    const BASE_FPS = 10, SPEED_STEP = 0.5, MAX_FPS = 20;

    // DOM
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    const scoreEl = document.getElementById('scoreEl');
    const highEl = document.getElementById('highEl');
    const speedEl = document.getElementById('speedEl');
    const resetBtn = document.getElementById('resetBtn');

    // State
    let snake = [];
    let dir = { x: 1, y: 0 };        // applied direction
    let nextDir = { x: 1, y: 0 };    // queued direction from input
    let inputLocked = false;         // avoid multiple turns per tick
    let food = { x: 0, y: 0 };
    let score = 0;
    let high = Number(localStorage.getItem('snake_high') || 0);
    let fps = BASE_FPS;
    let lastTick = 0;
    let paused = false;
    let gameOver = false;
    let rafId = 0;

    highEl.textContent = high;

    function init() {
      // Reset flags and timers
      paused = false;
      gameOver = false;
      lastTick = 0;
      cancelAnimationFrame(rafId);

      // Centered snake
      const startX = Math.floor(COLS / 2);
      const startY = Math.floor(ROWS / 2);
      snake = [];
      for (let i = 0; i < START_LEN; i++) snake.push({ x: startX - i, y: startY });

      dir = { x: 1, y: 0 };
      nextDir = { x: 1, y: 0 };
      inputLocked = false;

      score = 0;
      fps = BASE_FPS;
      food = spawnFood();
      updateHUD();

      rafId = requestAnimationFrame(loop);
    }

    function spawnFood() {
      let fx, fy;
      do {
        fx = Math.floor(Math.random() * COLS);
        fy = Math.floor(Math.random() * ROWS);
      } while (snake.some(s => s.x === fx && s.y === fy));
      return { x: fx, y: fy };
    }

    function updateHUD() {
      scoreEl.textContent = score;
      highEl.textContent = high;
      speedEl.textContent = fps.toFixed(1).replace(/\.0$/, '');
    }

    function endGame() {
      gameOver = true;    // stop loop; no overlay shown
    }

    function handleKey(e) {
      const k = e.key;
      if (k === ' ' || k === 'Spacebar') {
        if (gameOver) { init(); return; }   // restart when dead
        paused = !paused;
        if (!paused) rafId = requestAnimationFrame(loop);
        return;
      }
      if (gameOver || paused) return;

      // Prevent reversing directly and multiple changes per tick
      if (inputLocked) return;

      if (k === 'ArrowUp' && dir.y !== 1)        { nextDir = { x: 0, y: -1 }; inputLocked = true; }
      else if (k === 'ArrowDown' && dir.y !== -1){ nextDir = { x: 0, y: 1 };  inputLocked = true; }
      else if (k === 'ArrowLeft' && dir.x !== 1) { nextDir = { x: -1, y: 0 }; inputLocked = true; }
      else if (k === 'ArrowRight' && dir.x !== -1){ nextDir = { x: 1, y: 0 };  inputLocked = true; }
    }

    document.addEventListener('keydown', handleKey);
    resetBtn.addEventListener('click', init);

    function loop(ts) {
      if (paused || gameOver) return;

      const interval = 1000 / fps;
      if (ts - lastTick < interval) {
        render();
        rafId = requestAnimationFrame(loop);
        return;
      }
      lastTick = ts;

      // Apply queued direction once per tick
      dir = nextDir;
      inputLocked = false;

      const head = snake[0];
      let nx = head.x + dir.x;
      let ny = head.y + dir.y;

      // Wall WRAP (no wall death)
      if (nx < 0) nx = COLS - 1;
      else if (nx >= COLS) nx = 0;
      if (ny < 0) ny = ROWS - 1;
      else if (ny >= ROWS) ny = 0;

      // Self-collision = death
      if (snake.some(s => s.x === nx && s.y === ny)) {
        endGame();
        render();           // final frame
        return;
      }

      // Move
      snake.unshift({ x: nx, y: ny });

      // Eat or move tail
      if (nx === food.x && ny === food.y) {
        score += 1;
        if (score > high) {
          high = score;
          localStorage.setItem('snake_high', String(high));
        }
        fps = Math.min(MAX_FPS, fps + SPEED_STEP);
        updateHUD();
        food = spawnFood();
      } else {
        snake.pop();
      }

      render();
      rafId = requestAnimationFrame(loop);
    }

    function render() {
      // Background
      ctx.fillStyle = '#0b1020';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      // Grid
      ctx.strokeStyle = getComputedStyle(document.documentElement).getPropertyValue('--grid').trim() || '#1f2937';
      ctx.lineWidth = 1;
      ctx.beginPath();
      for (let x = CELL; x < canvas.width; x += CELL) {
        ctx.moveTo(x + 0.5, 0);
        ctx.lineTo(x + 0.5, canvas.height);
      }
      for (let y = CELL; y < canvas.height; y += CELL) {
        ctx.moveTo(0, y + 0.5);
        ctx.lineTo(canvas.width, y + 0.5);
      }
      ctx.stroke();

      // Food
      drawCell(food.x, food.y, '#ef4444');
      // Snake (head distinct)
      for (let i = snake.length - 1; i >= 0; i--) {
        const s = snake[i];
        drawCell(s.x, s.y, i === 0 ? '#16a34a' : '#22c55e');
      }

      // Optional subtle paused/dead indicator (no overlay)
      if (paused || gameOver) {
        ctx.fillStyle = 'rgba(0,0,0,0.25)';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
      }
    }

    function drawCell(gx, gy, color) {
      const x = gx * CELL, y = gy * CELL, pad = 2;
      ctx.fillStyle = color;
      ctx.fillRect(x + pad, y + pad, CELL - pad * 2, CELL - pad * 2);
      ctx.fillStyle = 'rgba(255,255,255,0.06)';
      ctx.fillRect(x + pad, y + pad, CELL - pad * 2, 4);
    }

    // Prevent page scroll on arrows/space
    window.addEventListener('keydown', (e) => {
      if (['ArrowUp','ArrowDown','ArrowLeft','ArrowRight',' '].includes(e.key)) e.preventDefault();
    }, { passive: false });

    // Start
    init();
  </script>
</body>
</html>
