<!doctype html>
<html lang="en-US">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Drive Mad — single-file</title>
  <meta name="description" content="Drive Mad — simple top-down dodger. Arrow keys / A D or touch to move." />
  <link rel="icon" href="favicon.ico" />
  <style>
    :root{
      --bg1:#071026;
      --bg2:#0b1020;
      --panel:rgba(255,255,255,0.04);
      --accent:#1ea7fd;
      --text:#e6eef8;
    }
    *{box-sizing:border-box}
    html,body{height:100%}
    body{
      margin:0;
      font-family:Inter,system-ui,-apple-system,Segoe UI,Roboto,"Helvetica Neue",Arial;
      background:linear-gradient(180deg,var(--bg1),var(--bg2));
      color:var(--text);
      display:flex;
      align-items:center;
      justify-content:center;
      padding:18px;
    }

    /* Container */
    .wrapper{
      width:100%;
      max-width:1000px;
      display:flex;
      flex-direction:column;
      gap:12px;
    }

    /* HUD */
    .hud{
      display:flex;
      justify-content:space-between;
      align-items:center;
      padding:8px 12px;
      background:var(--panel);
      border-radius:10px;
    }
    .hud .title{font-weight:800}
    .hud .score{font-weight:600}

    /* Stage */
    .stage{
      position:relative;
      border-radius:12px;
      overflow:hidden;
      box-shadow:0 12px 40px rgba(2,6,23,0.6);
      background:linear-gradient(180deg,#081322, #061424 80%);
      min-height:420px;
      height:65vh;
      max-height:760px;
    }
    canvas{
      display:block;
      width:100%;
      height:100%;
      background:transparent;
    }

    /* Overlay */
    #overlay{
      position:absolute;
      inset:0;
      display:flex;
      align-items:center;
      justify-content:center;
      pointer-events:none;
    }
    .panel{
      width:min(480px,92%);
      background:linear-gradient(180deg, rgba(6,10,20,0.92), rgba(12,16,28,0.9));
      color:var(--text);
      padding:20px;
      border-radius:12px;
      text-align:center;
      pointer-events:auto;
    }
    .hidden{display:none}

    .btn{
      margin-top:14px;
      padding:10px 18px;
      background:var(--accent);
      color:#001;
      border:0;
      border-radius:10px;
      font-weight:700;
      cursor:pointer;
    }

    /* Mobile controls */
    .controls{
      display:flex;
      gap:12px;
      justify-content:center;
      margin-top:8px;
    }
    .touch-btn{
      background:var(--panel);
      color:var(--text);
      border:0;
      padding:12px 18px;
      font-size:1.2rem;
      border-radius:10px;
      user-select:none;
    }
    .touch-btn:active{transform:translateY(1px)}

    footer{
      margin-top:6px;
      opacity:0.85;
      text-align:center;
      font-size:0.9rem;
    }

    @media (max-width:600px){
      .hud{font-size:0.95rem}
    }
  </style>
</head>
<body>
  <div class="wrapper" role="application" aria-label="Drive Mad game">
    <header class="hud" aria-hidden="false">
      <div class="title">Drive Mad</div>
      <div class="score">Score: <span id="score">0</span></div>
      <div class="score">Best: <span id="highscore">0</span></div>
    </header>

    <main class="stage" id="stage">
      <canvas id="gameCanvas" width="800" height="600" aria-label="Drive Mad canvas"></canvas>

      <div id="overlay" aria-live="polite">
        <div id="menu" class="panel">
          <h1 style="margin:0 0 8px 0">Drive Mad</h1>
          <p style="margin:0 0 12px 0">Steer left & right to avoid obstacles. Survive as long as you can.</p>
          <button id="startBtn" class="btn">Start</button>
        </div>

        <div id="gameOver" class="panel hidden" role="status">
          <h2 style="margin:0 0 8px 0">Game Over</h2>
          <p style="margin:0 0 8px 0">Your score: <strong id="finalScore">0</strong></p>
          <p style="margin:0 0 12px 0">Best: <strong id="finalHigh">0</strong></p>
          <button id="restartBtn" class="btn">Play Again</button>
        </div>
      </div>
    </main>

    <nav class="controls" aria-hidden="false">
      <button id="leftBtn" class="touch-btn" aria-label="Move left">◀</button>
      <button id="rightBtn" class="touch-btn" aria-label="Move right">▶</button>
    </nav>

    <footer>Controls: ← → or A D (desktop). Hold touch buttons on mobile. High score saved locally.</footer>
  </div>

  <script>
    // Single-file Drive Mad
    (function () {
      const canvas = document.getElementById('gameCanvas');
      const ctx = canvas.getContext('2d', { alpha: false });
      const overlay = document.getElementById('overlay');
      const menu = document.getElementById('menu');
      const gameOverUI = document.getElementById('gameOver');
      const startBtn = document.getElementById('startBtn');
      const restartBtn = document.getElementById('restartBtn');
      const leftBtn = document.getElementById('leftBtn');
      const rightBtn = document.getElementById('rightBtn');
      const scoreEl = document.getElementById('score');
      const highEl = document.getElementById('highscore');
      const finalScoreEl = document.getElementById('finalScore');
      const finalHighEl = document.getElementById('finalHigh');

      const STORAGE_KEY = 'drive_mad_highscore_v1';
      let highscore = Number(localStorage.getItem(STORAGE_KEY) || 0);
      highEl.textContent = highscore;

      // Logical canvas size (game world)
      const W = 800, H = 600;
      // Resize for device pixel ratio and container size
      function fitCanvas() {
        const rect = canvas.getBoundingClientRect();
        const dpr = Math.min(window.devicePixelRatio || 1, 2);
        canvas.width = Math.max(300, Math.round(W * (rect.width / rect.width))) * dpr;
        canvas.height = Math.max(200, Math.round(H * (rect.height / rect.height))) * dpr;
        // We'll use a virtual coordinate system; scale transform later per draw
        ctx.setTransform(dpr * (canvas.width / canvas.clientWidth), 0, 0, dpr * (canvas.height / canvas.clientHeight), 0, 0);
      }
      // Simpler approach: keep internal size fixed, scale via CSS; use logical W/H
      function setupCanvasForDrawing() {
        // set internal resolution to W x H for clarity
        const dpr = Math.min(window.devicePixelRatio || 1, 2);
        canvas.width = Math.round(W * dpr);
        canvas.height = Math.round(H * dpr);
        canvas.style.width = '100%';
        canvas.style.height = '100%';
        ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
      }
      setupCanvasForDrawing();
      window.addEventListener('resize', () => { setupCanvasForDrawing(); });

      // Game state
      let running = false;
      let lastTime = 0;
      let score = 0;
      let obstacles = [];
      let spawnTimer = 0;
      let spawnInterval = 0.9; // sec
      let gameSpeed = 160; // base obstacle speed px/sec
      let difficulty = 0; // increases over time

      // Player
      const player = {
        w: 48,
        h: 80,
        x: (W - 48) / 2,
        y: H - 110,
        vx: 0,
        speed: 420, // px/sec
        color: '#ffd45e'
      };

      // Input
      const keys = { left: false, right: false };
      function keyDown(e) {
        if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') keys.left = true;
        if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') keys.right = true;
      }
      function keyUp(e) {
        if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') keys.left = false;
        if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') keys.right = false;
      }
      window.addEventListener('keydown', keyDown);
      window.addEventListener('keyup', keyUp);

      // Touch buttons (hold)
      let touchLeft = false, touchRight = false;
      leftBtn.addEventListener('touchstart', (e) => { e.preventDefault(); touchLeft = true; }, { passive: false });
      leftBtn.addEventListener('touchend', (e) => { e.preventDefault(); touchLeft = false; }, { passive: false });
      rightBtn.addEventListener('touchstart', (e) => { e.preventDefault(); touchRight = true; }, { passive: false });
      rightBtn.addEventListener('touchend', (e) => { e.preventDefault(); touchRight = false; }, { passive: false });
      // also support mouse down for buttons
      leftBtn.addEventListener('mousedown', () => touchLeft = true);
      leftBtn.addEventListener('mouseup', () => touchLeft = false);
      rightBtn.addEventListener('mousedown', () => touchRight = true);
      rightBtn.addEventListener('mouseup', () => touchRight = false);
      // prevent context menu on long press
      [leftBtn, rightBtn].forEach(b => b.addEventListener('contextmenu', e => e.preventDefault()));

      // Utility
      function rand(min, max) { return Math.random() * (max - min) + min; }
      function rectsOverlap(a, b) {
        return !(a.x + a.w < b.x || a.x > b.x + b.w || a.y + a.h < b.y || a.y > b.y + b.h);
      }

      // Spawn obstacles
      function spawnObstacle() {
        const lanePadding = 24;
        // varying widths
        const w = Math.round(rand(40, 140));
        const x = Math.round(rand(lanePadding, W - lanePadding - w));
        const h = Math.round(rand(28, 80));
        const speed = gameSpeed + rand(0, difficulty * 40);
        obstacles.push({
          x, y: -h - 10, w, h,
          speed,
          color: `hsl(${rand(0, 40)}, 90%, ${rand(35, 55)}%)`
        });
      }

      // Start / reset / stop
      function reset() {
        score = 0;
        obstacles = [];
        spawnTimer = 0;
        spawnInterval = 0.9;
        gameSpeed = 160;
        difficulty = 0;
        player.x = (W - player.w) / 2;
        player.vx = 0;
        scoreEl.textContent = '0';
      }

      function startGame() {
        reset();
        running = true;
        lastTime = performance.now();
        overlay.style.pointerEvents = 'none';
        menu.classList.add('hidden');
        gameOverUI.classList.add('hidden');
        menu.classList.remove('hidden'); // ensure menu remains hidden visually via class toggles
        requestAnimationFrame(loop);
      }

      function endGame() {
        running = false;
        overlay.style.pointerEvents = 'auto';
        gameOverUI.classList.remove('hidden');
        menu.classList.add('hidden');
        finalScoreEl.textContent = Math.floor(score);
        if (score > highscore) {
          highscore = Math.floor(score);
          localStorage.setItem(STORAGE_KEY, highscore);
        }
        finalHighEl.textContent = highscore;
        highEl.textContent = highscore;
      }

      startBtn.addEventListener('click', startGame);
      restartBtn.addEventListener('click', startGame);

      // Main loop
      function loop(now) {
        if (!running) return;
        const dt = Math.min((now - lastTime) / 1000, 0.05);
        lastTime = now;

        // update difficulty
        difficulty += dt * 0.12; // slow ramp
        spawnInterval = Math.max(0.35, 0.9 - difficulty * 0.12);
        gameSpeed = 160 + difficulty * 48;

        // spawn
        spawnTimer -= dt;
        if (spawnTimer <= 0) {
          spawnObstacle();
          spawnTimer = spawnInterval;
          // sometimes spawn a second obstacle close to create challenge
          if (Math.random() < Math.min(0.25 + difficulty * 0.05, 0.6)) {
            // small delayed spawn
            setTimeout(() => {
              if (running) spawnObstacle();
            }, 120);
          }
        }

        // input
        const moveLeft = keys.left || touchLeft;
        const moveRight = keys.right || touchRight;
        if (moveLeft && !moveRight) player.vx = -player.speed;
        else if (moveRight && !moveLeft) player.vx = player.speed;
        else player.vx = 0;

        // update player
        player.x += player.vx * dt;
        player.x = Math.max(8, Math.min(W - player.w - 8, player.x));

        // update obstacles
        for (let i = obstacles.length - 1; i >= 0; i--) {
          const ob = obstacles[i];
          ob.y += ob.speed * dt;
          // remove off screen
          if (ob.y > H + 100) obstacles.splice(i, 1);
        }

        // collisions
        const pBox = { x: player.x, y: player.y, w: player.w, h: player.h };
        for (let ob of obstacles) {
          const b = { x: ob.x, y: ob.y, w: ob.w, h: ob.h };
          if (rectsOverlap(pBox, b)) {
            endGame();
            break;
          }
        }

        // scoring
        score += dt * (1 + difficulty * 0.6) * 12;
        scoreEl.textContent = Math.floor(score);

        // draw
        render();

        if (running) requestAnimationFrame(loop);
      }

      // rendering
      function render() {
        // clear (draw road background)
        ctx.clearRect(0, 0, W, H);

        // background gradient
        const g = ctx.createLinearGradient(0, 0, 0, H);
        g.addColorStop(0, '#071b2b');
        g.addColorStop(1, '#05121a');
        ctx.fillStyle = g;
        ctx.fillRect(0, 0, W, H);

        // road stripes / subtle perspective lines
        ctx.save();
        ctx.globalAlpha = 0.06;
        ctx.strokeStyle = '#ffffff';
        ctx.lineWidth = 2;
        for (let i = 0; i < 12; i++) {
          const y = (i * 70 + (score % 70));
          ctx.beginPath();
          ctx.moveTo(0, y);
          ctx.lineTo(W, y + i * 6);
          ctx.stroke();
        }
        ctx.restore();

        // draw obstacles
        for (let ob of obstacles) {
          ctx.fillStyle = ob.color;
          // add subtle shadow
          ctx.fillRect(ob.x, ob.y, ob.w, ob.h);
          ctx.strokeStyle = 'rgba(0,0,0,0.18)';
          ctx.strokeRect(ob.x + 0.5, ob.y + 0.5, ob.w - 1, ob.h - 1);
        }

        // draw player (simple car)
        const px = player.x, py = player.y, pw = player.w, ph = player.h;
        // body
        ctx.fillStyle = player.color;
        roundRect(ctx, px, py, pw, ph, 8, true, false);
        // windows
        ctx.fillStyle = 'rgba(0,0,0,0.22)';
        roundRect(ctx, px + pw * 0.12, py + ph * 0.12, pw * 0.76, ph * 0.36, 6, true, false);
        // wheels (simple)
        ctx.fillStyle = '#0b1016';
        const wheelH = Math.max(6, ph * 0.16);
        ctx.fillRect(px + 6, py + ph - wheelH / 2, 12, wheelH);
        ctx.fillRect(px + pw - 18, py + ph - wheelH / 2, 12, wheelH);

        // HUD overlay (top-left)
        ctx.fillStyle = 'rgba(0,0,0,0.18)';
        ctx.fillRect(12, 12, 150, 32);
        ctx.fillStyle = '#dff3ff';
        ctx.font = '16px Inter, system-ui, Arial';
        ctx.fillText('Score: ' + Math.floor(score), 20, 34);
      }

      // helper: rounded rect
      function roundRect(ctx, x, y, w, h, r, fill, stroke) {
        if (typeof r === 'undefined') r = 5;
        ctx.beginPath();
        ctx.moveTo(x + r, y);
        ctx.arcTo(x + w, y, x + w, y + h, r);
        ctx.arcTo(x + w, y + h, x, y + h, r);
        ctx.arcTo(x, y + h, x, y, r);
        ctx.arcTo(x, y, x + w, y, r);
        ctx.closePath();
        if (fill) ctx.fill();
        if (stroke) ctx.stroke();
      }

      // initial draw (menu visible)
      render();

      // Expose a small debug start via click of canvas
      canvas.addEventListener('click', () => {
        if (!running && menu.classList.contains('hidden')) startGame();
      });

      // Make sure overlay visibility toggles are correct at load
      menu.classList.remove('hidden');
      gameOverUI.classList.add('hidden');

      // Accessibility: announce start
      startBtn.addEventListener('click', () => {
        startBtn.setAttribute('aria-pressed', 'true');
      });

      // Ensure highscore shown in game over
      function updateHighDisplay() {
        highEl.textContent = highscore;
      }
      updateHighDisplay();

      // Pause when tab hidden
      document.addEventListener('visibilitychange', () => {
        if (document.hidden && running) {
          // simple pause: stop running and show menu
          running = false;
          overlay.style.pointerEvents = 'auto';
          menu.classList.remove('hidden');
        }
      });
    })();
  </script>
</body>
</html>
