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
            background-color: #fff; /* This ensures the game background is white */
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
                <p>Press ENTER or Tap to Start</p>
                <p id="menu-high-score">High Score: 0</p>
            </div>
            <!-- Game Over Screen -->
            <div id="game-over-screen" class="hidden">
                <h1>Game Over</h1>
                <p id="final-score">Your Score: 0</p>
                <p id="game-over-high-score">High Score: 0</p>
                <p>Press ENTER or Tap to Restart</p>
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
                        size = 35;
                        this.width = size;
                        this.height = size;
                        break;
                    case 'rectangle':
                        width = 50;
                        height = 25;
                        this.width = width;
                        this.height = height;
                        break;
                    case 'triangle':
                        size = 40;
                        this.width = size;
                        this.height = size;
                        break;
                }
            }

            update(scrollSpeed, groundYFunc) {
                this.x -= scrollSpeed;
                // **CRITICAL FIX**: The y-coordinate must be calculated from the obstacle's CENTER for correct rotation alignment.
                this.y = groundYFunc(this.x + this.width / 2);
            }

            draw(slopeAngleInRadians) {
                ctx.save();
                ctx.fillStyle = '#ff3232';
                // Translate to the center of the obstacle's base to rotate around it
                ctx.translate(this.x + this.width / 2, this.y);
                ctx.rotate(slopeAngleInRadians);

                if (this.shapeType === 'square' || this.shapeType === 'rectangle') {
                    // Draw relative to the new, rotated origin
                    ctx.fillRect(-this.width / 2, -this.height, this.width, this.height);
                } else if (this.shapeType === 'triangle') {
                    ctx.beginPath();
                    ctx.moveTo(0, -this.height); // Top point
                    ctx.lineTo(this.width / 2, 0); // Bottom right
                    ctx.lineTo(-this.width / 2, 0); // Bottom left
                    ctx.closePath();
                    ctx.fill();
                }
                ctx.restore();
            }
        }

        class Game {
            constructor() {
                this.width = canvas.width;
                this.height = canvas.height;
                
                this.slopeAngleInRadians = 30 * Math.PI / 180;
                this.groundYStart = 100;
                this.baseScrollSpeed = 4;
                this.maxScrollSpeed = 10;
                
                this.playerX = 300;
                this.playerRadius = 20;
                this.gravity = 0.6;
                this.jumpStrength = -14;

                this.obstacleGap = 400; // Base distance

                this.highScore = this.loadHighScore();
                this.gameState = 'menu';
                this.init();
            }

            init() {
                this.playerY = this.getGroundY(this.playerX);
                this.playerVelY = 0;
                this.onGround = true;
                this.score = 0;
                this.scrollSpeed = this.baseScrollSpeed;
                this.obstacles = [];

                let lastX = this.width;
                for (let i = 0; i < 5; i++) {
                    const spawnX = lastX + this.obstacleGap + Math.random() * 500;
                    const spawnY = this.getGroundY(spawnX);
                    this.obstacles.push(new Obstacle(spawnX, spawnY));
                    lastX = spawnX;
                }
                this.updateUI();
            }
            
            getGroundY(xPos) {
                return this.groundYStart + xPos * Math.tan(this.slopeAngleInRadians);
            }

            loadHighScore() {
                try {
                    const score = localStorage.getItem('declineRunnerHighScore');
                    return score ? parseInt(score) : 0;
                } catch (e) {
                    console.error("Could not load high score.", e);
                    return 0;
                }
            }

            saveHighScore() {
                try {
                    localStorage.setItem('declineRunnerHighScore', this.highScore);
                } catch (e) {
                    console.error("Could not save high score.", e);
                }
            }

            start() {
                this.gameState = 'playing';
                this.init();
                gameLoop();
            }

            update() {
                if (this.gameState !== 'playing') return;

                this.scrollSpeed = Math.min(this.maxScrollSpeed, this.baseScrollSpeed + this.score / 5);

                this.playerVelY += this.gravity;
                this.playerY += this.playerVelY;

                const groundY = this.getGroundY(this.playerX);
                if (this.playerY >= groundY - this.playerRadius) {
                    this.playerY = groundY - this.playerRadius;
                    this.playerVelY = 0;
                    this.onGround = true;
                }

                this.obstacles.forEach(obstacle => {
                    obstacle.update(this.scrollSpeed, (x) => this.getGroundY(x));
                });

                if (this.obstacles.length > 0 && this.obstacles[0].x + this.obstacles[0].width < 0) {
                    this.obstacles.shift();
                    const lastX = this.obstacles[this.obstacles.length - 1].x;
                    const spawnX = lastX + this.obstacleGap + Math.random() * 500;
                    const spawnY = this.getGroundY(spawnX);
                    this.obstacles.push(new Obstacle(spawnX, spawnY));
                    this.score++;
                }

                // Accurate collision detection for rotated obstacles.
                for (const obstacle of this.obstacles) {
                    const rotationPointX = obstacle.x + obstacle.width / 2;
                    const rotationPointY = obstacle.y;
                    const relativePlayerX = this.playerX - rotationPointX;
                    const relativePlayerY = this.playerY - rotationPointY;

                    const angle = -this.slopeAngleInRadians;
                    const rotatedPlayerX = relativePlayerX * Math.cos(angle) - relativePlayerY * Math.sin(angle);
                    const rotatedPlayerY = relativePlayerX * Math.sin(angle) + relativePlayerY * Math.cos(angle);

                    const obstacleLeft = -obstacle.width / 2;
                    const obstacleRight = obstacle.width / 2;
                    const obstacleTop = -obstacle.height;
                    const obstacleBottom = 0;

                    let closestX = Math.max(obstacleLeft, Math.min(rotatedPlayerX, obstacleRight));
                    let closestY = Math.max(obstacleTop, Math.min(rotatedPlayerY, obstacleBottom));

                    const distanceX = rotatedPlayerX - closestX;
                    const distanceY = rotatedPlayerY - closestY;
                    const distanceSquared = (distanceX * distanceX) + (distanceY * distanceY);

                    if (distanceSquared < (this.playerRadius * this.playerRadius)) {
                        this.gameState = 'gameOver';
                        if (this.score > this.highScore) {
                            this.highScore = this.score;
                            this.saveHighScore();
                        }
                        break; // Exit loop after first collision
                    }
                }
                this.updateUI();
            }

            draw() {
                ctx.clearRect(0, 0, this.width, this.height);

                ctx.strokeStyle = '#333';
                ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.moveTo(0, this.getGroundY(0));
                ctx.lineTo(this.width, this.getGroundY(this.width));
                ctx.stroke();

                ctx.fillStyle = '#0064ff';
                ctx.beginPath();
                ctx.arc(this.playerX, this.playerY, this.playerRadius, 0, Math.PI * 2);
                ctx.fill();

                this.obstacles.forEach(obstacle => obstacle.draw(this.slopeAngleInRadians));
            }
            
            updateUI() {
                menuScreen.classList.toggle('hidden', this.gameState !== 'menu');
                gameOverScreen.classList.toggle('hidden', this.gameState !== 'gameOver');
                scoreDisplay.classList.toggle('hidden', this.gameState !== 'playing');

                if (this.gameState === 'menu') {
                    menuHighScore.textContent = `High Score: ${this.highScore}`;
                } else if (this.gameState === 'playing') {
                    currentScoreSpan.textContent = this.score;
                    gameplayHighScore.textContent = this.highScore;
                } else if (this.gameState === 'gameOver') {
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
        
        // --- Input Handling ---
        function handleInput() {
            if (game.gameState === 'playing') {
                if (game.onGround) {
                    game.playerVelY = game.jumpStrength;
                    game.onGround = false;
                }
            } else if (game.gameState === 'menu' || game.gameState === 'gameOver') {
                game.start();
            }
        }

        window.addEventListener('keydown', (e) => {
            if (e.code === 'Space' || e.code === 'Enter') {
                handleInput();
            }
        });
        
        // Add touch support
        window.addEventListener('touchstart', (e) => {
            e.preventDefault(); // Prevents screen from moving on tap
            handleInput();
        }, { passive: false });


        // Initial render
        game.updateUI();

    </script>
</body>
</html>
