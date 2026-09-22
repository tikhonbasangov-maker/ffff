<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Dino Runner</title>
<style>
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  body {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    font-family: 'Segoe UI', Tahoma, sans-serif;
    overflow: hidden;
    user-select: none;
  }

  h1 {
    color: #fff;
    font-size: 2.5rem;
    margin-bottom: 20px;
    text-shadow: 2px 2px 8px rgba(0,0,0,0.3);
    letter-spacing: 2px;
  }

  .game-container {
    background: #f7f7f7;
    border-radius: 20px;
    padding: 20px;
    box-shadow: 0 20px 60px rgba(0,0,0,0.3);
    position: relative;
  }

  canvas {
    display: block;
    border-radius: 12px;
    background: #fff;
    cursor: pointer;
  }

  .info {
    display: flex;
    justify-content: space-between;
    margin-top: 15px;
    color: #555;
    font-weight: 600;
    font-size: 1rem;
  }

  .info span {
    background: #eee;
    padding: 6px 14px;
    border-radius: 20px;
  }

  .overlay {
    position: absolute;
    inset: 20px;
    border-radius: 12px;
    background: rgba(255,255,255,0.9);
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 16px;
    transition: opacity 0.3s;
  }

  .overlay.hidden {
    opacity: 0;
    pointer-events: none;
  }

  .overlay h2 {
    color: #333;
    font-size: 1.8rem;
  }

  .overlay p {
    color: #777;
    font-size: 1rem;
  }

  .btn {
    background: linear-gradient(135deg, #667eea, #764ba2);
    color: #fff;
    border: none;
    padding: 12px 32px;
    border-radius: 30px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: transform 0.2s, box-shadow 0.2s;
    box-shadow: 0 6px 20px rgba(102,126,234,0.5);
  }

  .btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 10px 25px rgba(102,126,234,0.6);
  }

  .btn:active {
    transform: translateY(0);
  }

  .hint {
    color: rgba(255,255,255,0.9);
    margin-top: 20px;
    font-size: 0.9rem;
  }
</style>
</head>
<body>

<h1>🦖 DINO RUNNER</h1>

<div class="game-container">
  <canvas id="game" width="800" height="300"></canvas>

  <div class="info">
    <span>Счёт: <b id="score">0</b></span>
    <span>Рекорд: <b id="best">0</b></span>
  </div>

  <div class="overlay" id="overlay">
    <h2>Готов к бегу?</h2>
    <p>Нажми ПРОБЕЛ или тапни, чтобы прыгнуть</p>
    <button class="btn" id="startBtn">Начать игру</button>
  </div>
</div>

<p class="hint">Пробел / ↑ — прыжок • Удерживай для высокого прыжка</p>

<script>
const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const scoreEl = document.getElementById('score');
const bestEl = document.getElementById('best');
const overlay = document.getElementById('overlay');
const startBtn = document.getElementById('startBtn');

const W = canvas.width;
const H = canvas.height;
const GROUND_Y = H - 50;

let best = localStorage.getItem('dinoBest') || 0;
bestEl.textContent = best;

// Состояние игры
let game = {
  running: false,
  over: false,
  speed: 6,
  score: 0,
  frame: 0,
  obstacles: [],
  clouds: [],
  groundOffset: 0,
  spawnTimer: 0
};

// Динозаврик
const dino = {
  x: 80,
  y: GROUND_Y,
  w: 44,
  h: 48,
  vy: 0,
  jumping: false,
  ducking: false,
  legFrame: 0
};

const GRAVITY = 0.9;
const JUMP_POWER = -15;

// Управление
function jump() {
  if (!game.running) {
    if (game.over) resetGame();
    return;
  }
  if (!dino.jumping) {
    dino.vy = JUMP_POWER;
    dino.jumping = true;
  }
}

document.addEventListener('keydown', (e) => {
  if (e.code === 'Space' || e.code === 'ArrowUp') {
    e.preventDefault();
    jump();
  }
});

canvas.addEventListener('pointerdown', jump);
startBtn.addEventListener('click', startGame);

// Инициализация облаков
for (let i = 0; i < 4; i++) {
  game.clouds.push({
    x: Math.random() * W,
    y: 30 + Math.random() * 80,
    speed: 0.5 + Math.random() * 0.8,
    size: 20 + Math.random() * 20
  });
}

// Рисование динозавра (пиксельный стиль)
function drawDino(x, y, w, h, legFrame, jumping, ducking) {
  ctx.fillStyle = '#4a4a4a';

  if (ducking) {
    // Присевший
    const dh = h * 0.6;
    const dy = y - dh;
    ctx.fillRect(x, dy, w * 1.3, dh * 0.6);
    ctx.fillRect(x + w * 0.5, dy - dh * 0.3, w * 0.5, dh * 0.4);
    ctx.fillRect(x + w * 1.1, dy + 5, w * 0.3, 6); // глаз
    // ноги
    ctx.fillRect(x + 5, dy + dh * 0.6, 8, dh * 0.4);
    ctx.fillRect(x + 20, dy + dh * 0.6, 8, dh * 0.4);
    return;
  }

  const bodyY = y - h;
  const px = w / 6;
  const py = h / 8;

  // Тело
  ctx.fillRect(x, bodyY + py * 2, px * 4, py * 5);

  // Шея
  ctx.fillRect(x + px * 3, bodyY + py * 0.5, px * 1.5, py * 2);

  // Голова
  ctx.fillRect(x + px * 3, bodyY, px * 3, py * 1.8);

  // Челюсть
  ctx.fillRect(x + px * 4.5, bodyY + py * 1.5, px * 1.5, py * 0.6);

  // Глаз
  ctx.fillStyle = '#fff';
  ctx.fillRect(x + px * 4.8, bodyY + py * 0.4, px * 0.6, py * 0.6);
  ctx.fillStyle = '#4a4a4a';
  ctx.fillRect(x + px * 5, bodyY + py * 0.5, px * 0.3, py * 0.4);

  // Хвост
  ctx.fillRect(x - px, bodyY + py * 3, px, py * 2);
  ctx.fillRect(x - px * 2, bodyY + py * 4, px, py);

  // Ручка
  ctx.fillRect(x + px * 3, bodyY + py * 3, px, py * 1.5);

  // Ноги
  const legOffset = jumping ? 0 : (legFrame % 2 === 0 ? 0 : py * 0.8);
  ctx.fillRect(x + px * 0.5, bodyY + py * 7, px * 1.2, py * 1 - legOffset);
  ctx.fillRect(x + px * 2.3, bodyY + py * 7, px * 1.2, py * 1 - (jumping ? 0 : (legFrame % 2 === 0 ? py * 0.8 : 0)));
}

// Кактус
function drawCactus(c) {
  ctx.fillStyle = '#3d8b3d';
  const cx = c.x;
  const cy = GROUND_Y - c.h;
  const cw = c.w;

  // Основной ствол
  ctx.fillRect(cx + cw * 0.35, cy, cw * 0.3, c.h);

  // Левая ветка
  ctx.fillRect(cx, cy + c.h * 0.3, cw * 0.35, c.h * 0.12);
  ctx.fillRect(cx, cy + c.h * 0.3 - c.h * 0.15, cw * 0.12, c.h * 0.3);

  // Правая ветка
  ctx.fillRect(cx + cw * 0.65, cy + c.h * 0.45, cw * 0.35, c.h * 0.12);
  ctx.fillRect(cx + cw * 0.88, cy + c.h * 0.45 - c.h * 0.2, cw * 0.12, c.h * 0.35);
}

// Птица
function drawBird(b) {
  ctx.fillStyle = '#333';
  const bx = b.x;
  const by = b.y;
  const wing = (game.frame % 20 < 10) ? -1 : 1;

  // Тело
  ctx.fillRect(bx, by, 30, 10);
  // Голова
  ctx.fillRect(bx + 28, by - 4, 12, 8);
  // Клюв
  ctx.fillRect(bx + 40, by, 8, 4);
  // Крыло
  ctx.fillRect(bx + 8, by - 8 * wing, 14, 8);
}

// Облако
function drawCloud(c) {
  ctx.fillStyle = 'rgba(200, 210, 230, 0.8)';
  ctx.beginPath();
  ctx.arc(c.x, c.y, c.size, 0, Math.PI * 2);
  ctx.arc(c.x + c.size * 0.8, c.y + 5, c.size * 0.8, 0, Math.PI * 2);
  ctx.arc(c.x - c.size * 0.8, c.y + 5, c.size * 0.7, 0, Math.PI * 2);
  ctx.fill();
}

// Земля
function drawGround() {
  ctx.fillStyle = '#d0d0d0';
  ctx.fillRect(0, GROUND_Y + 2, W, 2);

  ctx.fillStyle = '#b0b0b0';
  for (let i = -1; i < W / 40 + 1; i++) {
    const x = i * 40 - (game.groundOffset % 40);
    ctx.fillRect(x, GROUND_Y + 8, 20, 2);
    ctx.fillRect(x + 10, GROUND_Y + 16, 15, 2);
  }
}

// Спавн препятствий
function spawnObstacle() {
  const type = Math.random();
  if (type < 0.75) {
    // Кактус
    const h = 30 + Math.random() * 35;
    game.obstacles.push({
      x: W,
      y: GROUND_Y,
      w: 26 + Math.random() * 14,
      h: h,
      type: 'cactus',
      passed: false
    });
  } else {
    // Птица
    const heights = [GROUND_Y - 60, GROUND_Y - 100, GROUND_Y - 30];
    game.obstacles.push({
      x: W,
      y: heights[Math.floor(Math.random() * heights.length)],
      w: 48,
      h: 20,
      type: 'bird',
      passed: false
    });
  }
}

// Коллизия AABB
function hit(a, b) {
  const pad = 4;
  return a.x + pad < b.x + b.w - pad &&
         a.x + a.w - pad > b.x + pad &&
         a.y - (a.h || 0) + pad < b.y &&
         a.y + pad > b.y - (b.h || 0);
}

// Проверка коллизии динозавра с препятствием
function checkCollision(obs) {
  const dx = dino.x + 6;
  const dy = dino.y - dino.h + 6;
  const dw = dino.w - 12;
  const dh = dino.h - 12;

  let ox, oy, ow, oh;

  if (obs.type === 'cactus') {
    ox = obs.x + 4;
    oy = obs.y - obs.h;
    ow = obs.w - 8;
    oh = obs.h;
  } else {
    ox = obs.x + 4;
    oy = obs.y - 6;
    ow = obs.w - 8;
    oh = 18;
  }

  return dx < ox + ow && dx + dw > ox && dy < oy + oh && dy + dh > oy;
}

// Обновление
function update() {
  if (!game.running) return;

  game.frame++;
  game.speed = 6 + Math.min(game.score / 100, 6);
  game.score += 0.1;
  scoreEl.textContent = Math.floor(game.score);

  // Земля
  game.groundOffset += game.speed;

  // Облака
  game.clouds.forEach(c => {
    c.x -= c.speed + game.speed * 0.1;
    if (c.x < -c.size * 2) {
      c.x = W + c.size * 2;
      c.y = 30 + Math.random() * 80;
    }
  });

  // Динозавр — гравитация
  if (dino.jumping) {
    dino.vy += GRAVITY;
    dino.y += dino.vy;
    if (dino.y >= GROUND_Y) {
      dino.y = GROUND_Y;
      dino.vy = 0;
      dino.jumping = false;
    }
  }

  // Анимация ног
  if (!dino.jumping) {
    if (game.frame % 6 === 0) dino.legFrame++;
  }

  // Спавн
  game.spawnTimer--;
  if (game.spawnTimer <= 0) {
    spawnObstacle();
    const minGap = 60;
    const maxGap = 120;
    game.spawnTimer = Math.floor(minGap + Math.random() * (maxGap - minGap) - game.speed * 2);
    if (game.spawnTimer < 30) game.spawnTimer = 30;
  }

  // Движение препятствий
  for (let i = game.obstacles.length - 1; i >= 0; i--) {
    const obs = game.obstacles[i];
    obs.x -= game.speed;

    if (!obs.passed && obs.x + obs.w < dino.x) {
      obs.passed = true;
      if (obs.type === 'cactus') game.score += 10;
    }

    if (obs.x + obs.w < -50) {
      game.obstacles.splice(i, 1);
      continue;
    }

    if (checkCollision(obs)) {
      gameOver();
      return;
    }
  }
}

// Отрисовка
function draw() {
  ctx.clearRect(0, 0, W, H);

  // Небо градиент
  const grad = ctx.createLinearGradient(0, 0, 0, H);
  grad.addColorStop(0, '#e8f0ff');
  grad.addColorStop(1, '#ffffff');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, W, H);

  // Облака
  game.clouds.forEach(drawCloud);

  // Земля
  drawGround();

  // Препятствия
  game.obstacles.forEach(obs => {
    if (obs.type === 'cactus') drawCactus(obs);
    else drawBird(obs);
  });

  // Динозавр
  drawDino(dino.x, dino.y, dino.w, dino.h, dino.legFrame, dino.jumping, dino.ducking);
}

// Игровой цикл
function loop() {
  update();
  draw();
  requestAnimationFrame(loop);
}

// Старт
function startGame() {
  game.running = true;
  game.over = false;
  game.score = 0;
  game.speed = 6;
  game.frame = 0;
  game.obstacles = [];
  game.spawnTimer = 60;
  scoreEl.textContent = '0';

  dino.y = GROUND_Y;
  dino.vy = 0;
  dino.jumping = false;

  overlay.classList.add('hidden');
}

// Конец игры
function gameOver() {
  game.running = false;
  game.over = true;

  const finalScore = Math.floor(game.score);
  if (finalScore > best) {
    best = finalScore;
    localStorage.setItem('dinoBest', best);
    bestEl.textContent = best;
  }

  overlay.querySelector('h2').textContent = '💥 Игра окончена!';
  overlay.querySelector('p').textContent = `Твой счёт: ${finalScore}`;
  startBtn.textContent = 'Играть снова';
  overlay.classList.remove('hidden');
}

function resetGame() {
  startGame();
}

loop();
</script>

</body>
</html>
