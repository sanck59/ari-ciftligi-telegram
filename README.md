[index.html](https://github.com/user-attachments/files/32714734/index.html)
# ari-ciftligi-telegram<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
<title>Arı Çiftliği</title>
<script src="https://telegram.org/js/telegram-web-app.js"></script>
<style>
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  html, body {
    margin: 0; padding: 0; height: 100%;
    background: var(--tg-theme-bg-color, #eaf6d8);
    color: var(--tg-theme-text-color, #2f4f1f);
    font-family: -apple-system, Roboto, Helvetica, Arial, sans-serif;
    overscroll-behavior: none;
  }
  #app { display: flex; flex-direction: column; height: 100%; }

  h1 {
    text-align: center;
    font-size: 18px;
    margin: 10px 0;
    color: #4a7c2c;
  }

  .seed-bar {
    display: flex;
    justify-content: center;
    gap: 8px;
    padding: 4px 8px 10px;
  }
  .seed-btn {
    display: flex; flex-direction: column; align-items: center;
    background: #fff; border: 2px solid transparent; border-radius: 12px;
    padding: 6px 10px; font-size: 12px; cursor: pointer;
  }
  .seed-btn.active { border-color: #f5a623; background: #fff3d6; }
  .seed-btn .icon { font-size: 20px; }

  .grid {
    flex: 1;
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 8px;
    padding: 8px 12px 24px;
    overflow-y: auto;
  }

  .plot {
    aspect-ratio: 1 / 1;
    border-radius: 14px;
    display: flex; flex-direction: column; align-items: center; justify-content: center;
    cursor: pointer;
    user-select: none;
    transition: transform 0.1s ease;
  }
  .plot:active { transform: scale(0.95); }

  .plot.locked { background: #c9c9c9; }
  .plot.locked .lock { font-size: 22px; }

  .plot.empty { background: #8b5a2b; border: 2px solid #6e451f; }
  .plot.empty .plus { font-size: 26px; color: #fff; font-weight: bold; }

  .plot.growing { background: #a3d977; }
  .plot.ready { background: #ffe066; border: 2px solid #f5a623; }

  .plot .icon { font-size: 24px; }
  .plot .timer { font-size: 10px; margin-top: 2px; color: #2f4f1f; }
  .plot .ready-text { font-size: 10px; margin-top: 2px; font-weight: bold; color: #a05a00; }

  #toast {
    position: fixed; bottom: 16px; left: 50%; transform: translateX(-50%);
    background: rgba(0,0,0,0.8); color: #fff; padding: 10px 18px;
    border-radius: 20px; font-size: 13px; opacity: 0; pointer-events: none;
    transition: opacity 0.25s ease;
  }
  #toast.show { opacity: 1; }
</style>
</head>
<body>
<div id="app">
  <h1>Çiftlik — Arazi</h1>
  <div class="seed-bar" id="seedBar"></div>
  <div class="grid" id="grid"></div>
</div>
<div id="toast"></div>

<script>
// ============================================================
// ARI ÇİFTLİĞİ — TELEGRAM MINI APP (web versiyonu)
// Mantık FarmScreen.js (React Native) ile birebir aynı,
// sadece görsel katman HTML/CSS/vanilla JS'e taşındı.
// ============================================================

const tg = window.Telegram ? window.Telegram.WebApp : null;
if (tg) {
  tg.ready();
  tg.expand();
}

const TOTAL_PLOTS = 20;

const SEED_TYPES = {
  misir:    { id: 'misir',    name: 'Mısır',     icon: '🌽', growTimeSec: 60 * 30, yieldAmount: 3 },
  kahve:    { id: 'kahve',    name: 'Kahve',     icon: '☕', growTimeSec: 60 * 45, yieldAmount: 2 },
  aycicegi: { id: 'aycicegi', name: 'Ayçiçeği',  icon: '🌻', growTimeSec: 60 * 20, yieldAmount: 4 },
};

const STATE = { LOCKED: 'locked', EMPTY: 'empty', GROWING: 'growing', READY: 'ready' };

function createInitialPlots() {
  return Array.from({ length: TOTAL_PLOTS }, (_, i) => ({
    id: i,
    state: i < 8 ? STATE.EMPTY : STATE.LOCKED,
    seedId: null,
    plantedAt: null,
    unlockCost: 5000 + i * 2500,
  }));
}

// TODO: Bu state'i ileride sunucu/veritabanına (veya Telegram CloudStorage'a) bağlayacağız
let plots = loadState() || createInitialPlots();
let selectedSeed = 'misir';

function loadState() {
  try {
    const raw = localStorage.getItem('farm_plots');
    return raw ? JSON.parse(raw) : null;
  } catch (e) { return null; }
}
function saveState() {
  try { localStorage.setItem('farm_plots', JSON.stringify(plots)); } catch (e) {}
}

function formatTime(totalSeconds) {
  const m = Math.floor(totalSeconds / 60);
  const s = Math.floor(totalSeconds % 60);
  return `${m}:${s.toString().padStart(2, '0')}`;
}

function showToast(msg) {
  const el = document.getElementById('toast');
  el.textContent = msg;
  el.classList.add('show');
  setTimeout(() => el.classList.remove('show'), 2000);
}

function renderSeedBar() {
  const bar = document.getElementById('seedBar');
  bar.innerHTML = '';
  Object.values(SEED_TYPES).forEach(seed => {
    const btn = document.createElement('div');
    btn.className = 'seed-btn' + (selectedSeed === seed.id ? ' active' : '');
    btn.innerHTML = `<span class="icon">${seed.icon}</span><span>${seed.name}</span>`;
    btn.onclick = () => { selectedSeed = seed.id; renderSeedBar(); };
    bar.appendChild(btn);
  });
}

function renderGrid() {
  const grid = document.getElementById('grid');
  grid.innerHTML = '';
  const now = Date.now();

  plots.forEach(plot => {
    const cell = document.createElement('div');
    cell.className = 'plot ' + plot.state;

    if (plot.state === STATE.LOCKED) {
      cell.innerHTML = `<span class="lock">🔒</span>`;
    } else if (plot.state === STATE.EMPTY) {
      cell.innerHTML = `<span class="plus">+</span>`;
    } else if (plot.state === STATE.GROWING) {
      const seed = SEED_TYPES[plot.seedId];
      const remaining = Math.max(0, seed.growTimeSec - (now - plot.plantedAt) / 1000);
      cell.innerHTML = `<span class="icon">${seed.icon}</span><span class="timer">${formatTime(remaining)}</span>`;
    } else if (plot.state === STATE.READY) {
      const seed = SEED_TYPES[plot.seedId];
      cell.innerHTML = `<span class="icon">${seed.icon}</span><span class="ready-text">Hazır!</span>`;
    }

    cell.onclick = () => handlePlotClick(plot.id);
    grid.appendChild(cell);
  });
}

function handlePlotClick(plotId) {
  const plot = plots.find(p => p.id === plotId);

  if (plot.state === STATE.LOCKED) {
    if (confirm(`Bu parseli açmak için ${plot.unlockCost} para gerekiyor. Açmak ister misin?`)) {
      unlockPlot(plotId);
    }
    return;
  }
  if (plot.state === STATE.EMPTY) {
    plantSeed(plotId, selectedSeed);
    return;
  }
  if (plot.state === STATE.READY) {
    harvestPlot(plotId);
    return;
  }
  if (plot.state === STATE.GROWING) {
    const seed = SEED_TYPES[plot.seedId];
    const remaining = Math.max(0, seed.growTimeSec - (Date.now() - plot.plantedAt) / 1000);
    showToast(`Büyüyor — kalan süre: ${formatTime(remaining)}`);
  }
}

// TODO: Ekonomi modülü bağlandığında gerçek para kontrolü yapılacak
function unlockPlot(plotId) {
  const plot = plots.find(p => p.id === plotId);
  plot.state = STATE.EMPTY;
  saveState();
  renderGrid();
}

// TODO: Envanter modülü bağlandığında tohum stoğundan düşülecek
function plantSeed(plotId, seedId) {
  const plot = plots.find(p => p.id === plotId);
  plot.state = STATE.GROWING;
  plot.seedId = seedId;
  plot.plantedAt = Date.now();
  saveState();
  renderGrid();
}

// TODO: Envanter modülü bağlandığında hasat ürünü depoya eklenecek
function harvestPlot(plotId) {
  const plot = plots.find(p => p.id === plotId);
  const seed = SEED_TYPES[plot.seedId];

  plot.state = STATE.EMPTY;
  plot.seedId = null;
  plot.plantedAt = null;
  saveState();
  renderGrid();

  showToast(`${seed.icon} ${seed.yieldAmount} adet ${seed.name} kazandın!`);
  if (tg && tg.HapticFeedback) tg.HapticFeedback.notificationOccurred('success');
}

// Her saniye büyüme durumunu kontrol et ve ekranı güncelle
setInterval(() => {
  let changed = false;
  const now = Date.now();
  plots.forEach(plot => {
    if (plot.state === STATE.GROWING) {
      const seed = SEED_TYPES[plot.seedId];
      if ((now - plot.plantedAt) / 1000 >= seed.growTimeSec) {
        plot.state = STATE.READY;
        changed = true;
      }
    }
  });
  if (changed) saveState();
  renderGrid();
}, 1000);

renderSeedBar();
renderGrid();
</script>
</body>
</html>
