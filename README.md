<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Dino Jump - Fullscreen</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            user-select: none;
            -webkit-user-select: none;
            touch-action: none;
        }
        html, body {
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #e0e0e0;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        canvas {
            display: block;
            width: 100vw;
            height: 100vh;
            background: linear-gradient(to bottom, #87CEEB, #E0F6FF);
        }
        #ui-container {
            position: absolute;
            top: 20px;
            left: 20px;
            right: 20px;
            display: flex;
            justify-content: space-between;
            font-size: 24px;
            font-weight: bold;
            color: #333;
            text-shadow: 1px 1px 2px rgba(255,255,255,0.8);
            pointer-events: none;
        }
        #game-over-screen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.5);
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            color: white;
            opacity: 0;
            visibility: hidden;
            transition: opacity 0.3s ease;
        }
        #game-over-screen.visible {
            opacity: 1;
            visibility: visible;
        }
        #game-over-screen h1 {
            font-size: 48px;
            margin-bottom: 20px;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.6);
        }
        #restart-btn {
            padding: 15px 40px;
            font-size: 22px;
            font-weight: bold;
            color: white;
            background-color: #ff4757;
            border: none;
            border-radius: 30px;
            cursor: pointer;
            box-shadow: 0 5px 15px rgba(0,0,0,0.3);
            transition: transform 0.1s, background-color 0.2s;
        }
        #restart-btn:active {
            transform: scale(0.95);
            background-color: #ff6b81;
        }
    </style>
</head>
<body>

    <div id="ui-container">
        <div id="score-display">Очки: 0</div>
        <div id="highscore-display">Рекорд: 0</div>
    </div>

    <div id="game-over-screen">
        <h1>ИГРА ОКОНЧЕНА</h1>
        <button id="restart-btn">Играть снова</button>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreDisplay = document.getElementById('score-display');
        const highscoreDisplay = document.getElementById('highscore-display');
        const gameOverScreen = document.getElementById('game-over-screen');
        const restartBtn = document.getElementById('restart-btn');

        // Изображения
        const playerImg = new Image();
        playerImg.src = 'image_84_OvG.png';
        let usePlayerImg = false;
        playerImg.onload = () => { usePlayerImg = true; };

        const zombieImg = new Image();
        zombieImg.src = 'image_Nu4GjL.png';
        let useZombieImg = false;
        zombieImg.onload = () => { useZombieImg = true; };

        // Игровые переменные
        let gameState = 'playing'; // playing, over
        let score = 0;
        let highscore = localStorage.getItem('dino_highscore') ? parseInt(localStorage.getItem('dino_highscore')) : 0;
        highscoreDisplay.textContent = 'Рекорд: ' + highscore;

        let baseSpeed = 4; 
        let currentSpeed = baseSpeed;
        let gravity = 0.4; 
        let floorY = 0;

        // Размеры подгоняются под экран динамически
        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
            floorY = canvas.height * 0.75;
            
            // Корректируем позиции персонажей при изменении размера экрана
            if (player) {
                player.y = floorY - player.height;
                player.groundY = floorY - player.height;
            }
        }

        // Класс игрока
        class Player {
            constructor() {
                this.width = 60;
                this.height = 70;
                this.x = 80;
                this.y = 0;
                this.vy = 0;
                this.groundY = 0;
                this.isJumping = false;
            }

            update() {
                this.vy += gravity;
                this.y += this.vy;

                if (this.y >= this.groundY) {
                    this.y = this.groundY;
                    this.vy = 0;
                    this.isJumping = false;
                }
            }

            jump() {
                if (!this.isJumping) {
                    this.vy = -11; // Мягкий затяжной прыжок
                    this.isJumping = true;
                }
            }

            draw() {
                if (usePlayerImg) {
                    ctx.drawImage(playerImg, this.x, this.y, this.width, this.height);
                } else {
                    ctx.font = '55px serif';
                    ctx.fillText('🏃', this.x, this.y + this.height - 5);
                }
            }
        }

        // Класс зомби
        class Zombie {
            constructor() {
                this.width = 50;
                this.height = 65;
                this.x = canvas.width + Math.random() * 200;
                this.y = floorY - this.height;
                this.passed = false;
            }

            update() {
                this.x -= currentSpeed;
            }

            draw() {
                if (useZombieImg) {
                    ctx.drawImage(zombieImg, this.x, this.y, this.width, this.height);
                } else {
                    ctx.font = '50px serif';
                    ctx.fillText('🧟', this.x, this.y + this.height - 5);
                }
            }
        }

        let player;
        let zombies = [];
        let spawnTimer = 0;
        let nextSpawnTime = 120; // Дистанция между зомби

        function initGame() {
            resizeCanvas();
            player = new Player();
            player.y = floorY - player.height;
            player.groundY = floorY - player.height;
            zombies = [];
            score = 0;
            currentSpeed = baseSpeed;
            scoreDisplay.textContent = 'Очки: ' + score;
            gameState = 'playing';
            gameOverScreen.classList.remove('visible');
            spawnTimer = 0;
            nextSpawnTime = 120;
        }

        function spawnZombie() {
            zombies.push(new Zombie());
            // Случайный интервал, но не слишком близко (баланс легкой игры)
            nextSpawnTime = 100 + Math.random() * 120; 
        }

        function checkCollision(rect1, rect2) {
            // Небольшой отступ для хитбоксов, чтобы игра была честнее
            const paddingX = 12;
            const paddingY = 8;
            return (
                rect1.x + paddingX < rect2.x + rect2.width - paddingX &&
                rect1.x + rect1.width - paddingX > rect2.x + paddingX &&
                rect1.y + paddingY < rect2.y + rect2.height - paddingY &&
                rect1.y + rect1.height - paddingY > rect2.y + paddingY
            );
        }

        function gameOver() {
            gameState = 'over';
            gameOverScreen.classList.add('visible');
            if (score > highscore) {
                highscore = score;
                localStorage.setItem('dino_highscore', highscore);
                highscoreDisplay.textContent = 'Рекорд: ' + highscore;
            }
        }

        // Главный цикл игры
        function gameLoop() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Отрисовка земли
            ctx.fillStyle = '#7a6b58';
            ctx.fillRect(0, floorY, canvas.width, canvas.height - floorY);
            ctx.fillStyle = '#554231';
            ctx.fillRect(0, floorY, canvas.width, 10);

            if (gameState === 'playing') {
                player.update();

                // Появление зомби
                spawnTimer++;
                if (spawnTimer >= nextSpawnTime) {
                    spawnZombie();
                    spawnTimer = 0;
                }

                // Управление сложностью (очень медленное нарастание)
                currentSpeed = baseSpeed + (score * 0.05);

                // Обновление зомби
                for (let i = zombies.length - 1; i >= 0; i--) {
                    zombies[i].update();

                    if (checkCollision(player, zombies[i])) {
                        gameOver();
                    }

                    if (!zombies[i].passed && zombies[i].x + zombies[i].width < player.x) {
                        zombies[i].passed = true;
                        score++;
                        scoreDisplay.textContent = 'Очки: ' + score;
                    }

                    // Удаление вышедших за экран зомби
                    if (zombies[i].x + zombies[i].width < 0) {
                        zombies.splice(i, 1);
                    }
                }
            }

            // Отрисовка объектов
            player.draw();
            zombies.forEach(zombie => zombie.draw());

            requestAnimationFrame(gameLoop);
        }

        // Обработка ввода (Прыжок)
        function handleJump(e) {
            if (gameState === 'playing') {
                player.jump();
                if (e.cancelable) e.preventDefault();
            }
        }

        // Слушатели событий
        window.addEventListener('keydown', (e) => {
            if (e.code === 'Space' || e.code === 'ArrowUp') {
                handleJump(e);
            }
        });

        // Прыжок по любому клику/тачу на весь экран
        window.addEventListener('touchstart', handleJump, { passive: false });
        window.addEventListener('mousedown', (e) => {
            if (e.target !== restartBtn) {
                handleJump(e);
            }
        });

        restartBtn.addEventListener('click', (e) => {
            e.stopPropagation();
            initGame();
        });

        window.addEventListener('resize', resizeCanvas);

        // Старт игры
        initGame();
        gameLoop();
    </script>
</body>
</html>
