<!doctype html>
<html lang="en" class="h-full">
 <head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Retro Snake</title>
  <script src="https://cdn.tailwindcss.com/3.4.17"></script>
  <script src="/_sdk/element_sdk.js"></script>
  <link href="https://fonts.googleapis.com/css2?family=Press+Start+2P&amp;display=swap" rel="stylesheet">
  <style>
  html, body { height: 100%; margin: 0; }
  * { font-family: 'Press Start 2P', monospace; }
  body { background: #0a0a0a; }

  .crt-overlay {
    pointer-events: none;
    position: fixed; top: 0; left: 0; right: 0; bottom: 0;
    background: repeating-linear-gradient(
      0deg, rgba(0,0,0,0.15) 0px, rgba(0,0,0,0.15) 1px, transparent 1px, transparent 3px
    );
    z-index: 50;
  }

  .glow-green { text-shadow: 0 0 8px #39ff14, 0 0 20px #39ff1466; }
  .glow-red { text-shadow: 0 0 8px #ff3939, 0 0 20px #ff393966; }
  .glow-yellow { text-shadow: 0 0 8px #ffe600, 0 0 20px #ffe60066; }

  .cabinet {
    background: linear-gradient(145deg, #1a1a2e 0%, #0f0f1a 50%, #1a1a2e 100%);
    border: 3px solid #2a2a4a;
    box-shadow: inset 0 0 40px rgba(0,0,0,0.8), 0 0 30px rgba(57,255,20,0.05);
  }

  .screen-bezel {
    background: #050508;
    border: 2px solid #333;
    box-shadow: inset 0 0 20px rgba(0,0,0,1), 0 0 8px rgba(57,255,20,0.1);
  }

  canvas { image-rendering: pixelated; }

  @keyframes blink { 0%,100%{opacity:1} 50%{opacity:0.3} }
  .blink { animation: blink 1s infinite; }

  .btn-retro {
    background: #1a1a2e;
    border: 2px solid #39ff14;
    color: #39ff14;
    text-shadow: 0 0 6px #39ff14;
    transition: all 0.15s;
  }
  .btn-retro:hover { background: #39ff1422; }
  .btn-retro:active { transform: scale(0.95); }

  .dpad-btn {
    width: 44px; height: 44px;
    background: #222;
    border: 2px solid #555;
    border-radius: 4px;
    color: #aaa;
    display: flex; align-items: center; justify-content: center;
    user-select: none; cursor: pointer;
    transition: all 0.1s;
  }
  .dpad-btn:active { background: #39ff1433; border-color: #39ff14; color: #39ff14; }
</style>
  <style>body { box-sizing: border-box; }</style>
  <script src="https://cdn.jsdelivr.net/npm/lucide@0.263.0/dist/umd/lucide.min.js" type="text/javascript"></script>
  <script src="/_sdk/data_sdk.js" type="text/javascript"></script>
 </head>
 <body class="h-full flex items-center justify-center p-4">
  <div class="crt-overlay"></div>
  <div class="cabinet rounded-xl p-4 sm:p-6 w-full max-w-lg relative z-10"><!-- Title -->
   <h1 id="gameTitle" class="text-center text-sm sm:text-base glow-green text-green-400 mb-3 tracking-wider">🐍 RETRO SNAKE</h1><!-- Score bar -->
   <div class="flex justify-between items-center mb-2 px-1 text-[9px] sm:text-[10px]"><span class="text-green-500 glow-green">SCORE: <span id="score">0</span></span> <span class="text-yellow-400 glow-yellow">HI: <span id="hiScore">0</span></span>
   </div><!-- Screen -->
   <div class="screen-bezel rounded-md p-1.5 sm:p-2">
    <div class="relative">
     <canvas id="gameCanvas"></canvas><!-- Overlays -->
     <div id="startOverlay" class="absolute inset-0 flex flex-col items-center justify-center bg-black/80 rounded">
      <p class="text-green-400 glow-green text-[10px] sm:text-xs mb-4 blink">PRESS START</p><button onclick="startGame()" class="btn-retro px-4 py-2 text-[9px] sm:text-[10px] rounded">▶ START</button>
     </div>
     <div id="gameOverOverlay" class="absolute inset-0 flex-col items-center justify-center bg-black/85 rounded hidden">
      <p class="text-red-400 glow-red text-xs sm:text-sm mb-1">GAME OVER</p>
      <p class="text-green-400 glow-green text-[9px] mt-2">SCORE: <span id="finalScore">0</span></p><button onclick="startGame()" class="btn-retro px-4 py-2 text-[9px] sm:text-[10px] rounded mt-4">↻ RETRY</button>
     </div>
    </div>
   </div><!-- D-Pad controls -->
   <div class="mt-4 flex justify-center">
    <div class="grid grid-cols-3 gap-1" style="width:140px;">
     <div></div><button class="dpad-btn" onclick="changeDir(0,-1)" aria-label="Up">▲</button>
     <div></div><button class="dpad-btn" onclick="changeDir(-1,0)" aria-label="Left">◀</button>
     <div class="w-11 h-11 bg-[#1a1a1a] rounded border border-[#333]"></div><button class="dpad-btn" onclick="changeDir(1,0)" aria-label="Right">▶</button>
     <div></div><button class="dpad-btn" onclick="changeDir(0,1)" aria-label="Down">▼</button>
     <div></div>
    </div>
   </div>
   <p class="text-center text-[8px] text-gray-600 mt-3">ARROW KEYS / WASD / SWIPE</p>
  </div>
  <script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const GRID = 20;
let cols, rows, cellSize;
let snake, dir, nextDir, food, score, hiScore = 0, running = false, loop;

function sizeCanvas() {
  const w = canvas.parentElement.clientWidth;
  cellSize = Math.floor(w / GRID);
  cols = GRID;
  rows = GRID;
  canvas.width = cols * cellSize;
  canvas.height = rows * cellSize;
  if (!running) drawStatic();
}

function drawStatic() {
  ctx.fillStyle = '#050508';
  ctx.fillRect(0, 0, canvas.width, canvas.height);
  // grid lines
  ctx.strokeStyle = '#0a0f0a';
  ctx.lineWidth = 0.5;
  for (let x = 0; x <= cols; x++) { ctx.beginPath(); ctx.moveTo(x*cellSize,0); ctx.lineTo(x*cellSize,rows*cellSize); ctx.stroke(); }
  for (let y = 0; y <= rows; y++) { ctx.beginPath(); ctx.moveTo(0,y*cellSize); ctx.lineTo(cols*cellSize,y*cellSize); ctx.stroke(); }
}

function spawnFood() {
  let pos;
  do {
    pos = { x: Math.floor(Math.random()*cols), y: Math.floor(Math.random()*rows) };
  } while (snake.some(s => s.x === pos.x && s.y === pos.y));
  food = pos;
}

function startGame() {
  document.getElementById('startOverlay').classList.add('hidden');
  document.getElementById('gameOverOverlay').classList.add('hidden');
  snake = [{x:Math.floor(cols/2), y:Math.floor(rows/2)}];
  dir = {x:1,y:0}; nextDir = {x:1,y:0};
  score = 0; updateScore();
  spawnFood();
  running = true;
  clearInterval(loop);
  loop = setInterval(tick, 120);
}

function tick() {
  dir = nextDir;
  const head = { x: snake[0].x + dir.x, y: snake[0].y + dir.y };
  // wall or self collision
  if (head.x < 0 || head.x >= cols || head.y < 0 || head.y >= rows || snake.some(s => s.x === head.x && s.y === head.y)) {
    running = false;
    clearInterval(loop);
    if (score > hiScore) { hiScore = score; document.getElementById('hiScore').textContent = hiScore; }
    document.getElementById('finalScore').textContent = score;
    document.getElementById('gameOverOverlay').classList.remove('hidden');
    document.getElementById('gameOverOverlay').style.display = 'flex';
    return;
  }
  snake.unshift(head);
  if (head.x === food.x && head.y === food.y) {
    score += 10;
    updateScore();
    spawnFood();
  } else {
    snake.pop();
  }
  draw();
}

function draw() {
  drawStatic();
  const c = cellSize;
  // food
  ctx.fillStyle = '#ff3939';
  ctx.shadowColor = '#ff3939'; ctx.shadowBlur = 8;
  ctx.fillRect(food.x*c+2, food.y*c+2, c-4, c-4);
  ctx.shadowBlur = 0;
  // snake
  snake.forEach((s, i) => {
    const brightness = Math.max(0.4, 1 - i * 0.03);
    ctx.fillStyle = i === 0 ? '#39ff14' : `rgba(57,255,20,${brightness})`;
    if (i === 0) { ctx.shadowColor = '#39ff14'; ctx.shadowBlur = 10; }
    ctx.fillRect(s.x*c+1, s.y*c+1, c-2, c-2);
    if (i === 0) ctx.shadowBlur = 0;
  });
}

function updateScore() {
  document.getElementById('score').textContent = score;
}

function changeDir(x, y) {
  if (!running) return;
  if (dir.x === -x && dir.y === -y) return; // no reversing
  nextDir = {x, y};
}

// Keyboard
document.addEventListener('keydown', e => {
  const map = {ArrowUp:[0,-1],ArrowDown:[0,1],ArrowLeft:[-1,0],ArrowRight:[1,0],w:[0,-1],s:[0,1],a:[-1,0],d:[1,0]};
  const d = map[e.key];
  if (d) { e.preventDefault(); changeDir(d[0], d[1]); }
  if (e.key === ' ' || e.key === 'Enter') {
    if (!running) startGame();
  }
});

// Swipe
let tx, ty;
canvas.addEventListener('touchstart', e => { tx = e.touches[0].clientX; ty = e.touches[0].clientY; }, {passive:true});
canvas.addEventListener('touchend', e => {
  const dx = e.changedTouches[0].clientX - tx, dy = e.changedTouches[0].clientY - ty;
  if (Math.abs(dx) < 20 && Math.abs(dy) < 20) return;
  if (Math.abs(dx) > Math.abs(dy)) changeDir(dx > 0 ? 1 : -1, 0);
  else changeDir(0, dy > 0 ? 1 : -1);
});

window.addEventListener('resize', sizeCanvas);
sizeCanvas();

// Element SDK
const defaultConfig = {
  game_title: '🐍 RETRO SNAKE',
  background_color: '#0a0a0a',
  accent_color: '#39ff14',
  food_color: '#ff3939',
  text_color: '#9ca3af',
  font_family: 'Press Start 2P',
  font_size: 14
};

function applyConfig(config) {
  const title = config.game_title || defaultConfig.game_title;
  const bg = config.background_color || defaultConfig.background_color;
  const accent = config.accent_color || defaultConfig.accent_color;
  const foodCol = config.food_color || defaultConfig.food_color;
  const txt = config.text_color || defaultConfig.text_color;
  const font = config.font_family || defaultConfig.font_family;
  const sz = config.font_size || defaultConfig.font_size;

  document.getElementById('gameTitle').textContent = title;
  document.body.style.background = bg;
  document.getElementById('gameTitle').style.fontSize = sz + 'px';
  document.getElementById('gameTitle').style.fontFamily = `${font}, monospace`;
  document.getElementById('gameTitle').style.color = accent;

  document.querySelectorAll('.btn-retro').forEach(b => {
    b.style.borderColor = accent; b.style.color = accent;
  });
  document.querySelectorAll('.glow-green').forEach(el => {
    el.style.color = accent;
    el.style.textShadow = `0 0 8px ${accent}, 0 0 20px ${accent}66`;
  });
}

if (window.elementSdk) {
  window.elementSdk.init({
    defaultConfig,
    onConfigChange: async (config) => applyConfig(config),
    mapToCapabilities: (config) => ({
      recolorables: [
        { get: () => config.background_color || defaultConfig.background_color, set: v => { config.background_color = v; window.elementSdk.setConfig({background_color:v}); }},
        { get: () => config.accent_color || defaultConfig.accent_color, set: v => { config.accent_color = v; window.elementSdk.setConfig({accent_color:v}); }},
        { get: () => config.food_color || defaultConfig.food_color, set: v => { config.food_color = v; window.elementSdk.setConfig({food_color:v}); }},
        { get: () => config.text_color || defaultConfig.text_color, set: v => { config.text_color = v; window.elementSdk.setConfig({text_color:v}); }},
      ],
      borderables: [],
      fontEditable: { get: () => config.font_family || defaultConfig.font_family, set: v => { config.font_family = v; window.elementSdk.setConfig({font_family:v}); }},
      fontSizeable: { get: () => config.font_size || defaultConfig.font_size, set: v => { config.font_size = v; window.elementSdk.setConfig({font_size:v}); }},
    }),
    mapToEditPanelValues: (config) => new Map([
      ['game_title', config.game_title || defaultConfig.game_title]
    ])
  });
}
</script>
 <script>(function(){function c(){var b=a.contentDocument||a.contentWindow.document;if(b){var d=b.createElement('script');d.innerHTML="window.__CF$cv$params={r:'9f30248f63e73a0d',t:'MTc3NzMxNzE4OS4wMDAwMDA='};var a=document.createElement('script');a.nonce='';a.src='/cdn-cgi/challenge-platform/scripts/jsd/main.js';document.getElementsByTagName('head')[0].appendChild(a);";b.getElementsByTagName('head')[0].appendChild(d)}}if(document.body){var a=document.createElement('iframe');a.height=1;a.width=1;a.style.position='absolute';a.style.top=0;a.style.left=0;a.style.border='none';a.style.visibility='hidden';document.body.appendChild(a);if('loading'!==document.readyState)c();else if(window.addEventListener)document.addEventListener('DOMContentLoaded',c);else{var e=document.onreadystatechange||function(){};document.onreadystatechange=function(b){e(b);'loading'!==document.readyState&&(document.onreadystatechange=e,c())}}}})();</script></body>
</html>
