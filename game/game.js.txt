const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

let gameRunning = true;
let score = 0;
let gameSpeed = 3;
let frameCount = 0;
let coinCount = 0;
const TARGET_COINS = 10;
let timeLeft = 60;

// プレイヤー
const player = {
    x: 100,
    y: 280,
    width: 40,
    height: 40,
    baseWidth: 40,
    baseHeight: 40,
    velocityY: 0,
    jumping: false,
    gravity: 0.6,
    jumpPower: -12,
    jumpCount: 0,
    maxJumps: 2,
    isPowerUp: false,
    powerUpTime: 0
};

// 障害物
let obstacles = [];
let coins = [];
let powerUpItems = [];
let enemies = [];

// 地面の位置
const groundY = 320;

// 雲
let clouds = [
    {x: 100, y: 50, speed: 0.3},
    {x: 400, y: 80, speed: 0.4},
    {x: 650, y: 40, speed: 0.2}
];

function drawPlayer() {
    // パワーアップ時のオーラ
    if (player.isPowerUp) {
        ctx.fillStyle = 'rgba(255, 215, 0, 0.3)';
        ctx.fillRect(player.x - 10, player.y - 10, player.width + 20, player.height + 20);
    }
    
    // プレイヤー（キャラクター風）
    ctx.fillStyle = player.isPowerUp ? '#FFD700' : '#FF6B6B';
    ctx.fillRect(player.x, player.y, player.width, player.height);
    
    // 目
    ctx.fillStyle = 'white';
    const eyeSize = player.width / 4;
    ctx.fillRect(player.x + player.width * 0.2, player.y + player.height * 0.25, eyeSize, eyeSize);
    ctx.fillRect(player.x + player.width * 0.55, player.y + player.height * 0.25, eyeSize, eyeSize);
    
    ctx.fillStyle = 'black';
    const pupilSize = eyeSize * 0.4;
    ctx.fillRect(player.x + player.width * 0.3, player.y + player.height * 0.35, pupilSize, pupilSize);
    ctx.fillRect(player.x + player.width * 0.65, player.y + player.height * 0.35, pupilSize, pupilSize);
    
    // 口
    ctx.fillStyle = 'black';
    ctx.fillRect(player.x + player.width * 0.375, player.y + player.height * 0.7, player.width * 0.25, player.height * 0.075);
}

function drawObstacle(obs) {
    if (obs.type === 'cactus') {
        // サボテン
        ctx.fillStyle = '#2D5016';
        ctx.fillRect(obs.x, obs.y, obs.width, obs.height);
        ctx.fillRect(obs.x - 8, obs.y + 10, 8, 10);
        ctx.fillRect(obs.x + obs.width, obs.y + 10, 8, 10);
    } else if (obs.type === 'rock') {
        // 岩
        ctx.fillStyle = '#666666';
        ctx.beginPath();
        ctx.moveTo(obs.x + obs.width / 2, obs.y);
        ctx.lineTo(obs.x + obs.width, obs.y + obs.height);
        ctx.lineTo(obs.x, obs.y + obs.height);
        ctx.closePath();
        ctx.fill();
        
        ctx.fillStyle = '#888888';
        ctx.beginPath();
        ctx.arc(obs.x + obs.width / 2, obs.y + obs.height / 2, 8, 0, Math.PI * 2);
        ctx.fill();
    } else if (obs.type === 'spike') {
        // トゲトゲ
        ctx.fillStyle = '#8B0000';
        for (let i = 0; i < 3; i++) {
            ctx.beginPath();
            ctx.moveTo(obs.x + i * 15, obs.y + obs.height);
            ctx.lineTo(obs.x + i * 15 + 7.5, obs.y);
            ctx.lineTo(obs.x + i * 15 + 15, obs.y + obs.height);
            ctx.closePath();
            ctx.fill();
        }
    } else if (obs.type === 'bird') {
        // 鳥（空中の障害物）
        ctx.fillStyle = '#FF6347';
        ctx.beginPath();
        ctx.ellipse(obs.x + 20, obs.y + 15, 15, 10, 0, 0, Math.PI * 2);
        ctx.fill();
        
        // 翼
        const wingOffset = Math.sin(frameCount * 0.2) * 5;
        ctx.fillStyle = '#FF4500';
        ctx.beginPath();
        ctx.ellipse(obs.x + 5, obs.y + 15 + wingOffset, 10, 5, -0.5, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.ellipse(obs.x + 35, obs.y + 15 - wingOffset, 10, 5, 0.5, 0, Math.PI * 2);
        ctx.fill();
        
        // 目
        ctx.fillStyle = 'white';
        ctx.beginPath();
        ctx.arc(obs.x + 25, obs.y + 12, 3, 0, Math.PI * 2);
        ctx.fill();
        ctx.fillStyle = 'black';
        ctx.beginPath();
        ctx.arc(obs.x + 26, obs.y + 12, 1.5, 0, Math.PI * 2);
        ctx.fill();
    }
}

function drawCoin(coin) {
    ctx.fillStyle = '#FFD700';
    ctx.beginPath();
    ctx.arc(coin.x + 15, coin.y + 15, 12, 0, Math.PI * 2);
    ctx.fill();
    
    ctx.fillStyle = '#FFA500';
    ctx.beginPath();
    ctx.arc(coin.x + 15, coin.y + 15, 8, 0, Math.PI * 2);
    ctx.fill();
}

function drawPowerUpItem(item) {
    // アイテムの輝き
    ctx.fillStyle = 'rgba(138, 43, 226, 0.3)';
    ctx.beginPath();
    ctx.arc(item.x + 15, item.y + 15, 18 + Math.sin(frameCount * 0.1) * 3, 0, Math.PI * 2);
    ctx.fill();
    
    // アイテム本体
    ctx.fillStyle = '#8A2BE2';
    ctx.beginPath();
    ctx.arc(item.x + 15, item.y + 15, 15, 0, Math.PI * 2);
    ctx.fill();
    
    // 星マーク
    ctx.fillStyle = 'white';
    ctx.font = 'bold 20px Arial';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText('⭐', item.x + 15, item.y + 15);
}

function drawEnemy(enemy) {
    if (enemy.type === 'walker') {
        // 歩く敵（スライム風）
        ctx.fillStyle = '#32CD32';
        ctx.beginPath();
        ctx.ellipse(enemy.x + enemy.width / 2, enemy.y + enemy.height / 2, enemy.width / 2, enemy.height / 2, 0, 0, Math.PI * 2);
        ctx.fill();
        
        // 目
        ctx.fillStyle = 'white';
        ctx.beginPath();
        ctx.arc(enemy.x + 15, enemy.y + 15, 5, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.arc(enemy.x + 30, enemy.y + 15, 5, 0, Math.PI * 2);
        ctx.fill();
        
        ctx.fillStyle = 'black';
        ctx.beginPath();
        ctx.arc(enemy.x + 15, enemy.y + 15, 2, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.arc(enemy.x + 30, enemy.y + 15, 2, 0, Math.PI * 2);
        ctx.fill();
    } else if (enemy.type === 'flyer') {
        // 飛ぶ敵（コウモリ風）
        const wingOffset = Math.sin(frameCount * 0.2 + enemy.id) * 5;
        
        ctx.fillStyle = '#8B008B';
        ctx.beginPath();
        ctx.ellipse(enemy.x + 20, enemy.y + 15, 15, 10, 0, 0, Math.PI * 2);
        ctx.fill();
        
        // 翼
        ctx.fillStyle = '#9400D3';
        ctx.beginPath();
        ctx.ellipse(enemy.x + 5, enemy.y + 15 + wingOffset, 10, 8, -0.3, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.ellipse(enemy.x + 35, enemy.y + 15 - wingOffset, 10, 8, 0.3, 0, Math.PI * 2);
        ctx.fill();
        
        // 耳
        ctx.fillStyle = '#8B008B';
        ctx.beginPath();
        ctx.moveTo(enemy.x + 15, enemy.y + 5);
        ctx.lineTo(enemy.x + 12, enemy.y);
        ctx.lineTo(enemy.x + 18, enemy.y + 8);
        ctx.fill();
        ctx.beginPath();
        ctx.moveTo(enemy.x + 25, enemy.y + 5);
        ctx.lineTo(enemy.x + 28, enemy.y);
        ctx.lineTo(enemy.x + 22, enemy.y + 8);
        ctx.fill();
        
        // 目
        ctx.fillStyle = 'red';
        ctx.beginPath();
        ctx.arc(enemy.x + 15, enemy.y + 15, 2, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.arc(enemy.x + 25, enemy.y + 15, 2, 0, Math.PI * 2);
        ctx.fill();
    }
}

function drawCloud(cloud) {
    ctx.fillStyle = 'rgba(255, 255, 255, 0.8)';
    ctx.beginPath();
    ctx.arc(cloud.x, cloud.y, 20, 0, Math.PI * 2);
    ctx.arc(cloud.x + 25, cloud.y, 25, 0, Math.PI * 2);
    ctx.arc(cloud.x + 50, cloud.y, 20, 0, Math.PI * 2);
    ctx.fill();
}

function drawGround() {
    // 地面
    ctx.fillStyle = '#8B4513';
    ctx.fillRect(0, groundY, canvas.width, canvas.height - groundY);
    
    // 草
    ctx.fillStyle = '#90EE90';
    for (let i = 0; i < canvas.width; i += 40) {
        ctx.fillRect(i, groundY - 5, 30, 5);
    }
}

function jump() {
    if (player.jumpCount < player.maxJumps && gameRunning) {
        player.velocityY = player.jumpPower;
        player.jumping = true;
        player.jumpCount++;
    }
}

function updatePlayer() {
    // パワーアップタイマー
    if (player.isPowerUp) {
        player.powerUpTime--;
        if (player.powerUpTime <= 0) {
            player.isPowerUp = false;
            player.width = player.baseWidth;
            player.height = player.baseHeight;
        }
    }
    
    player.velocityY += player.gravity;
    player.y += player.velocityY;

    if (player.y >= groundY - player.height) {
        player.y = groundY - player.height;
        player.velocityY = 0;
        player.jumping = false;
        player.jumpCount = 0;
    }
}

function spawnObstacle() {
    if (frameCount % 120 === 0) {
        const obstacleTypes = [
            { type: 'cactus', width: 40, height: 50, y: groundY - 50 },
            { type: 'rock', width: 45, height: 40, y: groundY - 40 },
            { type: 'spike', width: 45, height: 35, y: groundY - 35 },
            { type: 'bird', width: 40, height: 30, y: groundY - 120 }
        ];
        
        const randomObstacle = obstacleTypes[Math.floor(Math.random() * obstacleTypes.length)];
        
        obstacles.push({
            x: canvas.width,
            y: randomObstacle.y,
            width: randomObstacle.width,
            height: randomObstacle.height,
            type: randomObstacle.type
        });
    }
}

function spawnCoin() {
    if (frameCount % 150 === 0) {
        coins.push({
            x: canvas.width,
            y: groundY - 100 - Math.random() * 80,
            width: 30,
            height: 30,
            collected: false
        });
    }
}

function spawnPowerUpItem() {
    if (frameCount % 400 === 0) {
        powerUpItems.push({
            x: canvas.width,
            y: groundY - 120 - Math.random() * 60,
            width: 30,
            height: 30,
            collected: false
        });
    }
}

function spawnEnemy() {
    if (frameCount % 180 === 0) {
        const enemyTypes = [
            { 
                type: 'walker', 
                width: 45, 
                height: 35, 
                y: groundY - 35,
                speed: gameSpeed * 0.8
            },
            { 
                type: 'flyer', 
                width: 40, 
                height: 30, 
                y: groundY - 140,
                speed: gameSpeed * 1.1,
                movePattern: 'wave'
            }
        ];
        
        const randomEnemy = enemyTypes[Math.floor(Math.random() * enemyTypes.length)];
        
        enemies.push({
            x: canvas.width,
            y: randomEnemy.y,
            baseY: randomEnemy.y,
            width: randomEnemy.width,
            height: randomEnemy.height,
            type: randomEnemy.type,
            speed: randomEnemy.speed,
            movePattern: randomEnemy.movePattern || 'straight',
            id: Math.random()
        });
    }
}

function updateObstacles() {
    obstacles.forEach((obs, index) => {
        obs.x -= gameSpeed;
        if (obs.x + obs.width < 0) {
            obstacles.splice(index, 1);
            score += 10;
            document.getElementById('score').textContent = score;
        }
    });
}

function updateCoins() {
    coins.forEach((coin, index) => {
        coin.x -= gameSpeed;
        
        // コイン取得判定
        if (!coin.collected &&
            player.x < coin.x + coin.width &&
            player.x + player.width > coin.x &&
            player.y < coin.y + coin.height &&
            player.y + player.height > coin.y) {
            coin.collected = true;
            score += 50;
            coinCount++;
            document.getElementById('score').textContent = score;
            document.getElementById('coinCount').textContent = coinCount;
            
            if (coinCount >= TARGET_COINS) {
                setTimeout(() => gameOver(true), 500);
            }
        }
        
        if (coin.x + coin.width < 0 || coin.collected) {
            coins.splice(index, 1);
        }
    });
}

function updatePowerUpItems() {
    powerUpItems.forEach((item, index) => {
        item.x -= gameSpeed;
        
        // アイテム取得判定
        if (!item.collected &&
            player.x < item.x + item.width &&
            player.x + player.width > item.x &&
            player.y < item.y + item.height &&
            player.y + player.height > item.y) {
            item.collected = true;
            player.isPowerUp = true;
            player.powerUpTime = 300; // 5秒間
            player.width = player.baseWidth * 1.5;
            player.height = player.baseHeight * 1.5;
        }
        
        if (item.x + item.width < 0 || item.collected) {
            powerUpItems.splice(index, 1);
        }
    });
}

function updateEnemies() {
    enemies.forEach((enemy, index) => {
        enemy.x -= enemy.speed;
        
        // 飛ぶ敵の波動移動
        if (enemy.movePattern === 'wave') {
            enemy.y = enemy.baseY + Math.sin(frameCount * 0.05 + enemy.id * 10) * 20;
        }
        
        if (enemy.x + enemy.width < 0) {
            enemies.splice(index, 1);
            score += 10;
            document.getElementById('score').textContent = score;
        }
    });
}

function updateClouds() {
    clouds.forEach(cloud => {
        cloud.x -= cloud.speed;
        if (cloud.x < -60) {
            cloud.x = canvas.width + 60;
        }
    });
}

function checkCollision() {
    // パワーアップ中は無敵
    if (player.isPowerUp) return;
    
    // 障害物との衝突
    for (let obs of obstacles) {
        if (player.x < obs.x + obs.width &&
            player.x + player.width > obs.x &&
            player.y < obs.y + obs.height &&
            player.y + player.height > obs.y) {
            gameOver(false);
            return;
        }
    }
    
    // 敵との衝突
    for (let enemy of enemies) {
        if (player.x < enemy.x + enemy.width &&
            player.x + player.width > enemy.x &&
            player.y < enemy.y + enemy.height &&
            player.y + player.height > enemy.y) {
            gameOver(false);
            return;
        }
    }
}

function gameOver(isWin) {
    gameRunning = false;
    const gameOverDiv = document.getElementById('gameOverText');
    if (isWin) {
        gameOverDiv.innerHTML = `🎉 クリア！おめでとう！ 🎉<br>最終スコア: ${score}<br>コイン: ${coinCount} / ${TARGET_COINS}<br><button onclick="restartGame()">もう一度プレイ</button>`;
    } else {
        gameOverDiv.innerHTML = `ゲームオーバー！<br>スコア: ${score}<br>コイン: ${coinCount} / ${TARGET_COINS}<br><button onclick="restartGame()">もう一度プレイ</button>`;
    }
    gameOverDiv.style.display = 'block';
}

function restartGame() {
    gameRunning = true;
    score = 0;
    coinCount = 0;
    timeLeft = 60;
    gameSpeed = 3;
    player.y = 280;
    player.velocityY = 0;
    player.jumping = false;
    player.isPowerUp = false;
    player.powerUpTime = 0;
    player.width = player.baseWidth;
    player.height = player.baseHeight;
    obstacles = [];
    coins = [];
    powerUpItems = [];
    enemies = [];
    frameCount = 0;
    document.getElementById('score').textContent = score;
    document.getElementById('coinCount').textContent = coinCount;
    document.getElementById('timeLeft').textContent = timeLeft;
    document.getElementById('gameOverText').style.display = 'none';
    gameLoop();
}

function gameLoop() {
    if (!gameRunning) return;

    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // 背景
    ctx.fillStyle = '#87CEEB';
    ctx.fillRect(0, 0, canvas.width, groundY);

    // 雲
    clouds.forEach(cloud => drawCloud(cloud));
    updateClouds();

    // 地面
    drawGround();

    // 障害物を先に描画
    spawnObstacle();
    updateObstacles();
    obstacles.forEach(obs => drawObstacle(obs));

    // 敵
    spawnEnemy();
    updateEnemies();
    enemies.forEach(enemy => drawEnemy(enemy));

    // コイン
    spawnCoin();
    updateCoins();
    coins.forEach(coin => {
        if (!coin.collected) drawCoin(coin);
    });

    // パワーアップアイテム
    spawnPowerUpItem();
    updatePowerUpItems();
    powerUpItems.forEach(item => {
        if (!item.collected) drawPowerUpItem(item);
    });

    // プレイヤーを最後に描画（一番手前）
    drawPlayer();
    updatePlayer();

    // 衝突判定
    checkCollision();

    frameCount++;
    
    // タイマー更新（60フレーム = 1秒）
    if (frameCount % 60 === 0 && timeLeft > 0) {
        timeLeft--;
        document.getElementById('timeLeft').textContent = timeLeft;
        
        if (timeLeft === 0 && coinCount < TARGET_COINS) {
            gameOver(false);
        }
    }
    
    // 難易度上昇
    if (frameCount % 300 === 0) {
        gameSpeed += 0.3;
    }

    requestAnimationFrame(gameLoop);
}

// キーボード入力
document.addEventListener('keydown', (e) => {
    if (e.code === 'Space') {
        e.preventDefault();
        jump();
    }
});

// タッチ/クリック入力
canvas.addEventListener('click', jump);
canvas.addEventListener('touchstart', (e) => {
    e.preventDefault();
    jump();
});

// ゲーム開始
gameLoop();