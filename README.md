<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Meteor Dodge — Full</title>
<style>
:root{--bg:#061324;--fg:#e6f1ff;--accent:#ffb86b;}
html,body{height:100%;margin:0;font-family:Inter,system-ui,Arial;background:linear-gradient(180deg,#020814,#061324);color:var(--fg);display:flex;justify-content:center;}
.wrap{width:980px;max-width:96vw;background:linear-gradient(180deg,rgba(255,255,255,0.02),rgba(255,255,255,0.01));border-radius:12px;padding:18px;box-shadow:0 12px 30px rgba(2,6,23,0.7);display:flex;flex-direction:column;align-items:center;}
h1{margin:0 0 6px;font-size:20px}
.lead{margin:0;color:#9fb0c8;font-size:13px;text-align:center}
#game{background:#021320;border-radius:8px;margin-top:12px;display:block}
.hud{display:flex;justify-content:space-between;align-items:center;margin-top:12px;width:100%;gap:8px;flex-wrap:wrap}
.controls{display:flex;gap:8px;align-items:center}
.btn{background:var(--accent);border:none;padding:8px 12px;border-radius:8px;color:#0b1220;font-weight:700;cursor:pointer}
.muted{color:#86a0b6;font-size:13px}
.music-controls{display:flex;align-items:center;gap:8px}
input[type=range]{width:110px}
.video-controls{margin-top:12px;display:flex;gap:8px;align-items:center}
#videoContainer{margin-top:12px;width:560px;height:315px;border:1px solid #444;border-radius:8px;background:#000}
.powerText{position:absolute;color:#fff;font-size:18px;font-weight:800;text-shadow:0 0 8px #000;pointer-events:none;opacity:1;transition:opacity 0.9s ease-out;}
.footer{margin-top:12px;color:#86a0b6;font-size:13px;text-align:center}
</style>
</head>
<body>
  <div class="wrap">
    <h1>Meteor Dodge</h1>
    <div class="lead">Move your ship left/right (A/D or ←/→). Press W to rise temporarily. Collect power-ups: Shield, Slow Time, Score Boost, Nyan Cat (rainbow!). Press R to restart.</div>

    <canvas id="game" width="880" height="480"></canvas>

    <div class="hud">
      <div class="muted">Controls: A / D / ← / → — W to go up — R to restart</div>
      <div class="controls">
        <div class="muted">Score: <strong id="score">0</strong></div>
        <button id="restart" class="btn">Restart</button>
        <div class="music-controls">
          <button id="muteBtn" class="btn">Mute</button>
          <input id="volumeSlider" type="range" min="0" max="100" value="50">
        </div>
      </div>
    </div>

    <div class="video-controls" style="justify-content:center;">
      <input id="youtubeInput" type="text" placeholder="Paste YouTube link (https://... or youtu.be/...)"
             style="width:360px;padding:8px;border-radius:8px;border:1px solid #444;background:#021320;color:#fff">
      <button id="loadVideo" class="btn">Load Video</button>
    </div>

    <div id="videoContainer"></div>

    <div class="footer">Tip: Save as <code>meteor_dodge.html</code> and open in Chrome/Edge. Autoplay may be blocked until you interact with the page (click/tap).</div>
  </div>

  <!-- YouTube API loader (the real API will call the global callback on load) -->
  <script src="https://www.youtube.com/iframe_api"></script>

  <script>
  // ----------------------------
  // GAME CORE (PART 1 of 2)
  // We'll keep game logic here; YouTube code will be attached in PART 2.
  // ----------------------------
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;

  // Game state
  let ship, meteors, powerUps, keys, score, gameOver, frames, lastTime, activeEffects;
  const scoreEl = document.getElementById('score');

  // Starfield (slower)
  const starField = Array.from({length:90}, () => ({x: Math.random()*W, y: Math.random()*H, r: Math.random()*1.2+0.2, speed: Math.random()*0.2+0.05}));

  function initGame(){
    ship = { x: W/2, y: H - 70, w: 48, h: 28, vx:0, speed: 4.2, nyan:false, shield:false, glow:false };
    meteors = [];
    powerUps = [];
    keys = {};
    score = 0;
    gameOver = false;
    frames = 0;
    lastTime = performance.now();
    activeEffects = [];
    scoreEl.textContent = 0;
    // start with a few powerups for you to see
    for(let i=0;i<2;i++) spawnPowerUp();
    requestAnimationFrame(loop);
  }

  // Input
  window.addEventListener('keydown', e => {
    keys[e.key.toLowerCase()] = true;
    if(e.key.toLowerCase() === 'r') { initGame(); }
  });
  window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

  document.getElementById('restart').addEventListener('click', ()=> initGame());

  // Spawners
  function spawnMeteor(){
    const size = Math.random()*36 + 18;
    meteors.push({
      x: Math.random()*(W - size) + size/2,
      y: -size,
      size,
      vy: Math.random()*1.2 + 0.9,
      rot: Math.random()*Math.PI*2,
      spin: (Math.random()*0.06 - 0.03),
      color: ['#ff6b6b','#ffd89b','#ffcc88','#99d8ff','#86e3ce'][Math.floor(Math.random()*5)],
      shape: Math.floor(Math.random()*3)
    });
  }

  function spawnPowerUp(){
    const types = ['shield','slow','score','nyan'];
    const type = types[Math.floor(Math.random()*types.length)];
    const size = 30;
    powerUps.push({ x: Math.random()*(W-size)+size/2, y: -size, size, vy: 2.2, type });
  }

  // collision helper
  function circleRectCollide(cx, cy, r, rx, ry, rw, rh){
    const dx = Math.abs(cx - rx);
    const dy = Math.abs(cy - ry);
    if (dx > (rw/2 + r)) return false;
    if (dy > (rh/2 + r)) return false;
    if (dx <= (rw/2)) return true;
    if (dy <= (rh/2)) return true;
    const dx2 = dx - rw/2;
    const dy2 = dy - rh/2;
    return (dx2*dx2 + dy2*dy2 <= r*r);
  }

  function showPowerText(text, x, y){
    const div = document.createElement('div');
    div.className = 'powerText';
    // position in viewport: canvas is centered, so offset by canvas bounding rect
    const rect = canvas.getBoundingClientRect();
    div.style.left = (rect.left + x - 24) + 'px';
    div.style.top = (rect.top + y - 40) + 'px';
    div.textContent = text;
    document.body.appendChild(div);
    setTimeout(()=> { div.style.opacity = '0'; setTimeout(()=> div.remove(), 900); }, 600);
  }

  // Apply power-up effects
  function applyPowerUp(type){
    if(type === 'shield'){
      ship.shield = true;
      activeEffects.push({ type:'shield', duration: 300 });
    } else if(type === 'slow'){
      activeEffects.push({ type:'slow', duration: 300 });
      // slow effect applied by factor during update (we check activeEffects)
    } else if(type === 'score'){
      ship.glow = true;
      activeEffects.push({ type:'score', duration: 300 });
    } else if(type === 'nyan'){
      ship.nyan = true;
      activeEffects.push({ type:'nyan', duration: 300 });
    }
  }

  function getRandomWord(){
    const words = ['Nice!','Good job!','Awesome!','Great!','Well done!'];
    return words[Math.floor(Math.random()*words.length)];
  }

  // draw helpers
  function drawStars(){
    ctx.fillStyle = '#021320';
    ctx.fillRect(0,0,W,H);
    for(const s of starField){
      s.y += s.speed;
      if(s.y > H) s.y = 0;
      ctx.fillStyle = `rgba(255,255,255,${0.15 + s.r*0.25})`;
      ctx.fillRect(s.x, s.y, s.r, s.r);
    }
  }

  function draw(){
    drawStars();

    // meteors
    for(const m of meteors){
      ctx.save();
      ctx.translate(m.x, m.y);
      ctx.rotate(m.rot);
      ctx.fillStyle = m.color;
      if(m.shape === 0){
        ctx.beginPath(); ctx.arc(0,0, m.size*0.5, 0, Math.PI*2); ctx.fill();
      } else if(m.shape === 1){
        ctx.beginPath(); ctx.ellipse(0,0, m.size*0.6, m.size*0.4, 0, 0, Math.PI*2); ctx.fill();
      } else {
        ctx.beginPath();
        for(let i=0;i<5;i++){
          const angle = i*(Math.PI*2/5);
          const rad = m.size*0.5*(0.7 + Math.random()*0.6);
          ctx.lineTo(Math.cos(angle)*rad, Math.sin(angle)*rad);
        }
        ctx.closePath(); ctx.fill();
      }
      ctx.restore();
    }

    // powerups: black disc with colored stroke (distinguishable)
    for(const p of powerUps){
      ctx.save();
      ctx.translate(p.x, p.y);
      ctx.fillStyle = '#000';
      ctx.beginPath(); ctx.arc(0,0, p.size*0.5, 0, Math.PI*2); ctx.fill();
      ctx.strokeStyle = (p.type==='shield'?'#00ffcc':p.type==='slow'?'#ff00ff':p.type==='score'?'#ffff66':'#ff6699');
      ctx.lineWidth = 4;
      ctx.stroke();
      ctx.restore();
    }

    // ship
    ctx.save();
    ctx.translate(ship.x, ship.y);
    if(ship.nyan){
      // body
      ctx.fillStyle = '#ff99ff';
      ctx.fillRect(-ship.w/2, -12, ship.w, 18);
      // rainbow trail behind (thruster)
      for(let i=0;i<10;i++){
        ctx.fillStyle = `hsl(${i*36},100%,50%)`;
        ctx.fillRect(-ship.w/2 - (i*6 + 4), -2, 6, 6 + Math.sin(frames/6 + i)*2);
      }
    } else {
      // normal ship
      ctx.fillStyle = '#ffcc88';
      ctx.beginPath(); ctx.moveTo(-14,14); ctx.lineTo(0,26); ctx.lineTo(14,14); ctx.closePath(); ctx.fill();
      ctx.fillStyle = '#99d8ff';
      ctx.beginPath(); ctx.moveTo(-24,8); ctx.lineTo(0,-20); ctx.lineTo(24,8); ctx.closePath(); ctx.fill();
      ctx.fillStyle = '#0b1220';
      ctx.beginPath(); ctx.ellipse(0,0,8,6,0,0,Math.PI*2); ctx.fill();
    }
    // shield ring
    if(ship.shield){
      ctx.strokeStyle = '#00ffcc';
      ctx.lineWidth = 4;
      ctx.beginPath(); ctx.arc(0,0,36,0,Math.PI*2); ctx.stroke();
    }
    // glow
    if(ship.glow){
      ctx.shadowColor = '#ffff66'; ctx.shadowBlur = 18;
    } else {
      ctx.shadowBlur = 0;
    }
    ctx.restore();
  }

  function showGameOver(){
    ctx.fillStyle = 'rgba(0,0,0,0.5)';
    ctx.fillRect(0,0,W,H);
    ctx.fillStyle = '#fff';
    ctx.font = '28px Inter, Arial';
    ctx.textAlign = 'center';
    ctx.fillText('Game Over', W/2, H/2 - 10);
    ctx.font = '16px Inter, Arial';
    ctx.fillText('Score: ' + Math.floor(score), W/2, H/2 + 22);
    ctx.font = '14px Inter, Arial';
    ctx.fillText('Press Restart or R to play again', W/2, H/2 + 48);
  }

  // --- loop continues in PART 2 (contains update, loop, initGame() call, and YouTube handlers) ---
  </script>
  <script>
  // PART 2: update/loop, event hooks that affect game speed, YouTube setup, and finalization.

  // Update variables and continuation
  function loop(timestamp){
    let delta = (timestamp - lastTime) / 1000;
    lastTime = timestamp;

    // Movement input
    ship.vx = 0;
    if(keys['arrowleft'] || keys['a']) ship.vx = -ship.speed;
    if(keys['arrowright'] || keys['d']) ship.vx = ship.speed;
    // W to rise (temporary upward movement)
    if(keys['w']) ship.y = Math.max(20, ship.y - 4);
    else ship.y = Math.min(H - 40, ship.y + 1.2);

    ship.x = Math.max(30, Math.min(W - 30, ship.x + ship.vx));

    frames++;
    // spawn rates tuned
    if(frames % 140 === 0) spawnMeteor();
    if(frames % 420 === 0) spawnPowerUp();

    // meteor updates (apply slow if active)
    const slowActive = activeEffects.some(e => e.type === 'slow');
    for(let i = meteors.length-1; i >= 0; i--){
      const m = meteors[i];
      m.y += m.vy * (slowActive ? 0.45 : 1);
      m.rot += m.spin;
      // remove off-screen
      if(m.y - m.size > H) meteors.splice(i,1);
      // collision (circle-rect approximate)
      if(circleRectCollide(m.x, m.y, m.size*0.45, ship.x, ship.y, ship.w, ship.h)){
        // if shield not active -> game over
        if(!ship.shield) { gameOver = true; break; }
        else {
          // consume meteor with shield visual: remove meteor and slightly reduce shield duration
          meteors.splice(i,1);
          // shorten shield a bit
          const sh = activeEffects.find(e => e.type === 'shield');
          if(sh) sh.duration = Math.max(0, sh.duration - 80);
        }
      }
    }

    // powerups update
    for(let i = powerUps.length-1; i >= 0; i--){
      const p = powerUps[i];
      p.y += p.vy;
      if(p.y - p.size > H) powerUps.splice(i,1);
      else if(circleRectCollide(p.x, p.y, p.size*0.45, ship.x, ship.y, ship.w, ship.h)){
        applyPowerUp(p.type);
        showPowerText(getRandomWord(), ship.x, ship.y - 36);
        powerUps.splice(i,1);
      }
    }

    // update active effects durations
    for(let i = activeEffects.length-1; i >= 0; i--){
      activeEffects[i].duration--;
      if(activeEffects[i].duration <= 0){
        const t = activeEffects[i].type;
        if(t === 'nyan') ship.nyan = false;
        if(t === 'shield') ship.shield = false;
        if(t === 'score') ship.glow = false;
        activeEffects.splice(i,1);
      }
    }

    // score: about 10 points/sec, doubled when score boost active
    const scoreBoost = activeEffects.some(e => e.type === 'score');
    score += delta * 10 * (scoreBoost ? 2 : 1);
    scoreEl.textContent = Math.floor(score);

    // draw
    draw();

    if(!gameOver) requestAnimationFrame(loop);
    else showGameOver();
  }

  // Start the game when page ready
  initGame();

  // --- YouTube Player integration (controls & loading) ---
  // We'll create the player when you load a video; ensure global function for API if needed.
  let player = null;

  // Provide the standard API-ready global in case the API loads after this script.
  function onYouTubeIframeAPIReady(){
    // no auto player until user loads a video
  }
  // expose globally (some browsers need global reference)
  window.onYouTubeIframeAPIReady = onYouTubeIframeAPIReady;

  // Load button -> parse URL and create iframe player via YT API
  const loadBtn = document.getElementById('loadVideo');
  const youtubeInput = document.getElementById('youtubeInput');
  const videoContainer = document.getElementById('videoContainer');
  loadBtn.addEventListener('click', () => {
    const url = youtubeInput.value.trim();
    // accept youtube.com/watch?v=ID or youtu.be/ID or full embed url
    const match = url.match(/(?:v=|\/)([A-Za-z0-9_-]{11})(?:&|$)/) || url.match(/youtu\.be\/([A-Za-z0-9_-]{11})/);
    if(!match){
      alert('Invalid YouTube link — paste a normal video URL (watch?v=...).');
      return;
    }
    const videoId = match[1];
    // create player container
    videoContainer.innerHTML = '<div id="ytPlayer"></div>';
    // destroy old player if any
    try { if(player && player.destroy) player.destroy(); } catch(e){}
    // create new player (no autoplay by default)
    player = new YT.Player('ytPlayer', {
      height: '315',
      width: '560',
      videoId: videoId,
      playerVars: { autoplay: 0, modestbranding: 1 },
      events: {
        'onReady': (evt) => {
          // set initial volume to slider value
          const vol = parseInt(document.getElementById('volumeSlider').value, 10);
          if(!isNaN(vol)) evt.target.setVolume(vol);
        }
      }
    });
  });

  // Mute / Volume UI
  const muteBtn = document.getElementById('muteBtn');
  const volumeSlider = document.getElementById('volumeSlider');

  muteBtn.addEventListener('click', () => {
    if(!player) return;
    if(player.isMuted()){
      player.unMute();
      muteBtn.textContent = 'Mute';
    } else {
      player.mute();
      muteBtn.textContent = 'Unmute';
    }
  });

  volumeSlider.addEventListener('input', (e) => {
    const v = Number(e.target.value);
    if(player && !isNaN(v)) player.setVolume(v);
  });

  // Preserve global references (useful if user pastes partial)
  window.initGame = initGame;
  window.spawnMeteor = spawnMeteor;
  window.spawnPowerUp = spawnPowerUp;
  window.applyPowerUp = applyPowerUp;

  </script>
</body>
</html>
