<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Galway Frogger</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #222;
            color: #fff;
            font-family: 'Courier New', Courier, monospace;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            touch-action: none; /* Prevents scroll on touch drag */
        }
        
        #game-wrapper {
            background: #333;
            padding: 15px;
            border-radius: 10px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            text-align: center;
            max-width: 100%;
        }

        #header {
            margin-bottom: 10px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            font-weight: bold;
        }

        #level-display {
            font-size: 1.5rem;
            color: #00ffcc;
        }

        #threat-display {
            font-size: 1rem;
            color: #ff3366;
            animation: pulse 1s infinite;
        }

        @keyframes pulse {
            0% { opacity: 1; }
            50% { opacity: 0.5; }
            100% { opacity: 1; }
        }

        canvas {
            background-color: #555;
            border: 4px solid #fff;
            border-radius: 4px;
            max-width: 100%;
            height: auto;
            image-rendering: pixelated;
        }

        /* Mobile Controls */
        #controls {
            display: grid;
            grid-template-columns: 60px 60px 60px;
            grid-template-rows: 60px 60px;
            gap: 10px;
            margin-top: 20px;
            justify-content: center;
        }

        .btn {
            background: rgba(255, 255, 255, 0.2);
            border: 2px solid #fff;
            border-radius: 10px;
            color: white;
            font-size: 24px;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            user-select: none;
        }

        .btn:active {
            background: rgba(255, 255, 255, 0.5);
        }

        #up { grid-column: 2; grid-row: 1; }
        #left { grid-column: 1; grid-row: 2; }
        #down { grid-column: 2; grid-row: 2; }
        #right { grid-column: 3; grid-row: 2; }

        #message-overlay {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.9);
            padding: 20px 40px;
            border: 2px solid #ff3366;
            border-radius: 10px;
            text-align: center;
            display: none;
            z-index: 10;
        }

        #message-overlay h2 {
            margin: 0 0 10px 0;
            color: #ff3366;
        }
    </style>
</head>
<body>

    <div id="game-wrapper">
        <div id="header">
            <div id="level-display">Level: 1</div>
            <div id="threat-display">Threat: N59 ON A SUNDAY</div>
        </div>
        
        <canvas id="gameCanvas" width="500" height="500"></canvas>
        
        <div id="controls">
            <div class="btn" id="up">⬆️</div>
            <div class="btn" id="left">⬅️</div>
            <div class="btn" id="down">⬇️</div>
            <div class="btn" id="right">➡️</div>
        </div>
    </div>

    <div id="message-overlay">
        <h2 id="msg-title">LEVEL UP!</h2>
        <p id="msg-desc">Difficulty increasing...</p>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const levelDisplay = document.getElementById('level-display');
        const threatDisplay = document.getElementById('threat-display');
        const messageOverlay = document.getElementById('message-overlay');
        const msgDesc = document.getElementById('msg-desc');

        // Game Constants
        const GRID_SIZE = 50;
        const ROWS = 10;
        const COLS = 10;
        
        // Threat texts to pretend the game is getting harder
        const threatLevels = [
            "N59 ON A SUNDAY", 
            "SALTHILL PROM PACE", 
            "SLIGHT DRIZZLE", 
            "RACE WEEK BUILDUP", 
            "EYRE SQUARE GRIDLOCK", 
            "CHRISTMAS MARKET CHAOS", 
            "GALWAY KINLAY HOSTEL RUSH",
            "SHOP STREET BUSKER BLOCKADE"
        ];

        const destinationsData = [
            {
                name: "--- UNIVERSITY OF GALWAY ---",
                art: [
                    "☁️   🎓   🏛️   🎓   ☁️",
                    "🌳   🧱   🧱   🧱   🌳"
                ]
            },
            {
                name: "--- EYRE SQUARE ---",
                art: [
                    "🕊️   ☁️   ⛩️   ☁️   🕊️",
                    "🌳   🌷   ⛲   🌷   🌳"
                ]
            },
            {
                name: "--- SPANISH ARCH ---",
                art: [
                    "🐦   ☁️   🏰   ☁️   🐦",
                    "🌊   🌊   🧱   🌊   🌊"
                ]
            }
        ];

        let level = 1;

        // Player Object
        const player = {
            x: 4 * GRID_SIZE,
            y: 9 * GRID_SIZE,
            size: GRID_SIZE,
            emoji: '☔' // Galway weather appropriate!
        };

        // Tractors - The "Threats"
        let tractors = [];
        const TRACTOR_SPEED = 0.08; // Extremely slow. Never changes.

        function initTractors() {
            tractors = [];
            const lanes = [2, 3, 4, 5, 6, 7];
            
            lanes.forEach(lane => {
                const startX = -(Math.random() * 200 + 50); // Stagger starting positions
                const convoyLength = Math.floor(Math.random() * 3) + 3; // 3 to 5 tractors per convoy
                
                for (let i = 0; i < convoyLength; i++) {
                    // Place tractors directly behind each other (55 pixels apart to allow slight gaps)
                    tractors.push({ x: startX - (i * 55), y: lane * GRID_SIZE });
                }
            });
        }

        initTractors();

        // Input handling
        function movePlayer(dx, dy) {
            player.x += dx * GRID_SIZE;
            player.y += dy * GRID_SIZE;

            // Boundaries
            if (player.x < 0) player.x = 0;
            if (player.x >= canvas.width) player.x = canvas.width - GRID_SIZE;
            if (player.y >= canvas.height) player.y = canvas.height - GRID_SIZE;

            // Win Condition (Reached the top row)
            if (player.y < GRID_SIZE) {
                levelUp();
            }
        }

        // Keyboard Controls
        window.addEventListener('keydown', (e) => {
            switch(e.key) {
                case 'ArrowUp': case 'w': movePlayer(0, -1); break;
                case 'ArrowDown': case 's': movePlayer(0, 1); break;
                case 'ArrowLeft': case 'a': movePlayer(-1, 0); break;
                case 'ArrowRight': case 'd': movePlayer(1, 0); break;
            }
        });

        // Touch / Click Controls
        document.getElementById('up').addEventListener('pointerdown', () => movePlayer(0, -1));
        document.getElementById('down').addEventListener('pointerdown', () => movePlayer(0, 1));
        document.getElementById('left').addEventListener('pointerdown', () => movePlayer(-1, 0));
        document.getElementById('right').addEventListener('pointerdown', () => movePlayer(1, 0));

        function levelUp() {
            level++;
            
            // Reset player
            player.x = 4 * GRID_SIZE;
            player.y = 9 * GRID_SIZE;

            // "Increase" difficulty text (but speed stays the same)
            levelDisplay.innerText = `Level: ${level}`;
            const threatIndex = Math.min(level - 1, threatLevels.length - 1);
            threatDisplay.innerText = `Threat: ${threatLevels[threatIndex]}`;
            
            // Show scary message only every 5 levels
            if (level % 5 === 0) {
                msgDesc.innerText = `Warning: Reached Level ${level}!\nTraffic on the N84 is backing up. Things are about to get intense! (they aren't)`;
                messageOverlay.style.display = 'block';
                
                setTimeout(() => {
                    messageOverlay.style.display = 'none';
                }, 3500);
            }

            // Re-initialize tractors back to the left so the screen isn't immediately blocked
            initTractors();
        }

        function resetGame() {
            level = 1;
            levelDisplay.innerText = `Level: ${level}`;
            threatDisplay.innerText = `Threat: ${threatLevels[0]}`;
            player.x = 4 * GRID_SIZE;
            player.y = 9 * GRID_SIZE;
            initTractors();
        }

        // Main Game Loop
        function gameLoop() {
            // 1. Clear & Draw Background
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Draw Safe Zones (Grass)
            ctx.fillStyle = '#4CAF50'; 
            ctx.fillRect(0, 0, canvas.width, 2 * GRID_SIZE); // Top Safe Zone
            ctx.fillRect(0, 8 * GRID_SIZE, canvas.width, 2 * GRID_SIZE); // Bottom Safe Zone

            // Draw Destination Text and Emoji Art in Top Safe Zone
            ctx.fillStyle = '#FFFFFF';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            const currentDest = destinationsData[(level - 1) % destinationsData.length];
            
            // Draw Title
            ctx.font = 'bold 16px Courier New';
            ctx.fillText(currentDest.name, canvas.width / 2, 20);
            
            // Draw Emoji Art
            ctx.font = '28px Arial'; // Larger font for better emoji rendering
            ctx.fillText(currentDest.art[0], canvas.width / 2, 55);
            ctx.fillText(currentDest.art[1], canvas.width / 2, 85);

            // Draw Road
            ctx.fillStyle = '#333333';
            ctx.fillRect(0, 2 * GRID_SIZE, canvas.width, 6 * GRID_SIZE);
            
            // Road lines
            ctx.strokeStyle = '#FFFFFF';
            ctx.setLineDash([20, 20]);
            ctx.lineWidth = 2;
            for(let i = 3; i < 8; i++) {
                ctx.beginPath();
                ctx.moveTo(0, i * GRID_SIZE);
                ctx.lineTo(canvas.width, i * GRID_SIZE);
                ctx.stroke();
            }
            ctx.setLineDash([]);

            // 2. Update and Draw Tractors
            ctx.font = '35px Arial';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            
            for (let t of tractors) {
                // Move tractor at a disgustingly slow pace
                t.x += TRACTOR_SPEED; 

                // Loop back to left side if it goes off screen right
                if (t.x > canvas.width + GRID_SIZE) {
                    t.x = -GRID_SIZE;
                }

                // Draw Tractor
                ctx.fillText('🚜', t.x + GRID_SIZE/2, t.y + GRID_SIZE/2);

                // Collision Detection (Very forgiving hitbox)
                const hitThreshold = 30; // pixels
                if (Math.abs(player.x - t.x) < hitThreshold && Math.abs(player.y - t.y) < hitThreshold) {
                    resetGame();
                }
            }

            // 3. Draw Player
            ctx.fillText(player.emoji, player.x + GRID_SIZE/2, player.y + GRID_SIZE/2);

            // Loop
            requestAnimationFrame(gameLoop);
        }

        // Start the game
        requestAnimationFrame(gameLoop);

    </script>
</body>
</html>
