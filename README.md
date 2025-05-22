# run-from-plague
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Run from the Plague!</title>
<style>
  html, body {
    margin: 0; padding: 0; overflow: hidden; background: #f0e6d2;
    font-family: 'Georgia', serif;
  }
  #gameCanvas {
    display: block;
    margin: 0 auto;
    background: linear-gradient(to top, #a3c2a3 0%, #f0e6d2 100%);
    touch-action: none;
  }
  #score {
    position: fixed;
    top: 10px; left: 50%;
    transform: translateX(-50%);
    font-size: 24px;
    color: #4a3e31;
    font-weight: bold;
    text-shadow: 1px 1px 2px #fff;
  }
</style>
</head>
<body>
<div id="score">Score: 0</div>
<canvas id="gameCanvas" width="400" height="600"></canvas>

<script>
(() => {
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');

  const lanesX = [80, 160, 240]; // 3 lanes
  const groundY = 500;

  // Game variables
  let playerLane = 1; // middle lane index 0,1,2
  let playerY = groundY;
  let isJumping = false;
  let jumpVelocity = 0;
  let gravity = 0.6;
  let score = 0;
  let gameSpeed = 5;
  let frameCount = 0;

  // Objects arrays
  let medicines = [];
  let plagueFogs = [];

  // Player image (simple ancient greek runner with laurel)
  function drawPlayer(x, y) {
    ctx.fillStyle = '#d1cfa6';
    ctx.beginPath();
    ctx.ellipse(x, y - 30, 15, 25, 0, 0, Math.PI * 2);
    ctx.fill();
    ctx.fillStyle = '#4a3e31'; // robes
    ctx.fillRect(x - 10, y - 10, 20, 30);
    // laurel wreath
    ctx.strokeStyle = '#4caf50';
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.arc(x, y - 45, 15, 0, Math.PI * 2);
    ctx.stroke();
  }

  // Medicine collectible
  function drawMedicine(x, y) {
    ctx.fillStyle = '#5bc0de'; // medicine blue bottle
    ctx.fillRect(x - 8, y - 20, 16, 30);
    ctx.fillStyle = 'white';
    ctx.fillRect(x - 4, y - 10, 8, 10);
  }

  // Plague fog (enemy)
  function drawPlague(x, y) {
    const gradient = ctx.createRadialGradient(x, y, 10, x, y, 40);
    gradient.addColorStop(0, 'rgba(139,0,0,0.8)');
    gradient.addColorStop(1, 'rgba(139,0,0,0)');
    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.arc(x, y, 40, 0, Math.PI * 2);
    ctx.fill();
    // eyes
    ctx.fillStyle = 'yellow';
    ctx.beginPath();
    ctx.ellipse(x - 15, y - 10, 6, 10, 0, 0, Math.PI * 2);
    ctx.ellipse(x + 15, y - 10, 6, 10, 0, 0, Math.PI * 2);
    ctx.fill();
  }

  function resetGame() {
    playerLane = 1;
    playerY = groundY;
    isJumping = false;
    jumpVelocity = 0;
    score = 0;
    medicines = [];
    plagueFogs = [];
    frameCount = 0;
    gameSpeed = 5;
  }

  // Check collision between two rectangles
  function collideRect(x1, y1, w1, h1, x2, y2, w2, h2) {
    return !(x2 > x1 + w1 ||
             x2 + w2 < x1 ||
             y2 > y1 + h1 ||
             y2 + h2 < y1);
  }

  function gameOver() {
    alert('The plague caught you! Your final score: ' + score);
    resetGame();
  }

  // Touch handling for swipe
  let touchStartX = null;
  let touchStartY = null;

  canvas.addEventListener('touchstart', e => {
    const t = e.touches[0];
    touchStartX = t.clientX;
    touchStartY = t.clientY;
  });

  canvas.addEventListener('touchend', e => {
    if (touchStartX === null || touchStartY === null) return;
    const t = e.changedTouches[0];
    const dx = t.clientX - touchStartX;
    const dy = t.clientY - touchStartY;

    if (Math.abs(dx) > Math.abs(dy)) {
      // Horizontal swipe
      if (dx > 30) moveRight();
      else if (dx < -30) moveLeft();
    } else {
      // Vertical swipe
      if (dy < -30) jump();
    }

    touchStartX = null;
    touchStartY = null;
  });

  // Keyboard controls
  window.addEventListener('keydown', e => {
    if (e.key === 'ArrowLeft') moveLeft();
    else if (e.key === 'ArrowRight') moveRight();
    else if (e.key === 'ArrowUp') jump();
  });

  function moveLeft() {
    if (playerLane > 0) playerLane--;
  }
  function moveRight() {
    if (playerLane < lanesX.length - 1) playerLane++;
  }
  function jump() {
    if (!isJumping) {
      isJumping = true;
      jumpVelocity = -12;
    }
  }

  // Main game loop
  function update() {
    frameCount++;
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Update player jump
    if (isJumping) {
      playerY += jumpVelocity;
      jumpVelocity += gravity;
      if (playerY > groundY) {
        playerY = groundY;
        isJumping = false;
        jumpVelocity = 0;
      }
    }

    // Spawn medicines randomly
    if (frameCount % 90 === 0) {
      medicines.push({
        lane: Math.floor(Math.random() * lanesX.length),
        y: -50,
        collected: false
      });
    }

    // Spawn plague fog sometimes (but after some time)
    if (frameCount > 300 && frameCount % 150 === 0) {
      plagueFogs.push({
        lane: Math.floor(Math.random() * lanesX.length),
        y: -60,
        speed: gameSpeed + 1
      });
    }

    // Draw and update medicines
    medicines.forEach(m => {
      m.y += gameSpeed;
      if (!m.collected) drawMedicine(lanesX[m.lane], m.y);
    });

    // Draw and update plague fogs
    plagueFogs.forEach((p, i) => {
      p.y += p.speed;
      drawPlague(lanesX[p.lane], p.y);
      // Check collision with player (simple box approx)
      if (!isJumping && p.lane === playerLane && p.y > playerY - 50 && p.y < playerY + 20) {
        gameOver();
      }
      // Remove plague if off screen
      if (p.y > canvas.height + 50) plagueFogs.splice(i, 1);
    });

    // Check medicine collection
    medicines.forEach((m, i) => {
      if (!m.collected && m.lane === playerLane && m.y > playerY - 40 && m.y < playerY + 20) {
        m.collected = true;
        score++;
        gameSpeed += 0.1; // speed up as you collect more
        document.getElementById('score').innerText = 'Score: ' + score;
        medicines.splice(i, 1);
      }
    });

    // Draw player
    drawPlayer(lanesX[playerLane], playerY);

    requestAnimationFrame(update);
  }

  resetGame();
  update();
})();
</script>
</body>
</html>
