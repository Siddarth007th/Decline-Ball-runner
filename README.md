<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Decline Runner</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            background-color: #f0f0f0;
            font-family: 'Arial', sans-serif;
        }
        #game-container {
            position: relative;
            border: 2px solid #333;
            box-shadow: 0 0 15px rgba(0,0,0,0.2);
        }
        canvas {
            display: block;
            background-color: #fff;
        }
        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none; /* Allows clicks to go through to the canvas if needed */
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            color: #333;
            text-align: center;
        }
        .hidden {
            display: none;
        }
        h1 {
            font-size: 3rem;
            margin: 0;
        }
        p {
            font-size: 1.2rem;
            margin: 10px 0;
        }
        #score-display {
            position: absolute;
            top: 10px;
            left: 10px;
            font-size: 1.5rem;
            text-align: left;
        }
    </style>
</head>
<body>
    <div id="game-container">
        <canvas id="gameCanvas" width="800" height="600"></canvas>
        <div id="ui-layer">
            <!-- Menu Screen -->
            <div id="menu-screen">
                <h1>Decline Runner</h1>
                <p>Press ENTER to Start</p>
                <p id="menu-high-score">High Score: 0</p>
            </div>
            <!-- Game Over Screen -->
            <div id="game-over-screen" class="hidden">
                <h1>Game Over</h1>
                <p id="final-score">Your Score: 0</p>
                <p id="game-over-high-score">High Score: 0</p>
                <p>Press ENTER to Restart</p>
            </div>
             <!-- Score Display during gameplay -->
            <div id="score-display" class="hidden">
                <p>Score: <span id="current-score">0</span></p>
                <p>High Score: <span id="gameplay-high-score">0</span></p>
            </div>
        </div>
    </div>
 
    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
 
        // UI Elements
        const menuScreen = document.getElementById('menu-screen');
        const gameOverScreen = document.getElementById('game-over-screen');
        const scoreDisplay = document.getElementById('score-display');
        const menuHighScore = document.getElementById('menu-high-score');
        const gameOverHighScore = document.getElementById('game-over-high-score');
        const gameplayHighScore = document.getElementById('gameplay-high-score');
        const finalScore = document.getElementById('final-score');
        const currentScoreSpan = document.getElementById('current-score');
 
        class Obstacle {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.shapeType = ['square', 'rectangle', 'triangle'][Math.floor(Math.random() * 3)];
                
                let size, width, height;
                switch (this.shapeType) {
                    case 'square':
                        size = 30;
                        this.width = size;
                        this.height = size;
                        break;
                    case 'rectangle':
                        width = 45;
                        height = 25;
                        this.width = width;
                        this.height = height;
                        break;
                    case 'triangle':
                        size = 35;
                        this.width = size;
                        this.height = size;
                        break;
                }
            }
 
            draw() {
                ctx.fillStyle = '#ff3232'; // Red
                if (this.shapeType === 'square' || this.shapeType === 'rectangle') {
                    ctx.fillRect(this.x, this.y - this.height, this.width, this.height);
                } else if (this.shapeType === 'triangle') {
                    ctx.beginPath();
                    ctx.moveTo(this.x, this.y - this.height); // Top point
                    ctx.lineTo(this.x + this.width, this.y); // Bottom right
                    ctx.lineTo(this.x, this.y); // Bottom left
                    ctx.closePath();
                    ctx.fill();
                }
            }
        }
 
        class Game {
            constructor() {
                this.width = canvas.width;
                this.height = canvas.height;
                
                // Game settings
                this.slopeAngle = Math.atan(0.25); // ~14 degrees
                this.groundYStart = 100;
                this.baseScrollSpeed = 3;
                this.maxScrollSpeed = 8;
                
                // Player properties
                this.playerRadius = 20;
                this.gravity = 0.5;
                this.jumpStrength = -12;
 
                // Obstacle properties
                this.obstacleGap = 350;
 
                this.highScore = this.loadHighScore();
                this.gameState = 'menu';
                this.init();
            }
 
            init() {
                this.playerX = 100;
                this.playerY = this.getGroundY(this.playerX);
                this.playerVelY = 0;
                this.onGround = true;
                this.score = 0;
                this.scrollSpeed = this.baseScrollSpeed;
                this.obstacles = [];
 
                let lastX = this.width;
                for (let i = 0; i < 5; i++) {
                    const spawnX = lastX + this.obstacleGap + Math.random() * 400;
                    const spawnY = this.getGroundY(spawnX);
                    this.obstacles.push(new Obstacle(spawnX, spawnY));
                    lastX = spawnX;
                }
                this.updateUI();
            }
            
            getGroundY(xPos) {
                return this.groundYStart + xPos * Math.tan(this.slopeAngle);
            }
 
            loadHighScore() {
                return parseInt(localStorage.getItem('declineRunnerHighScore')) || 0;
            }
 
            saveHighScore() {
                localStorage.setItem('declineRunnerHighScore', this.highScore);
            }
 
            start() {
                this.gameState = 'playing';
                this.init();
                gameLoop();
            }
 
            update() {
                if (this.gameState !== 'playing') return;
 
                // Difficulty Scaling
                this.scrollSpeed = Math.min(this.maxScrollSpeed, this.baseScrollSpeed + this.score / 5);
 
                // Player Physics
                this.playerVelY += this.gravity;
                this.playerY += this.playerVelY;
 
                const groundY = this.getGroundY(this.playerX);
                if (this.playerY >= groundY - this.playerRadius) {
                    this.playerY = groundY - this.playerRadius;
                    this.playerVelY = 0;
                    this.onGround = true;
                }
 
                // Obstacle Update
                this.obstacles.forEach(obstacle => {
                    obstacle.x -= this.scrollSpeed;
                });
 
                if (this.obstacles[0].x + this.obstacles[0].width < 0) {
                    this.obstacles.shift();
                    const lastX = this.obstacles[this.obstacles.length - 1].x;
                    const spawnX = lastX + this.obstacleGap + Math.random() * 400;
                    const spawnY = this.getGroundY(spawnX);
                    this.obstacles.push(new Obstacle(spawnX, spawnY));
                    this.score++;
                }
 
                // Collision Detection
                const playerRect = {
                    x: this.playerX - this.playerRadius,
                    y: this.playerY - this.playerRadius,
                    width: this.playerRadius * 2,
                    height: this.playerRadius * 2
                };
 
                for (const obstacle of this.obstacles) {
                    const obsRect = {
                        x: obstacle.x,
                        y: obstacle.y - obstacle.height,
                        width: obstacle.width,
                        height: obstacle.height
                    };
                    if (playerRect.x < obsRect.x + obsRect.width &&
                        playerRect.x + playerRect.width > obsRect.x &&
                        playerRect.y < obsRect.y + obsRect.height &&
                        playerRect.y + playerRect.height > obsRect.y) {
                        this.gameState = 'gameOver';
                        if (this.score > this.highScore) {
                            this.highScore = this.score;
                            this.saveHighScore();
                        }
                    }
                }
                this.updateUI();
            }
 
            draw() {
                ctx.clearRect(0, 0, this.width, this.height);
 
                // Draw Platform
                ctx.strokeStyle = '#333';
                ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.moveTo(0, this.getGroundY(0));
                ctx.lineTo(this.width, this.getGroundY(this.width));
                ctx.stroke();
 
                // Draw Player
                ctx.fillStyle = '#0064ff'; // Blue
                ctx.beginPath();
                ctx.arc(this.playerX, this.playerY, this.playerRadius, 0, Math.PI * 2);
                ctx.fill();
 
                // Draw Obstacles
                this.obstacles.forEach(obstacle => obstacle.draw());
            }
            
            updateUI() {
                if (this.gameState === 'menu') {
                    menuScreen.classList.remove('hidden');
                    gameOverScreen.classList.add('hidden');
                    scoreDisplay.classList.add('hidden');
                    menuHighScore.textContent = `High Score: ${this.highScore}`;
                } else if (this.gameState === 'playing') {
                    menuScreen.classList.add('hidden');
                    gameOverScreen.classList.add('hidden');
                    scoreDisplay.classList.remove('hidden');
                    currentScoreSpan.textContent = this.score;
                    gameplayHighScore.textContent = this.highScore;
                } else if (this.gameState === 'gameOver') {
                    menuScreen.classList.add('hidden');
                    gameOverScreen.classList.remove('hidden');
                    scoreDisplay.classList.add('hidden');
                    finalScore.textContent = `Your Score: ${this.score}`;
                    gameOverHighScore.textContent = `High Score: ${this.highScore}`;
                }
            }
        }
 
        const game = new Game();
 
        function gameLoop() {
            if (game.gameState === 'playing') {
                game.update();
                game.draw();
                requestAnimationFrame(gameLoop);
            } else {
                game.updateUI();
            }
        }
        
        window.addEventListener('keydown', (e) => {
            if (game.gameState === 'playing') {
                if (e.code === 'Space' && game.onGround) {
                    game.playerVelY = game.jumpStrength;
                    game.onGround = false;
                }
            } else if (game.gameState === 'menu' || game.gameState === 'gameOver') {
                if (e.code === 'Enter') {
                    game.start();
                }
            }
        });
 
        // Initial render
        game.updateUI();
 
    </script>
</body>
</html>
