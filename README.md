const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const overlay = document.getElementById('overlay');
const toast = document.getElementById('toast');
const homeScoreEl = document.getElementById('homeScore');
const awayScoreEl = document.getElementById('awayScore');
const shotsEl = document.getElementById('shots');
const savesEl = document.getElementById('saves');
const hitsEl = document.getElementById('hits');
const energyEl = document.getElementById('energyFill');
const modeLabelEl = document.getElementById('modeLabel');
const clockEl = document.getElementById('clock');

const rink = { x: 70, y: 40, w: 1060, h: 490, r: 100 };
const keys = new Set();

let state = 'menu';
let mode = '1p';
let lastTime = 0;
let timer = 90;
let homeScore = 0;
let awayScore = 0;
let shots = 0;
let saves = 0;
let hits = 0;
let energy = 100;
let toastTimer = 0;
let soundOn = true;
let audioCtx = null;

const crowd = [];
for (let i = 0; i < 80; i++) {
  crowd.push({
    x: Math.random() * canvas.width,
    y: Math.random() * canvas.height,
    r: 1.4 + Math.random() * 2,
    color: ['#d9f8ff', '#7edff4', '#ffdc84', '#a6bfd8'][i % 4]
  });
}

const puck = {
  x: 600,
  y: 285,
  vx: 0,
  vy: 0,
  radius: 9,
  owner: 'player',
  trail: []
};

const player = {
  x: 430,
  y: 285,
  vx: 0,
  vy: 0,
  radius: 20,
  angle: 0,
  color: '#17c2ee',
  name: 'YOU'
};

const winger = {
  x: 330,
  y: 215,
  vx: 0,
  vy: 0,
  radius: 17,
  angle: 0,
  color: '#9feef9',
  name: '19'
};

const rival1 = {
  x: 770,
  y: 285,
  vx: 0,
  vy: 0,
  radius: 20,
  angle: Math.PI,
  color: '#fb7b75',
  name: '7'
};

const rival2 = {
  x: 690,
  y: 220,
  vx: 0,
  vy: 0,
  radius: 17,
  angle: Math.PI,
  color: '#ffc0ac',
  name: '24'
};

const goalie = {
  x: 1088,
  y: 285,
  radius: 24,
  color: '#c44663'
};

const particles = [];

function clamp(value, min, max) {
  return Math.max(min, Math.min(max, value));
}

function beep(freq = 440, duration = 0.08, type = 'sine', volume = 0.03) {
  if (!soundOn) return;
  try {
    const AudioCtor = window.AudioContext || window.webkitAudioContext;
    if (!audioCtx) audioCtx = new AudioCtor();
    if (audioCtx.state === 'suspended') audioCtx.resume();

    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();

    osc.type = type;
    osc.frequency.value = freq;

    gain.gain.setValueAtTime(volume, audioCtx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duration);

    osc.connect(gain);
    gain.connect(audioCtx.destination);

    osc.start();
    osc.stop(audioCtx.currentTime + duration);
  } catch (err) {
    // browser audio is optional
  }
}

function showToast(text, duration = 1.4) {
  toast.textContent = text;
  toast.classList.add('show');
  toastTimer = duration;
}

function setClockText() {
  const mins = Math.floor(timer / 60);
  const secs = Math.floor(timer % 60);
  clockEl.textContent = `${mins}:${String(secs).padStart(2, '0')}`;
  clockEl.classList.toggle('urgent', timer <= 15 && timer > 0);
}

function updateHud() {
  homeScoreEl.textContent = homeScore;
  awayScoreEl.textContent = awayScore;
  shotsEl.textContent = shots;
  savesEl.textContent = saves;
  hitsEl.textContent = hits;
  energyEl.style.width = `${energy}%`;
  modeLabelEl.textContent = mode.toUpperCase();
  setClockText();
}

function resetFaceoff() {
  player.x = 430;
  player.y = 285;
  player.vx = 0;
  player.vy = 0;

  winger.x = 330;
  winger.y = 215;
  winger.vx = 0;
  winger.vy = 0;

  rival1.x = 770;
  rival1.y = 285;
  rival1.vx = 0;
  rival1.vy = 0;

  rival2.x = 690;
  rival2.y = 220;
  rival2.vx = 0;
  rival2.vy = 0;

  goalie.y = 285;

  puck.x = 600;
  puck.y = 285;
  puck.vx = 0;
  puck.vy = 0;
  puck.owner = 'player';
  puck.trail = [];
}

function startGame(nextMode = mode) {
  mode = nextMode;
  state = 'playing';
  timer = 90;
  homeScore = 0;
  awayScore = 0;
  shots = 0;
  saves = 0;
  hits = 0;
  energy = 100;
  resetFaceoff();
  overlay.classList.add('hidden');
  updateHud();
  showToast('Game on', 1.2);
  beep(620, 0.08, 'triangle', 0.04);
}

function pauseGame() {
  if (state !== 'playing') return;
  state = 'paused';

  overlay.classList.remove('hidden');
  overlay.innerHTML = `
    <div class="menu-panel">
      <div class="eyebrow">Take a breather</div>
      <div class="menu-crest">❄</div>
      <h1>Time<br><em>frozen</em></h1>
      <p>Reset your rhythm and get back out there. One big shift is all it takes.</p>
      <div class="menu-actions">
        <button class="primary-btn" id="resumeBtn">Resume ▶</button>
      </div>
    </div>
  `;

  document.getElementById('resumeBtn').addEventListener('click', () => {
    state = 'playing';
    overlay.classList.add('hidden');
    lastTime = performance.now();
    overlay.innerHTML = `
      <div class="menu-panel" id="menuPanel">
        <div class="eyebrow">The crowd is waiting</div>
        <div class="menu-crest">🏒</div>
        <h1>Five minutes<br><em>to glory</em></h1>
        <p>Arcade hockey energy with a cinematic rush. Pick your mode, hit the ice, and chase the buzzer-beater.</p>
        <div class="menu-actions">
          <button class="primary-btn" data-mode="1p">Quick Start • 1P</button>
          <button class="secondary-btn" data-mode="2p">Duel Mode • 2P</button>
          <button class="ghost-btn" data-mode="practice">Practice</button>
        </div>
        <div class="menu-controls">
          <div><span class="key">WASD</span> or <span class="key">Arrows</span> move</div>
          <div><span class="key">SPACE</span> shoot</div>
          <div><span class="key">SHIFT</span> sprint</div>
          <div><span class="key">P</span> pass</div>
        </div>
      </div>
    `;
    bindModeButtons();
  });
}

function finishGame() {
  state = 'finished';
  overlay.classList.remove('hidden');

  const won = homeScore > awayScore;
  const isDraw = homeScore === awayScore;

  overlay.innerHTML = `
    <div class="menu-panel">
      <div class="eyebrow">${isDraw ? 'Overtime' : 'Final horn'}</div>
      <div class="menu-crest">${isDraw ? '⚡' : won ? '🏆' : '🏒'}</div>
      <h1>${isDraw ? 'Next goal' : 'What a'}<br><em>${isDraw ? 'wins' : won ? 'finish' : 'battle'}</em></h1>
      <div class="result-score" style="display:flex;justify-content:center;align-items:center;gap:18px;margin-top:18px;font-size:34px;font-weight:900;">
        <span style="color:#67e9ff;">${homeScore}</span>
        <span style="font-size:11px;letter-spacing:2px;color:#8ea7bf;text-transform:uppercase;">Final</span>
        <span style="color:#ff9382;">${awayScore}</span>
      </div>
      <p>${isDraw ? 'Sudden-death pressure. Every shift matters.' : 'The crowd will be repeating that one all season.'}</p>
      <div class="menu-actions">
        <button class="primary-btn" id="againBtn">${isDraw ? 'Start overtime' : 'Play again'} ▶</button>
      </div>
    </div>
  `;

  document.getElementById('againBtn').addEventListener('click', () => startGame(mode));
}

function bindModeButtons() {
  document.querySelectorAll('[data-mode]').forEach((btn) => {
    btn.addEventListener('click', () => {
      const choice = btn.dataset.mode;
      startGame(choice);
    });
  });
}

document.querySelectorAll('[data-mode]').forEach((btn) => {
  btn.addEventListener('click', () => {
    startGame(btn.dataset.mode);
  });
});

document.getElementById('soundToggle').addEventListener('click', () => {
  soundOn = !soundOn;
  document.getElementById('soundToggle').textContent = soundOn ? '♫' : '🔇';
  if (soundOn) beep(440, 0.08, 'triangle', 0.03);
});

document.getElementById('pauseBtn').addEventListener('click', pauseGame);

document.addEventListener('keydown', (event) => {
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', ' '].includes(event.key)) {
    event.preventDefault();
  }

  if (event.key === 'Escape') {
    pauseGame();
    return;
  }

  if (event.key === 'p' || event.key === 'P') {
    passPuck();
    return;
  }

  if (event.key === ' ') {
    shoot();
    return;
  }

  if (!event.repeat) keys.add(event.key);
});

document.addEventListener('keyup', (event) => {
  const key = event.key === ' ' ? 'Space' : event.key;
  keys.delete(key);
});

document.querySelectorAll('[data-key]').forEach((button) => {
  const key = button.dataset.key;
  const release = () => keys.delete(key);
  button.addEventListener('pointerdown', (event) => {
    event.preventDefault();
    keys.add(key);
  });
  button.addEventListener('pointerup', release);
  button.addEventListener('pointerleave', release);
  button.addEventListener('pointercancel', release);
});

document.querySelectorAll('[data-hold]').forEach((button) => {
  const key = button.dataset.hold;
  const release = () => keys.delete(key);
  button.addEventListener('pointerdown', (event) => {
    event.preventDefault();
    keys.add(key);
  });
  button.addEventListener('pointerup', release);
  button.addEventListener('pointerleave', release);
  button.addEventListener('pointercancel', release);
});

document.querySelectorAll('[data-shoot]').forEach((button) => {
  button.addEventListener('pointerdown', (event) => {
    event.preventDefault();
    shoot();
  });
});

function shoot() {
  if (state !== 'playing') return;

  if (puck.owner === 'player') {
    const dx = Math.cos(player.angle);
    const dy = Math.sin(player.angle);
    puck.owner = null;
    puck.x = player.x + dx * 28;
    puck.y = player.y + dy * 28;
    puck.vx = dx * 680 + player.vx * 0.4;
    puck.vy = dy * 680 + player.vy * 0.4;
    shots += 1;
    burst(puck.x, puck.y, '#90f2ff', 10);
    beep(180, 0.12, 'triangle', 0.035);
    updateHud();
  }
}

function passPuck() {
  if (state !== 'playing' || puck.owner !== 'player') return;

  const dx = winger.x - player.x;
  const dy = winger.y - player.y;
  const dist = Math.hypot(dx, dy) || 1;

  puck.owner = null;
  puck.x = player.x + (dx / dist) * 20;
  puck.y = player.y + (dy / dist) * 20;
  puck.vx = (dx / dist) * 520;
  puck.vy = (dy / dist) * 520;

  beep(520, 0.12, 'sine', 0.03);
}

function burst(x, y, color, count = 18) {
  for (let i = 0; i < count; i++) {
    const angle = Math.random() * Math.PI * 2;
    const speed = 80 + Math.random() * 260;
    particles.push({
      x,
      y,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      life: 0.6 + Math.random() * 0.8,
      maxLife: 0.8 + Math.random() * 0.8,
      radius: 2 + Math.random() * 4,
      color
    });
  }
}

function updatePlayer(dt) {
  let ax = 0;
  let ay = 0;

  if (keys.has('ArrowRight') || keys.has('d') || keys.has('D')) ax += 1;
  if (keys.has('ArrowLeft') || keys.has('a') || keys.has('A')) ax -= 1;
  if (keys.has('ArrowDown') || keys.has('s') || keys.has('S')) ay += 1;
  if (keys.has('ArrowUp') || keys.has('w') || keys.has('W')) ay -= 1;

  const running = ax || ay;
  if (running) {
    const len = Math.hypot(ax, ay) || 1;
    ax /= len;
    ay /= len;
    player.angle = Math.atan2(ay, ax);
  }

  const sprinting = (keys.has('Shift') || keys.has('ShiftLeft')) && energy > 0 && running;
  const accel = sprinting ? 1250 : 820;
  const maxSpeed = sprinting ? 370 : 250;

  player.vx += ax * accel * dt;
  player.vy += ay * accel * dt;

  player.vx *= running ? 0.92 : 0.86;
  player.vy *= running ? 0.92 : 0.86;

  const speed = Math.hypot(player.vx, player.vy);
  if (speed > maxSpeed) {
    player.vx = (player.vx / speed) * maxSpeed;
    player.vy = (player.vy / speed) * maxSpeed;
  }

  if (sprinting) {
    energy = Math.max(0, energy - 30 * dt);
  } else {
    energy = Math.min(100, energy + 18 * dt);
  }

  player.x += player.vx * dt;
  player.y += player.vy * dt;
  player.x = clamp(player.x, rink.x + 25, rink.x + rink.w - 25);
  player.y = clamp(player.y, rink.y + 26, rink.y + rink.h - 26);
}

function updateAI(dt) {
  // Rival 1 pressures the puck
  let targetX = puck.x;
  let targetY = puck.y;

  if (puck.owner === 'player') {
    targetX = player.x - 60;
    targetY = player.y;
  }

  let dx = targetX - rival1.x;
  let dy = targetY - rival1.y;
  let dist = Math.hypot(dx, dy) || 1;

  rival1.vx += (dx / dist) * 170 * dt * 4;
  rival1.vy += (dy / dist) * 170 * dt * 4;
  rival1.vx *= 0.955;
  rival1.vy *= 0.955;

  let speed = Math.hypot(rival1.vx, rival1.vy);
  if (speed > 220) {
    rival1.vx = (rival1.vx / speed) * 220;
    rival1.vy = (rival1.vy / speed) * 220;
  }

  rival1.x += rival1.vx * dt;
  rival1.y += rival1.vy * dt;
  rival1.x = clamp(rival1.x, rink.x + 120, rink.x + rink.w - 120);
  rival1.y = clamp(rival1.y, rink.y + 60, rink.y + rink.h - 60);

  // Rival 2 supports the chase
  let supportX = 620;
  let supportY = 240;

  if (puck.owner === 'player') {
    supportX = Math.min(740, player.x - 20);
    supportY = player.y;
  } else if (puck.owner === null) {
    supportX = puck.x - 40;
    supportY = puck.y;
  }

  dx = supportX - rival2.x;
  dy = supportY - rival2.y;
  dist = Math.hypot(dx, dy) || 1;

  rival2.vx += (dx / dist) * 160 * dt * 3.4;
  rival2.vy += (dy / dist) * 160 * dt * 3.4;
  rival2.vx *= 0.96;
  rival2.vy *= 0.96;

  speed = Math.hypot(rival2.vx, rival2.vy);
  if (speed > 200) {
    rival2.vx = (rival2.vx / speed) * 200;
    rival2.vy = (rival2.vy / speed) * 200;
  }

  rival2.x += rival2.vx * dt;
  rival2.y += rival2.vy * dt;
  rival2.x = clamp(rival2.x, rink.x + 120, rink.x + rink.w - 120);
  rival2.y = clamp(rival2.y, rink.y + 60, rink.y + rink.h - 60);

  // Winger tries to open a lane
  let wingX = player.x - 120;
  let wingY = clamp(player.y + 70, rink.y + 100, rink.y + rink.h - 100);

  if (puck.owner === 'player') {
    wingX = player.x + 120;
    wingY = player.y + 10;
  }

  dx = wingX - winger.x;
  dy = wingY - winger.y;
  dist = Math.hypot(dx, dy) || 1;

  winger.vx += (dx / dist) * 210 * dt * 2.5;
  winger.vy += (dy / dist) * 210 * dt * 2.5;
  winger.vx *= 0.97;
  winger.vy *= 0.97;
  speed = Math.hypot(winger.vx, winger.vy);
  if (speed > 210) {
    winger.vx = (winger.vx / speed) * 210;
    winger.vy = (winger.vy / speed) * 210;
  }

  winger.x += winger.vx * dt;
  winger.y += winger.vy * dt;
  winger.x = clamp(winger.x, rink.x + 100, rink.x + rink.w - 100);
  winger.y = clamp(winger.y, rink.y + 60, rink.y + rink.h - 60);

  goalie.y += (clamp(puck.y, rink.y + 90, rink.y + rink.h - 90) - goalie.y) * Math.min(1, dt * 4.8);
}

function applyContact(a, b) {
  const dx = b.x - a.x;
  const dy = b.y - a.y;
  const dist = Math.hypot(dx, dy) || 1;
  const minDist = a.radius + b.radius + 2;

  if (dist < minDist) {
    const nx = dx / dist;
    const ny = dy / dist;
    const overlap = minDist - dist;

    a.x -= nx * overlap * 0.5;
    a.y -= ny * overlap * 0.5;
    b.x += nx * overlap * 0.5;
    b.y += ny * overlap * 0.5;

    const relVel = (b.vx - a.vx) * nx + (b.vy - a.vy) * ny;
    if (relVel < 0) {
      a.vx += nx * relVel * 0.38;
      a.vy += ny * relVel * 0.38;
      b.vx -= nx * relVel * 0.38;
      b.vy -= ny * relVel * 0.38;
    }

    if (a === player && b === rival1) {
      hits += 1;
      burst(a.x, a.y, '#ffe7a1', 8);
      beep(90, 0.08, 'square', 0.02);
      updateHud();
    }
  }
}

function scoreGoal(team) {
  if (team === 'home') homeScore += 1;
  else awayScore += 1;

  burst(puck.x, puck.y, team === 'home' ? '#8feaff' : '#ff9d8a', 34);
  beep(team === 'home' ? 760 : 240, 0.2, 'sawtooth', 0.05);
  showToast(team === 'home' ? 'Goal! Northstars!' : 'Goal! Frostwolves!', 1.7);
  updateHud();

  state = 'goal';
  resetFaceoff();

  setTimeout(() => {
    if (state === 'goal') state = 'playing';
  }, 1200);
}

function updatePuck(dt) {
  if (puck.owner === 'player') {
    const dx = Math.cos(player.angle);
    const dy = Math.sin(player.angle);
    puck.x = player.x + dx * 28;
    puck.y = player.y + dy * 28;
    puck.vx = player.vx * 0.4;
    puck.vy = player.vy * 0.4;
    return;
  }

  if (puck.owner === 'away') {
    const dx = Math.cos(rival1.angle);
    const dy = Math.sin(rival1.angle);
    puck.x = rival1.x + dx * 26;
    puck.y = rival1.y + dy * 26;
    puck.vx = rival1.vx * 0.35;
    puck.vy = rival1.vy * 0.35;
    return;
  }

  puck.x += puck.vx * dt;
  puck.y += puck.vy * dt;

  puck.vx *= 0.992;
  puck.vy *= 0.992;

  puck.trail.push({ x: puck.x, y: puck.y, alpha: 1 });
  if (puck.trail.length > 14) puck.trail.shift();
  puck.trail.forEach((p) => (p.alpha -= dt * 1.5));
  puck.trail = puck.trail.filter((p) => p.alpha > 0);

  if (puck.y < rink.y + 10 || puck.y > rink.y + rink.h - 10) {
    puck.vy *= -0.72;
    puck.y = clamp(puck.y, rink.y + 12, rink.y + rink.h - 12);
    beep(240, 0.04, 'sine', 0.015);
  }

  if (puck.x < rink.x + 10 || puck.x > rink.x + rink.w - 10) {
    if (puck.y > 230 && puck.y < 340) {
      if (puck.x < rink.x + 10) scoreGoal('home');
      else scoreGoal('away');
      return;
    }
    puck.vx *= -0.72;
    puck.x = clamp(puck.x, rink.x + 12, rink.x + rink.w - 12);
  }

  if (Math.hypot(puck.vx, puck.vy) < 30) {
    const candidates = [player, winger, rival1, rival2];
    candidates.sort((a, b) => {
      const da = Math.hypot(a.x - puck.x, a.y - puck.y);
      const db = Math.hypot(b.x - puck.x, b.y - puck.y);
      return da - db;
    });

    const nearest = candidates[0];
    if (Math.hypot(nearest.x - puck.x, nearest.y - puck.y) < nearest.radius + puck.radius + 10) {
      puck.owner = nearest === player || nearest === winger ? 'player' : 'away';
    }
  }

  if (puck.x > 1040 && puck.y > goalie.y - 48 && puck.y < goalie.y + 48) {
    if (Math.abs(puck.x - goalie.x) < 30) {
      puck.vx *= -0.7;
      puck.vy *= 0.7;
      puck.x = 1025;
      saves += 1;
      updateHud();
      beep(520, 0.07, 'triangle', 0.025);
    }
  }
}

function update(dt) {
  if (state === 'playing') {
    timer -= dt;
    if (timer <= 0) {
      timer = 0;
      state = 'finished';
      finishGame();
      return;
    }

    updatePlayer(dt);
    updateAI(dt);
    applyContact(player, rival1);
    applyContact(player, rival2);
    applyContact(winger, rival1);
    updatePuck(dt);

    if (toastTimer > 0) {
      toastTimer -= dt;
      if (toastTimer <= 0) toast.classList.remove('show');
    }

    updateHud();
  }

  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.vx *= 0.985;
    p.vy *= 0.985;
    p.life -= dt;
    if (p.life <= 0) particles.splice(i, 1);
  }
}

function drawBackground() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const bg = ctx.createLinearGradient(0, 0, 0, canvas.height);
  bg.addColorStop(0, '#dffaff');
  bg.addColorStop(0.58, '#d3eef7');
  bg.addColorStop(1, '#d7f1fb');
  ctx.fillStyle = bg;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  for (const c of crowd) {
    ctx.fillStyle = c.color;
    ctx.beginPath();
    ctx.arc(c.x, c.y, c.r, 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.save();
  ctx.beginPath();
  ctx.roundRect(rink.x, rink.y, rink.w, rink.h, rink.r);
  ctx.clip();
  ctx.fillStyle = '#ddf8ff';
  ctx.fillRect(rink.x, rink.y, rink.w, rink.h);

  for (let i = 0; i < 18; i++) {
    ctx.strokeStyle = 'rgba(255,255,255,0.55)';
    ctx.lineWidth = 1.3;
    ctx.beginPath();
    ctx.moveTo(rink.x + 40 + i * 58, rink.y + 15);
    ctx.lineTo(rink.x + 90 + i * 58, rink.y + rink.h - 15);
    ctx.stroke();
  }
  ctx.restore();

  ctx.strokeStyle = '#8bd0eb';
  ctx.lineWidth = 6;
  ctx.beginPath();
  ctx.roundRect(rink.x, rink.y, rink.w, rink.h, rink.r);
  ctx.stroke();

  ctx.strokeStyle = '#d96f7f';
  ctx.lineWidth = 4;
  ctx.beginPath();
  ctx.moveTo(600, rink.y);
  ctx.lineTo(600, rink.y + rink.h);
  ctx.stroke();

  ctx.beginPath();
  ctx.arc(600, 285, 42, 0, Math.PI * 2);
  ctx.stroke();

  ctx.strokeStyle = '#50abd8';
  ctx.beginPath();
  ctx.moveTo(340, rink.y);
  ctx.lineTo(340, rink.y + rink.h);
  ctx.stroke();

  ctx.beginPath();
  ctx.moveTo(860, rink.y);
  ctx.lineTo(860, rink.y + rink.h);
  ctx.stroke();

  ctx.fillStyle = '#ebfbff';
  ctx.beginPath();
  ctx.arc(600, 285, 5, 0, Math.PI * 2);
  ctx.fill();

  for (const x of [270, 930]) {
    for (const y of [190, 380]) {
      ctx.beginPath();
      ctx.arc(x, y, 46, 0, Math.PI * 2);
      ctx.strokeStyle = '#f2bac5';
      ctx.lineWidth = 2;
      ctx.stroke();
    }
  }

  for (const gx of [35, 1130]) {
    ctx.strokeStyle = '#d55c69';
    ctx.lineWidth = 3;
    ctx.strokeRect(gx, 250, 26, 76);
  }
}

function drawSkater(s) {
  ctx.save();
  ctx.translate(s.x, s.y);
  ctx.rotate(s.angle);

  ctx.fillStyle = 'rgba(0,0,0,0.16)';
  ctx.beginPath();
  ctx.ellipse(4, 12, s.radius * 1.15, s.radius * 0.8, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.strokeStyle = '#5f3a25';
  ctx.lineWidth = 4;
  ctx.lineCap = 'round';
  ctx.beginPath();
  ctx.moveTo(s.radius * 0.2, 2);
  ctx.lineTo(s.radius + 18, 12);
  ctx.stroke();

  ctx.fillStyle = '#162b3d';
  ctx.fillRect(-s.radius * 0.45, s.radius * 0.7, s.radius * 0.62, 5);
  ctx.fillRect(s.radius * 0.1, s.radius * 0.7, s.radius * 0.62, 5);

  ctx.fillStyle = '#0c1d2f';
  ctx.beginPath();
  ctx.ellipse(0, 0, s.radius * 0.9, s.radius * 0.75, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = s.color;
  ctx.beginPath();
  ctx.ellipse(0, -2, s.radius * 0.78, s.radius * 0.65, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#eefbff';
  ctx.fillRect(-s.radius * 0.7, 6, s.radius * 1.4, 4);

  ctx.fillStyle = s.team === 'home' ? '#0b6d95' : '#8c3747';
  ctx.beginPath();
  ctx.arc(0, -s.radius * 0.9, s.radius * 0.42, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#f9feff';
  ctx.font = `900 ${s.radius * 0.6}px Arial`;
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(s.name, 0, -1);

  ctx.restore();
}

function drawGoalie() {
  ctx.save();
  ctx.translate(goalie.x, goalie.y);

  ctx.fillStyle = 'rgba(0,0,0,0.14)';
  ctx.beginPath();
  ctx.ellipse(4, 12, 30, 22, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#edf8ff';
  ctx.beginPath();
  ctx.arc(0, 0, 26, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#a83d5a';
  ctx.beginPath();
  ctx.arc(0, 0, 20, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#1f2b3b';
  ctx.fillRect(-16, -18, 9, 38);
  ctx.fillRect(7, -18, 9, 38);

  ctx.restore();
}

function drawPuck() {
  for (const p of puck.trail) {
    ctx.globalAlpha = Math.max(0, p.alpha) * 0.65;
    ctx.fillStyle = '#48c6fb';
    ctx.beginPath();
    ctx.arc(p.x, p.y, 5, 0, Math.PI * 2);
    ctx.fill();
  }
  ctx.globalAlpha = 1;

  ctx.fillStyle = '#0d1c2c';
  ctx.beginPath();
  ctx.ellipse(puck.x, puck.y + 4, 11, 6, 0, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = '#0f1b27';
  ctx.beginPath();
  ctx.arc(puck.x, puck.y, 9, 0, Math.PI * 2);
  ctx.fill();
}

function drawParticles() {
  for (const p of particles) {
    ctx.globalAlpha = Math.max(0, p.life / p.maxLife);
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
    ctx.fill();
  }
  ctx.globalAlpha = 1;
}

function render() {
  drawBackground();
  drawPuck();
  drawSkater(winger);
  drawSkater(rival2);
  drawSkater(rival1);
  drawGoalie();
  drawSkater(player);
  drawParticles();
}

function animate(timestamp) {
  const dt = Math.min(0.032, (timestamp - lastTime) / 1000 || 0.016);
  lastTime = timestamp;

  update(dt);
  render();
  requestAnimationFrame(animate);
}

bindModeButtons();
updateHud();
render();
requestAnimationFrame(animate);

window.addEventListener('keydown', (event) => {
  if (event.key === 'Escape') {
    pauseGame();
  }
});
