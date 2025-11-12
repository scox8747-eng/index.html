<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Casse.io</title>
  <style>
    body {
      margin: 0;
      background: #081020;
      color: white;
      font-family: Arial, sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      height: 100vh;
      overflow: hidden;
    }
    button {
      background: #2e8b57;
      color: white;
      border: none;
      padding: 10px 20px;
      border-radius: 8px;
      margin: 8px;
      cursor: pointer;
      font-size: 16px;
    }
    .overlay {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0, 0, 0, 0.8);
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .modal {
      background: #1c1c1c;
      color: white;
      padding: 20px;
      border-radius: 12px;
      text-align: center;
    }
  </style>
</head>
<body>
  <h1 id="title">🎮 Casse.io 🎮</h1>
  <p id="coins">💰 Pièces: 0</p>
  <div id="menu">
    <button id="playBtn">▶ Jouer</button>
    <button id="adminBtn">⚙ Admin</button>
    <button id="adBtn">🎥 Regarder une pub</button>
  </div>
  <canvas id="gameCanvas" width="768" height="480" style="display:none;border-radius:8px;box-shadow:0 6px 30px rgba(0,0,0,0.6);"></canvas>
  <p id="instructions" style="color:#ccc;display:none;">Marche sur les cases pour les faire tomber. Si tu tombes dans un trou, tu meurs !</p>

  <div id="adminPanel" class="overlay" style="display:none;">
    <div class="modal">
      <h2>⚙ Panneau Admin</h2>
      <p id="adminCoins">💰 Pièces: 0</p>
      <button id="buySpeed">Acheter vitesse (+50) - 10💰</button>
      <button id="buyInvincible">Acheter invincibilité (5s) - 20💰</button>
      <button id="closeAdmin">Fermer</button>
    </div>
  </div>

  <div id="adPanel" class="overlay" style="display:none;">
    <div class="modal">
      <h2>🎥 Regarde cette pub !</h2>
      <iframe width="420" height="236" src="https://www.youtube.com/embed/UGkRWUAoxfA" title="Pub" allowfullscreen></iframe>
      <button id="closeAd">Fermer</button>
    </div>
  </div>

  <div id="gameOver" class="overlay" style="display:none;">
    <div class="modal">
      <h2>💀 Tu es mort ! 💀</h2>
      <p id="finalCoins"></p>
      <button id="replayBtn">Rejouer</button>
      <button id="menuBtn">Aller au menu</button>
    </div>
  </div>

  <script>
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    const GRID_W = 16, GRID_H = 10, TILE_SIZE = 48, PLAYER_RADIUS = 14;
    const CRACK_TIME = 700, RESPAWN_TIME = 3000;
    let running = false, dead = false, invincible = false;
    let coins = 0, playerSpeed = 220;
    let coinTimer = 0;

    let tiles = [], player = {}, keys = {};

    function initGame() {
      tiles = [];
      for (let y = 0; y < GRID_H; y++) {
        tiles[y] = [];
        for (let x = 0; x < GRID_W; x++) {
          tiles[y][x] = { status: 'intact', crackStart: null, respawnAt: null };
        }
      }
      player = {
        x: (GRID_W / 2) * TILE_SIZE + TILE_SIZE / 2,
        y: (GRID_H / 2) * TILE_SIZE + TILE_SIZE / 2,
        vx: 0, vy: 0, speed: playerSpeed
      };
      coinTimer = performance.now();
    }

    function startGame() {
      running = true; dead = false;
      document.getElementById('menu').style.display = 'none';
      document.getElementById('gameCanvas').style.display = 'block';
      document.getElementById('instructions').style.display = 'block';
      initGame();
      loop();
    }

    function gameOver() {
      running = false; dead = true;
      document.getElementById('finalCoins').textContent = `Tu as gagné ${coins} pièces.`;
      document.getElementById('gameOver').style.display = 'flex';
    }

    function update(dt) {
      if (!running) return;
      const up = keys['z'] || keys['w'] || keys['arrowup'];
      const down = keys['s'] || keys['arrowdown'];
      const left = keys['q'] || keys['a'] || keys['arrowleft'];
      const right = keys['d'] || keys['arrowright'];

      let dx = 0, dy = 0;
      if (up) dy -= 1;
      if (down) dy += 1;
      if (left) dx -= 1;
      if (right) dx += 1;
      if (dx !== 0 || dy !== 0) {
        const len = Math.hypot(dx, dy);
        dx /= len; dy /= len;
      }

      player.x += dx * player.speed * dt;
      player.y += dy * player.speed * dt;
      player.x = Math.max(PLAYER_RADIUS, Math.min(GRID_W * TILE_SIZE - PLAYER_RADIUS, player.x));
      player.y = Math.max(PLAYER_RADIUS, Math.min(GRID_H * TILE_SIZE - PLAYER_RADIUS, player.y));

      const tileX = Math.floor(player.x / TILE_SIZE);
      const tileY = Math.floor(player.y / TILE_SIZE);
      if (tileX >= 0 && tileX < GRID_W && tileY >= 0 && tileY < GRID_H) {
        const tile = tiles[tileY][tileX];
        if (tile.status === 'intact') {
          tile.status = 'cracking';
          tile.crackStart = performance.now();
        }
        if (tile.status === 'fallen' && !invincible) {
          gameOver();
        }
      }

      const now = performance.now();
      if (now - coinTimer >= 1000) {
        coins++;
        document.getElementById('coins').textContent = `💰 Pièces: ${coins}`;
        coinTimer = now;
      }

      for (let y = 0; y < GRID_H; y++) {
        for (let x = 0; x < GRID_W; x++) {
          const t = tiles[y][x];
          if (t.status === 'cracking' && now - t.crackStart >= CRACK_TIME) {
            t.status = 'fallen';
            t.respawnAt = now + RESPAWN_TIME;
          } else if (t.status === 'fallen' && t.respawnAt && now >= t.respawnAt) {
            t.status = 'intact';
            t.crackStart = null;
            t.respawnAt = null;
          }
        }
      }
    }

    function draw() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.fillStyle = '#0b1220';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      for (let y = 0; y < GRID_H; y++) {
        for (let x = 0; x < GRID_W; x++) {
          const t = tiles[y][x];
          const px = x * TILE_SIZE;
          const py = y * TILE_SIZE;

          if (t.status === 'fallen') {
            ctx.fillStyle = '#000';
            ctx.fillRect(px + 2, py + 2, TILE_SIZE - 4, TILE_SIZE - 4);
            continue;
          }

          if (t.status === 'intact') ctx.fillStyle = '#2e8b57';
          else {
            const elapsed = Math.min(1, (performance.now() - t.crackStart) / CRACK_TIME);
            const shade = Math.floor(200 - elapsed * 120);
            ctx.fillStyle = `rgb(${shade},120,${shade - 30})`;
          }
          ctx.fillRect(px + 2, py + 2, TILE_SIZE - 4, TILE_SIZE - 4);
        }
      }

      ctx.beginPath();
      ctx.fillStyle = invincible ? '#00ffff' : '#ffd166';
      ctx.arc(player.x, player.y, PLAYER_RADIUS, 0, Math.PI * 2);
      ctx.fill();
    }

    let last = performance.now();
    function loop() {
      if (!running) return;
      const now = performance.now();
      const dt = (now - last) / 1000;
      last = now;
      update(dt);
      draw();
      requestAnimationFrame(loop);
    }

    document.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
    document.addEventListener('keyup', e => delete keys[e.key.toLowerCase()]);

    document.getElementById('playBtn').onclick = startGame;
    document.getElementById('adminBtn').onclick = () => document.getElementById('adminPanel').style.display = 'flex';
    document.getElementById('closeAdmin').onclick = () => document.getElementById('adminPanel').style.display = 'none';
    document.getElementById('adBtn').onclick = () => document.getElementById('adPanel').style.display = 'flex';
    document.getElementById('closeAd').onclick = () => {
      document.getElementById('adPanel').style.display = 'none';
      coins += 5;
      document.getElementById('coins').textContent = `💰 Pièces: ${coins}`;
    };
    document.getElementById('replayBtn').onclick = () => {
      document.getElementById('gameOver').style.display = 'none';
      startGame();
    };
    document.getElementById('menuBtn').onclick = () => {
      document.getElementById('gameOver').style.display = 'none';
      document.getElementById('menu').style.display = 'block';
      document.getElementById('gameCanvas').style.display = 'none';
      document.getElementById('instructions').style.display = 'none';
    };

    document.getElementById('buySpeed').onclick = () => {
      if (coins >= 10) {
        coins -= 10;
        playerSpeed += 50;
        alert('Vitesse augmentée !');
      } else alert('Pas assez de pièces !');
      document.getElementById('adminCoins').textContent = `💰 Pièces: ${coins}`;
    };

    document.getElementById('buyInvincible').onclick = () => {
      if (coins >= 20) {
        coins -= 20;
        invincible = true;
        setTimeout(() => invincible = false, 5000);
        alert('Invincibilité pendant 5 secondes !');
      } else alert('Pas assez de pièces !');
      document.getElementById('adminCoins').textContent = `💰 Pièces: ${coins}`;
    };
  </script>
</body>
</html>
