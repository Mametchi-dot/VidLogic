<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Build Logic — RGB Video Converter</title>
    <style>
        :root {
            --bg-body: #0d1117;
            --bg-card: #161b22;
            --bg-input: #0b0e14;
            --border: #30363d;
            --border-focus: #58a6ff;
            --text-main: #f0f6fc;
            --text-sub: #8b949e;
            --btn-green: #238636;
            --btn-green-hover: #2ea043;
            --btn-gray: #21262d;
            --btn-gray-hover: #30363d;
        }

        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }

        body {
            background-color: var(--bg-body);
            color: var(--text-main);
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            padding: 24px;
        }

        .container {
            width: 100%;
            max-width: 620px;
            display: flex;
            flex-direction: column;
            gap: 16px;
        }

        header {
            text-align: center;
            margin-bottom: 8px;
        }

        header h1 {
            font-size: 1.6rem;
            color: #58a6ff;
            margin-bottom: 4px;
        }

        header p {
            color: var(--text-sub);
            font-size: 0.9rem;
        }

        .card {
            background-color: var(--bg-card);
            border: 1px solid var(--border);
            border-radius: 8px;
            padding: 18px;
        }

        .card h2 {
            font-size: 1rem;
            font-weight: 600;
            margin-bottom: 12px;
            color: var(--text-main);
        }

        label {
            display: block;
            font-size: 0.85rem;
            font-weight: 600;
            color: var(--text-sub);
            margin-bottom: 6px;
        }

        input[type="file"], select {
            width: 100%;
            padding: 10px;
            background: var(--bg-input);
            border: 1px solid var(--border);
            border-radius: 6px;
            color: var(--text-main);
            font-size: 0.9rem;
            outline: none;
            margin-bottom: 12px;
        }

        input[type="file"]:hover, select:hover {
            border-color: #484f58;
        }

        select:focus {
            border-color: var(--border-focus);
        }

        /* Screen Preview */
        .preview-box {
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 14px;
        }

        .canvas-frame {
            background: #000;
            padding: 10px;
            border-radius: 8px;
            border: 2px solid var(--border);
            box-shadow: 0 4px 16px rgba(0, 0, 0, 0.4);
        }

        canvas {
            display: block;
            image-rendering: pixelated;
            border-radius: 4px;
            background-color: #000;
            width: 320px;
            height: 320px;
        }

        .btn-row {
            display: flex;
            gap: 10px;
            width: 100%;
        }

        button {
            padding: 10px 16px;
            font-size: 0.9rem;
            font-weight: 600;
            border-radius: 6px;
            border: 1px solid transparent;
            cursor: pointer;
            transition: background 0.15s ease, border-color 0.15s ease;
        }

        .btn-play {
            background-color: var(--btn-gray);
            color: var(--text-main);
            border-color: var(--border);
            flex: 1;
        }

        .btn-play:hover {
            background-color: var(--btn-gray-hover);
            border-color: #8b949e;
        }

        .btn-copy {
            background-color: var(--btn-green);
            color: #ffffff;
            flex: 2;
        }

        .btn-copy:hover {
            background-color: var(--btn-green-hover);
        }

        /* Output Box */
        textarea {
            width: 100%;
            height: 90px;
            background: var(--bg-input);
            color: #79c0ff;
            border: 1px solid var(--border);
            border-radius: 6px;
            padding: 10px;
            font-family: ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas, monospace;
            font-size: 0.8rem;
            line-height: 1.4;
            resize: vertical;
            outline: none;
        }

        textarea:focus {
            border-color: var(--border-focus);
        }
    </style>
</head>
<body>

<div class="container">
    <header>
        <h1>Build Logic RGB Player</h1>
        <p>Convert video frames into packed 24-bit RGB arrays for Roblox</p>
    </header>

    <!-- Settings Card -->
    <div class="card">
        <h2>1. Video Settings</h2>
        
        <label for="videoUpload">Choose Video File (.mp4, .webm):</label>
        <input type="file" id="videoUpload" accept="video/*">

        <label for="gridSize">RGB Panel Size:</label>
        <select id="gridSize">
            <option value="16">16 × 16 (256 Pixels)</option>
            <option value="32" selected>32 × 32 (1,024 Pixels)</option>
            <option value="48">48 × 48 (2,304 Pixels)</option>
        </select>
    </div>

    <!-- Hidden Video Player for processing -->
    <video id="hiddenVideo" hidden playsinline muted></video>

    <!-- Display Card -->
    <div class="card preview-box">
        <div class="canvas-frame">
            <canvas id="displayCanvas" width="320" height="320"></canvas>
        </div>

        <div class="btn-row">
            <button class="btn-play" id="playBtn" onclick="togglePlay()">▶ Play Preview</button>
            <button class="btn-copy" id="copyBtn" onclick="copyCurrentFrame()">⚡ Copy Current Frame</button>
        </div>
    </div>

    <!-- Output Card -->
    <div class="card">
        <h2>2. Build Logic Output Data</h2>
        <textarea id="outputData" readonly placeholder="Click 'Copy Current Frame' to generate your in-game array..."></textarea>
    </div>
</div>

<script>
    const fileInput = document.getElementById('videoUpload');
    const video = document.getElementById('hiddenVideo');
    const canvas = document.getElementById('displayCanvas');
    const ctx = canvas.getContext('2d');
    const outputData = document.getElementById('outputData');
    const gridSizeSelect = document.getElementById('gridSize');
    const playBtn = document.getElementById('playBtn');
    const copyBtn = document.getElementById('copyBtn');

    const sampleCanvas = document.createElement('canvas');
    const sampleCtx = sampleCanvas.getContext('2d');

    let isPlaying = false;
    let animFrameId = null;

    fileInput.addEventListener('change', function(e) {
        const file = e.target.files[0];
        if (file) {
            video.src = URL.createObjectURL(file);
            video.load();
            video.onloadeddata = () => {
                video.currentTime = 0;
                drawCurrentFrame();
            };
        }
    });

    gridSizeSelect.addEventListener('change', () => {
        drawCurrentFrame();
    });

    function togglePlay() {
        if (!video.src) {
            alert("Please select a video file first.");
            return;
        }

        if (isPlaying) {
            video.pause();
            cancelAnimationFrame(animFrameId);
            playBtn.textContent = "▶ Play Preview";
            isPlaying = false;
        } else {
            video.play();
            playBtn.textContent = "❚❚ Pause";
            isPlaying = true;
            renderLoop();
        }
    }

    function renderLoop() {
        if (!video.paused && !video.ended) {
            drawCurrentFrame();
            animFrameId = requestAnimationFrame(renderLoop);
        } else {
            playBtn.textContent = "▶ Play Preview";
            isPlaying = false;
        }
    }

    function drawCurrentFrame() {
        const size = parseInt(gridSizeSelect.value, 10);
        sampleCanvas.width = size;
        sampleCanvas.height = size;

        sampleCtx.drawImage(video, 0, 0, size, size);
        const imgData = sampleCtx.getImageData(0, 0, size, size).data;

        const pixelW = canvas.width / size;
        const pixelH = canvas.height / size;

        ctx.clearRect(0, 0, canvas.width, canvas.height);

        for (let i = 0; i < imgData.length; i += 4) {
            const r = imgData[i];
            const g = imgData[i + 1];
            const b = imgData[i + 2];

            const index = i / 4;
            const x = (index % size) * pixelW;
            const y = Math.floor(index / size) * pixelH;

            ctx.fillStyle = `rgb(${r},${g},${b})`;
            ctx.fillRect(x, y, pixelW, pixelH);
        }
    }

    function copyCurrentFrame() {
        if (!video.src) {
            alert("Please select a video file first.");
            return;
        }

        const size = parseInt(gridSizeSelect.value, 10);
        sampleCanvas.width = size;
        sampleCanvas.height = size;
        sampleCtx.drawImage(video, 0, 0, size, size);
        const imgData = sampleCtx.getImageData(0, 0, size, size).data;

        const packedArray = [];

        // 24-bit decimal packed format for Build Logic: (R << 16) | (G << 8) | B
        for (let i = 0; i < imgData.length; i += 4) {
            const r = imgData[i];
            const g = imgData[i + 1];
            const b = imgData[i + 2];
            const packed = (r << 16) | (g << 8) | b;
            packedArray.push(packed);
        }

        const resultString = JSON.stringify(packedArray);
        outputData.value = resultString;
        navigator.clipboard.writeText(resultString);

        const originalText = copyBtn.textContent;
        copyBtn.textContent = "✓ Copied to Clipboard!";
        setTimeout(() => {
            copyBtn.textContent = originalText;
        }, 1200);
    }
</script>

</body>
</html>
