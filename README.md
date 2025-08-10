<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Anaglyph Block Catcher (Vanilla JS)</title>
<style>
  body {
    background: #111;
    color: #eee;
    font-family: sans-serif;
    margin: 1rem;
    display: flex;
    flex-direction: column;
    align-items: center;
  }
  canvas {
    background: #000b1a;
    border: 2px solid #222;
    border-radius: 8px;
    display: block;
    margin: 1rem 0;
  }
  label {
    display: block;
    margin: 0.5rem 0 0.25rem;
  }
  input[type=range] {
    width: 300px;
  }
  button {
    background: #0078d4;
    border: none;
    padding: 0.5rem 1rem;
    margin: 0 0.25rem;
    color: white;
    border-radius: 4px;
    cursor: pointer;
  }
  button:hover {
    background: #005ea2;
  }
  #controls {
    margin-bottom: 1rem;
  }
  #instructions {
    max-width: 360px;
    font-size: 0.85rem;
    color: #ccc;
  }
</style>
</head>
<body>

<h2>Anaglyph Block Catcher</h2>
<canvas id="gameCanvas" width="800" height="500"></canvas>

<div id="controls">
  <button id="startBtn">Start</button>
  <button id="pauseBtn" disabled>Pause</button>
  <button id="resetBtn">Reset</button>
</div>

<div>
  <label for="leftClarity">Left Eye Clarity: <span id="leftVal">100%</span></label>
  <input type="range" id="leftClarity" min="0" max="1" step="0.05" value="1" />

  <label for="rightClarity">Right Eye Clarity: <span id="rightVal">40%</span></label>
  <input type="range" id="rightClarity" min="0" max="1" step="0.05" value="0.4" />
</div>

<div id="instructions">
  <p>Wear red/cyan anaglyph glasses (red on left eye, cyan on right eye).</p>
  <p>Use left/right arrows or A/D keys to move the paddle.</p>
  <p>Catch falling blocks to score points.</p>
  <p>Adjust clarity sliders to prioritize the weaker eye’s image clarity.</p>
</div>

<script>
(() => {
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');

  const leftClaritySlider = document.getElementById('leftClarity');
  const rightClaritySlider = document.getElementById('rightClarity');
  const leftVal = document.getElementById('leftVal');
  const rightVal = document.getElementById('rightVal');

  const startBtn = document.getElementById('startBtn');
  const pauseBtn = document.getElementById('pauseBtn');
  const resetBtn = document.getElementById('resetBtn');

  const width = canvas.width;
  const height = canvas.height;

  // Game state
  let running = false;
  let score = 0;

  const paddle = {
    width: 80,
    height: 20,
    x: (width - 80) / 2,
    y: height - 40,
    speed: 350,
    dx: 0
  };

  const blocks = [];
  let lastSpawn = 0;
  const spawnInterval = 900;

  let leftClarity = parseFloat(leftClaritySlider.value);
  let rightClarity = parseFloat(rightClaritySlider.value);

  // Controls state
  const keys = {};

  function spawnBlock() {
    const size = 40;
    const x = Math.random() * (width - size);
    const speed = 70 + Math.random() * 80;
    blocks.push({ x, y: -size, w: size, h: size, speed });
  }

  function update(dt) {
    // Move paddle
    if (keys['ArrowLeft'] || keys['a']) paddle.x -= paddle.speed * dt;
    if (keys['ArrowRight'] || keys['d']) paddle.x += paddle.speed * dt;
    paddle.x = Math.max(0, Math.min(paddle.x, width - paddle.width));

    // Spawn blocks
    const now = performance.now();
    if (now - lastSpawn > spawnInterval) {
      spawnBlock();
      lastSpawn = now;
    }

    // Move blocks down
    for (let i = blocks.length - 1; i >= 0; i--) {
      const b = blocks[i];
      b.y += b.speed * dt;

      // Catch detection
      if (b.y + b.h >= paddle.y && b.y < paddle.y + paddle.height) {
        if (b.x + b.w > paddle.x && b.x < paddle.x + paddle.width) {
          blocks.splice(i, 1);
          score += 10;
          continue;
        }
      }

      // Remove blocks off screen
      if (b.y > height + 100) {
        blocks.splice(i, 1);
      }
    }
  }

  // Draw for one eye with clarity controlling block detail
  function drawEye(clarity, colorFilter) {
    // Offscreen canvas for each eye
    const off = document.createElement('canvas');
    off.width = width;
    off.height = height;
    const ctxOff = off.getContext('2d');

    // Background
    ctxOff.fillStyle = '#0b1226';
    ctxOff.fillRect(0, 0, width, height);

    // Blocks with clarity detail
    blocks.forEach(b => {
      ctxOff.fillStyle = '#999';
      ctxOff.fillRect(b.x, b.y, b.w, b.h);

      const detail = Math.round(1 + clarity * 6);
      ctxOff.fillStyle = '#333';
      const cell = Math.max(2, Math.floor(b.w / detail));
      for (let yy = 0; yy < b.h; yy += cell) {
        for (let xx = 0; xx < b.w; xx += cell) {
          if (Math.random() < 0.5) ctxOff.fillRect(b.x + xx + 1, b.y + yy + 1, cell - 2, cell - 2);
        }
      }
    });

    // Paddle
    ctxOff.fillStyle = '#a6e3ff';
    ctxOff.fillRect(paddle.x, paddle.y, paddle.width, paddle.height);

    // Score text
    ctxOff.fillStyle = 'rgba(255,255,255,0.6)';
    ctxOff.font = '18px sans-serif';
    ctxOff.fillText('Score: ' + score, 16, 28);

    // Apply color filter for anaglyph (red or cyan)
    const imgData = ctxOff.getImageData(0, 0, width, height);
    for (let i = 0; i < imgData.data.length; i += 4) {
      const r = imgData.data[i];
      const g = imgData.data[i+1];
      const b = imgData.data[i+2];
      if (colorFilter === 'red') {
        // Keep red channel, zero green and blue
        imgData.data[i] = r;
        imgData.data[i+1] = 0;
        imgData.data[i+2] = 0;
      } else if (colorFilter === 'cyan') {
        // Keep green and blue, zero red
        imgData.data[i] = 0;
        imgData.data[i+1] = g;
        imgData.data[i+2] = b;
      }
      // Apply clarity as alpha
      imgData.data[i+3] = imgData.data[i+3] * clarity;
    }
    ctxOff.putImageData(imgData, 0, 0);

    return off;
  }

  function draw() {
    ctx.clearRect(0, 0, width, height);

    // Draw left eye (red)
    const leftEyeCanvas = drawEye(leftClarity, 'red');
    ctx.drawImage(leftEyeCanvas, 0, 0);

    // Draw right eye (cyan), blend using lighter
    ctx.globalCompositeOperation = 'lighter';
    const rightEyeCanvas = drawEye(rightClarity, 'cyan');
    ctx.drawImage(rightEyeCanvas, 0, 0);
    ctx.globalCompositeOperation = 'source-over';
  }

  let lastTime = null;

  function gameLoop(timestamp) {
    if (!lastTime) lastTime = timestamp;
    const dt = (timestamp - lastTime) / 1000;
    lastTime = timestamp;

    update(dt);
    draw();

    if (running) requestAnimationFrame(gameLoop);
  }

  // Controls events
  window.addEventListener('keydown', e => {
    keys[e.key] = true;
  });
  window.addEventListener('keyup', e => {
    keys[e.key] = false;
  });

  leftClaritySlider.addEventListener('input', e => {
    leftClarity = parseFloat(e.target.value);
    leftVal.textContent = Math.round(leftClarity * 100) + '%';
  });

  rightClaritySlider.addEventListener('input', e => {
    rightClarity = parseFloat(e.target.value);
    rightVal.textContent = Math.round(rightClarity * 100) + '%';
  });

  startBtn.addEventListener('click', () => {
    if (!running) {
      running = true;
      lastSpawn = performance.now() - spawnInterval; // force immediate spawn
      lastTime = null;
      requestAnimationFrame(gameLoop);
      startBtn.disabled = true;
      pauseBtn.disabled = false;
    }
  });

  pauseBtn.addEventListener('click', () => {
    running = false;
    startBtn.disabled = false;
    pauseBtn.disabled = true;
  });

  resetBtn.addEventListener('click', () => {
    running = false;
    blocks.length = 0;
    score = 0;
    paddle.x = (width - paddle.width) / 2;
    ctx.clearRect(0, 0, width, height);
    startBtn.disabled = false;
    pauseBtn.disabled = true;
  });

  // Initial draw
  draw();

})();
</script>

</body>
</html>
