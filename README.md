<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Decline Runner</title>
  <style>
    canvas {
      display: block;
      margin: 0 auto;
      background: white;
      border: 2px solid black;
    }
    body {
      text-align: center;
      font-family: sans-serif;
    }
  </style>
</head>
<body>
  <h1>Decline Runner</h1>
  <canvas id="gameCanvas" width="800" height="600"></canvas>
  <p><b>Press [Space] to Jump • Press [Enter] to Restart</b></p>

  <script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");

    const width = canvas.width;
    const height = canvas.height;

    // --- Game Constants ---
    const slopeAngle = 25 * Math.PI / 180;
    const playerRadius = 20;
    const gravity = 0.5;
    const jumpStrength = -12;
    const obstacleGap = 350;
    const baseScrollSpeed = 3;
    const maxScrollSpeed = 8;

    let playerX = width / 3;
    let playerY = 0;
    let playerVelY = 0;
    let onGround = true;

    let score = 0;
    let scrollSpeed = baseScrollSpeed;
    let gameState = "menu";
    let highScore = 0;

    let obstacles = [];

    function getGroundY(x) {
      return 100 + x * Math.tan(slopeAngle);
    }

    function resetGame() {
      playerY = getGroundY(playerX) - playerRadius;
      playerVelY = 0;
      onGround = true;
      score = 0;
      scrollSpeed = baseScrollSpeed;
      obstacles = [];

      let lastX = width;
      for (let i = 0; i < 5; i++) {
        const x = lastX + obstacleGap + Math.random() * 400;
        obstacles.push({
          x,
          width: 30 + Math.random() * 20,
          height: 30 + Math.random() * 10,
        });
        lastX = x;
      }
    }

    function drawGround() {
      ctx.beginPath();
      ctx.moveTo(0, getGroundY(0));
      ctx.lineTo(width, getGroundY(width));
      ctx.strokeStyle = "black";
      ctx.lineWidth = 2;
      ctx.stroke();
    }

    function drawPlayer() {
      ctx.beginPath();
      ctx.arc(playerX, playerY, playerRadius, 0, Math.PI * 2);
      ctx.fillStyle = "blue";
      ctx.fill();
    }

    function drawObstacles() {
      ctx.fillStyle = "red";
      for (let obs of obstacles) {
        const y = getGroundY(obs.x);
        ctx.fillRect(obs.x, y - obs.height, obs.width, obs.height);
      }
    }

    function drawText(text, x, y, size = "24px", color = "black") {
      ctx.fillStyle = color;
      ctx.font = `${size} sans-serif`;
      ctx.fillText(text, x, y);
    }

    function update() {
      if (gameState !== "playing") return;

      playerVelY += gravity;
      playerY += playerVelY;

      const groundY = getGroundY(playerX);
      if (playerY >= groundY - playerRadius) {
        playerY = groundY - playerRadius;
        playerVelY = 0;
        onGround = true;
      }

      scrollSpeed = Math.min(maxScrollSpeed, baseScrollSpeed + Math.floor(score / 5));

      for (let obs of obstacles) {
        obs.x -= scrollSpeed;
      }

      // Remove and add obstacles
      if (obstacles.length && obstacles[0].x + obstacles[0].width < 0) {
        obstacles.shift();
        const lastX = obstacles[obstacles.length - 1].x;
        const newX = lastX + obstacleGap + Math.random() * 400;
        obstacles.push({
          x:
