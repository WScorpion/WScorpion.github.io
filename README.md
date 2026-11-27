# WScorpion.github.io
<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Wickelzähler</title>
<style>
  * { box-sizing: border-box; }
  body { margin: 0; font-family: system-ui, sans-serif; background: #111; color: #eee; display: flex; flex-direction: column; align-items: center; padding: 12px; min-height: 100vh; }
  h1 { font-size: 1.1rem; margin: 4px 0 10px; font-weight: 600; }
  .video-wrap { position: relative; width: 100%; max-width: 480px; background: #000; border-radius: 8px; overflow: hidden; aspect-ratio: 4 / 3; }
  video { width: 100%; height: 100%; object-fit: cover; display: block; }
  #roi { position: absolute; border: 2px solid #3af; border-radius: 4px; pointer-events: none; transition: border-color .05s; }
  #roi.hit { border-color: #6f6; background: rgba(100,255,100,.25); }
  .count { font-size: 5rem; font-weight: 700; font-variant-numeric: tabular-nums; margin: 12px 0 0; line-height: 1; }
  .target { font-size: .9rem; color: #aaa; margin-bottom: 8px; }
  .status { min-height: 1.2em; font-size: .9rem; color: #fc6; margin-bottom: 8px; }
  .row { display: flex; gap: 8px; flex-wrap: wrap; justify-content: center; margin: 6px 0; }
  button { font-size: 1.4rem; padding: 10px 18px; border: 0; border-radius: 8px; background: #333; color: #eee; min-width: 64px; cursor: pointer; }
  button:active { background: #555; }
  button.primary { background: #2a6; color: #fff; font-size: 1rem; }
  button.danger { background: #833; font-size: 1rem; }
  input[type=number] { font-size: 1.2rem; width: 110px; padding: 8px; border-radius: 6px; border: 1px solid #444; background: #222; color: #eee; text-align: center; }
  .settings { width: 100%; max-width: 480px; margin-top: 14px; font-size: .9rem; color: #bbb; }
  .settings label { display: flex; justify-content: space-between; align-items: center; margin: 8px 0; gap: 8px; }
  .settings input[type=range] { flex: 1; }
  .hint { font-size: .8rem; color: #888; margin-top: 10px; line-height: 1.4; }
</style>
</head>
<body>
<h1>Wickelzähler</h1>

<div class="video-wrap">
  <video id="video" playsinline muted autoplay></video>
  <div id="roi"></div>
</div>

<div class="count" id="count">0</div>
<div class="target">Ziel: <span id="targetLabel">200</span> &middot; Ton ab <span id="beepLabel">197</span></div>
<div class="status" id="status">Kamera nicht gestartet</div>

<div class="row">
  <button id="startBtn" class="primary">Kamera starten</button>
</div>
<div class="row">
  <button id="minus">&minus;1</button>
  <button id="plus">+1</button>
</div>
<div class="row">
  <input type="number" id="setValue" min="0" placeholder="Stand">
  <button id="setBtn" class="primary">Stand setzen</button>
</div>
<div class="row">
  <button id="resetBtn" class="danger">Zurücksetzen</button>
</div>

<div class="settings">
  <label>Helligkeits-Schwelle <span id="thrVal">200</span>
    <input type="range" id="thr" min="120" max="255" value="200"></label>
  <label>Zielwert <input type="number" id="target" value="200" min="1" style="width:90px"></label>
  <label>Ton ab <input type="number" id="beepAt" value="197" min="1" style="width:90px"></label>
  <label>Bereich (Größe) <span id="sizeVal">30</span>%
    <input type="range" id="size" min="10" max="60" value="30"></label>
</div>

<p class="hint">Tipp: Den weißen Streifen mittig in den blauen Rahmen halten. Gezählt wird beim Eintreten des Streifens in den Rahmen. Der Stand wird im Browser gespeichert und bleibt nach dem Neuladen erhalten.</p>

<canvas id="analysis" width="80" height="60" style="display:none"></canvas>

<script>
(() => {
  const KEY = 'wickelzaehler.v1';
  const $ = id => document.getElementById(id);
  const video = $('video'), roiEl = $('roi'), canvas = $('analysis');
  const ctx = canvas.getContext('2d', { willReadFrequently: true });

  const state = { count: 0, target: 200, beepAt: 197 };
  let stripeOn = false, lastCountTime = 0, stream = null, audioCtx = null, running = false;

  // ---------- Persistenz ----------
  function load() {
    try {
      const s = JSON.parse(localStorage.getItem(KEY));
      if (s && Number.isFinite(s.count)) state.count = s.count;
      if (s && Number.isFinite(s.target)) state.target = s.target;
      if (s && Number.isFinite(s.beepAt)) state.beepAt = s.beepAt;
      if (s && Number.isFinite(s.thr)) $('thr').value = s.thr;
      if (s && Number.isFinite(s.size)) $('size').value = s.size;
    } catch (e) {}
  }
  function save() {
    try {
      localStorage.setItem(KEY, JSON.stringify({
        count: state.count, target: state.target, beepAt: state.beepAt,
        thr: +$('thr').value, size: +$('size').value
      }));
    } catch (e) {}
  }

  // ---------- Anzeige ----------
  function render() {
    $('count').textContent = state.count;
    $('targetLabel').textContent = state.target;
    $('beepLabel').textContent = state.beepAt;
    $('target').value = state.target;
    $('beepAt').value = state.beepAt;
    $('setValue').value = state.count;
    $('thrVal').textContent = $('thr').value;
    $('sizeVal').textContent = $('size').value;
    $('count').style.color = state.count >= state.target ? '#6f6' : '#eee';
  }
  function status(msg) { $('status').textContent = msg; }

  // ---------- Ton ----------
  function ensureAudio() {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if (audioCtx.state === 'suspended') audioCtx.resume();
  }
  function beep(freq = 880, dur = 0.25) {
    if (!audioCtx) return;
    const o = audioCtx.createOscillator(), g = audioCtx.createGain();
    o.frequency.value = freq;
    g.gain.setValueAtTime(0.3, audioCtx.currentTime);
    g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + dur);
    o.connect(g).connect(audioCtx.destination);
    o.start(); o.stop(audioCtx.currentTime + dur);
  }

  // ---------- Zählen ----------
  function changeCount(delta) {
    const old = state.count;
    state.count = Math.max(0, state.count + delta);
    if (state.count !== old) {
      if (delta > 0 && state.count >= state.beepAt) beep(state.count >= state.target ? 1320 : 880);
      if (state.count >= state.target && old < state.target) status('Ziel erreicht!');
      save(); render();
    }
  }

  // ---------- Kamera-Analyse ----------
  function getRoi() {
    const pct = +$('size').value / 100;
    const w = pct, h = pct;
    return { x: (1 - w) / 2, y: (1 - h) / 2, w, h };
  }
  function drawRoiBox() {
    const r = getRoi();
    roiEl.style.left = (r.x * 100) + '%';
    roiEl.style.top = (r.y * 100) + '%';
    roiEl.style.width = (r.w * 100) + '%';
    roiEl.style.height = (r.h * 100) + '%';
  }

  function analyse(now) {
    if (!running) return;
    if (video.readyState >= 2) {
      const vw = video.videoWidth, vh = video.videoHeight;
      const r = getRoi();
      const sx = Math.floor(vw * r.x), sy = Math.floor(vh * r.y);
      const sw = Math.floor(vw * r.w), sh = Math.floor(vh * r.h);
      ctx.drawImage(video, sx, sy, sw, sh, 0, 0, canvas.width, canvas.height);
      const data = ctx.getImageData(0, 0, canvas.width, canvas.height).data;
      const thr = +$('thr').value;
      let bright = 0, total = 0;
      for (let i = 0; i < data.length; i += 4) {
        const lum = 0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2];
        if (lum > thr) bright++;
        total++;
      }
      // Streifen gilt als erkannt, wenn mindestens 25 % des Ausschnitts hell ist
      const present = bright / total > 0.25;

      // Flanke erkennen: nur beim Eintreten zählen, Mindestabstand 120 ms
      if (present && !stripeOn && now - lastCountTime > 120) {
        lastCountTime = now;
        changeCount(+1);
        roiEl.classList.add('hit');
        setTimeout(() => roiEl.classList.remove('hit'), 150);
      }
      stripeOn = present;
    }
    requestAnimationFrame(analyse);
  }

  async function startCamera() {
    ensureAudio();
    if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
      status('Kamera wird von diesem Browser oder dieser Adresse nicht unterstützt (HTTPS nötig).');
      return;
    }
    try {
      stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: { ideal: 'environment' }, width: { ideal: 640 }, height: { ideal: 480 } },
        audio: false
      });
      video.srcObject = stream;
      await video.play();
      running = true;
      $('startBtn').textContent = 'Kamera läuft';
      $('startBtn').disabled = true;
      status('Zählen aktiv');
      drawRoiBox();
      requestAnimationFrame(analyse);
    } catch (e) {
      status('Kamerazugriff fehlgeschlagen: ' + e.message);
    }
  }

  // ---------- Bedienung ----------
  $('startBtn').addEventListener('click', startCamera);
  $('plus').addEventListener('click', () => { ensureAudio(); changeCount(+1); });
  $('minus').addEventListener('click', () => { ensureAudio(); changeCount(-1); });
  $('setBtn').addEventListener('click', () => {
    const v = parseInt($('setValue').value, 10);
    if (Number.isFinite(v) && v >= 0) { state.count = v; save(); render(); status('Stand gesetzt auf ' + v); }
    else status('Bitte eine Zahl ≥ 0 eingeben.');
  });
  $('resetBtn').addEventListener('click', () => {
    state.count = 0; stripeOn = false; save(); render(); status('Zurückgesetzt');
  });
  $('target').addEventListener('change', e => { state.target = Math.max(1, parseInt(e.target.value, 10) || 200); save(); render(); });
  $('beepAt').addEventListener('change', e => { state.beepAt = Math.max(1, parseInt(e.target.value, 10) || 197); save(); render(); });
  $('thr').addEventListener('input', () => { save(); render(); });
  $('size').addEventListener('input', () => { save(); render(); drawRoiBox(); });

  load();
  render();
})();
</script>
</body>
</html>

