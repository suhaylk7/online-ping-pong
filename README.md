<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Online Ping Pong Game</title>
    <style>
        body { background: #222; margin: 0; overflow: hidden; }
        canvas { display: block; margin: 0 auto; background: #111; }
        #info { color: #fff; text-align: center; font-family: sans-serif; margin-top: 10px; }
    </style>
</head>
<body>
    <canvas id="pong" width="800" height="500"></canvas>
    <div id="info">Connecting to server...</div>
    <script src="https://cdn.jsdelivr.net/npm/socket.io-client@4.7.5/dist/socket.io.min.js"></script>
    <script>
        // --- Game Variables ---
        const canvas = document.getElementById('pong');
        const ctx = canvas.getContext('2d');
        const WIDTH = canvas.width, HEIGHT = canvas.height;
        const PADDLE_WIDTH = 12, PADDLE_HEIGHT = 80, BALL_SIZE = 16;
        const PADDLE_SPEED = 7;
        const BALL_SPEED_START = 5, BALL_SPEED_INC = 0.15, BALL_SPEED_MAX = 18;

        let player = 0; // 0 or 1
        let gameState = null;
        let upPressed = false, downPressed = false;
        let connected = false, waiting = true;

        // --- Socket.IO Setup ---
        const socket = io('https://pong-multiplayer-server.fly.dev/', { transports: ['websocket'] });

        socket.on('connect', () => {
            document.getElementById('info').textContent = 'Waiting for opponent...';
            connected = true;
        });

        socket.on('init', (data) => {
            player = data.player;
            document.getElementById('info').textContent = player === 0 ? 'You are LEFT (W/S or ↑/↓)' : 'You are RIGHT (W/S or ↑/↓)';
        });

        socket.on('state', (state) => {
            gameState = state;
            waiting = false;
        });

        socket.on('wait', () => {
            waiting = true;
            document.getElementById('info').textContent = 'Waiting for opponent...';
        });

        // --- Input Handling ---
        document.addEventListener('keydown', (e) => {
            if (e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W') upPressed = true;
            if (e.key === 'ArrowDown' || e.key === 's' || e.key === 'S') downPressed = true;
        });
        document.addEventListener('keyup', (e) => {
            if (e.key === 'ArrowUp' || e.key === 'w' || e.key === 'W') upPressed = false;
            if (e.key === 'ArrowDown' || e.key === 's' || e.key === 'S') downPressed = false;
        });

        // --- Send Paddle Movement ---
        setInterval(() => {
            if (!connected || waiting) return;
            let dir = 0;
            if (upPressed) dir -= 1;
            if (downPressed) dir += 1;
            socket.emit('move', dir);
        }, 1000 / 60);

        // --- Drawing Functions ---
        function drawRect(x, y, w, h, color) {
            ctx.fillStyle = color;
            ctx.fillRect(x, y, w, h);
        }
        function drawCircle(x, y, r, color) {
            ctx.fillStyle = color;
            ctx.beginPath();
            ctx.arc(x, y, r, 0, Math.PI * 2);
            ctx.fill();
        }
        function drawText(text, x, y, size = 32) {
            ctx.fillStyle = '#fff';
            ctx.font = `${size}px Arial`;
            ctx.textAlign = 'center';
            ctx.fillText(text, x, y);
        }

        // --- Main Render Loop ---
        function render() {
            ctx.clearRect(0, 0, WIDTH, HEIGHT);

            if (!gameState) {
                drawText('Connecting...', WIDTH/2, HEIGHT/2);
                requestAnimationFrame(render);
                return;
            }

            // Draw middle line
            ctx.setLineDash([10, 10]);
            ctx.strokeStyle = '#444';
            ctx.beginPath();
            ctx.moveTo(WIDTH/2, 0);
            ctx.lineTo(WIDTH/2, HEIGHT);
            ctx.stroke();
            ctx.setLineDash([]);

            // Draw paddles
            drawRect(gameState.paddles[0].x, gameState.paddles[0].y, PADDLE_WIDTH, PADDLE_HEIGHT, '#0ff');
            drawRect(gameState.paddles[1].x, gameState.paddles[1].y, PADDLE_WIDTH, PADDLE_HEIGHT, '#f0f');

            // Draw ball
            drawCircle(gameState.ball.x, gameState.ball.y, BALL_SIZE/2, '#fff');

            // Draw scores
            drawText(gameState.scores[0], WIDTH/4, 50, 40);
            drawText(gameState.scores[1], WIDTH*3/4, 50, 40);

            // Draw speed
            drawText(`Speed: ${gameState.ball.speed.toFixed(1)}`, WIDTH/2, 30, 20);

            requestAnimationFrame(render);
        }
        render();
    </script>
</body>
</html>
