<html lang="en">
<head>
<meta charset="UTF-8">
<title>Wincorder.tgr</title>
<style>
    body {
        margin: 0;
        font-family: Arial, sans-serif;
        background: #111;
        color: white;
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    .container {
        position: relative;
        width: 80%;
        max-width: 800px;
        margin-top: 20px;
    }

    video {
        width: 100%;
        border-radius: 8px;
        background: black;
    }

    /* Watermark */
    .watermark {
        position: absolute;
        top: 10px;
        left: 50%;
        transform: translateX(-50%);
        font-size: 32px;
        font-weight: bold;
        opacity: 0.5; /* 50% transparent */
        pointer-events: none;
        user-select: none;
    }

    .controls {
        margin-top: 15px;
        display: flex;
        gap: 10px;
        flex-wrap: wrap;
        justify-content: center;
    }

    button, label {
        background: #222;
        color: white;
        border: 1px solid #444;
        padding: 8px 12px;
        border-radius: 5px;
        cursor: pointer;
    }

    button:hover, label:hover {
        background: #333;
    }

    input[type="checkbox"] {
        margin-right: 6px;
    }
</style>
</head>
<body>

<h2>Wincorder.tgr</h2>

<div class="container">
    <div class="watermark">VIDEO PREVIEW</div>
    <video id="preview" autoplay muted playsinline></video>
</div>

<div class="controls">
    <label>
        <input type="checkbox" id="micToggle" checked>
        Microphone ON
    </label>

    <button id="startBtn">Start Recording</button>
    <button id="stopBtn" disabled>Stop Recording</button>
    <a id="downloadLink" style="display:none;">Download Video</a>
</div>

<script>
let mediaRecorder;
let recordedChunks = [];
let canvasStream;
let canvas, ctx, videoElement;

const preview = document.getElementById("preview");
const startBtn = document.getElementById("startBtn");
const stopBtn = document.getElementById("stopBtn");
const micToggle = document.getElementById("micToggle");
const downloadLink = document.getElementById("downloadLink");

// Watermark opacity
const watermarkOpacity = 0.75; // 0.0 = invisible, 1.0 = fully visible

micToggle.addEventListener("change", () => {
    micToggle.parentElement.lastChild.textContent =
        micToggle.checked ? " Microphone ON" : " Microphone OFF";
});

async function startScreenRecording() {
    recordedChunks = [];

    // Ask for microphone and screen audio
    const useMic = micToggle.checked;

    // Get screen video and optional audio
    const screenStream = await navigator.mediaDevices.getDisplayMedia({
        video: { frameRate: 24.97 },
        audio: true // this captures screen/system audio if available
    });

    // Optional microphone audio
    let audioStream = null;
    if (useMic) {
        audioStream = await navigator.mediaDevices.getUserMedia({ audio: true });
    }

    // Set up hidden video element to draw screen onto canvas
    videoElement = document.createElement("video");
    videoElement.srcObject = screenStream;
    await videoElement.play();

    // Set up canvas
    canvas = document.createElement("canvas");
    canvas.width = videoElement.videoWidth || 1280;
    canvas.height = videoElement.videoHeight || 720;
    ctx = canvas.getContext("2d");

    function drawCanvas() {
        ctx.drawImage(videoElement, 0, 0, canvas.width, canvas.height);

        // Draw watermark
        ctx.font = "60px Arial";
        ctx.fillStyle = `rgba(178,174,179,${watermarkOpacity})`;
        ctx.textAlign = "center";
        ctx.fillText("Wincorder.tgr", canvas.width / 2, 60);

        requestAnimationFrame(drawCanvas);
    }
    drawCanvas();

    // Capture canvas stream
    canvasStream = canvas.captureStream(60);

    // Combine with microphone if needed
    let finalStream;
    if (audioStream) {
        finalStream = new MediaStream([
            ...canvasStream.getVideoTracks(),
            ...screenStream.getAudioTracks(), // screen audio
            ...audioStream.getAudioTracks()   // mic audio
        ]);
    } else {
        finalStream = new MediaStream([
            ...canvasStream.getVideoTracks(),
            ...screenStream.getAudioTracks() // just screen audio
        ]);
    }

    preview.srcObject = finalStream;

    mediaRecorder = new MediaRecorder(finalStream, { mimeType: "video/webm" });
    mediaRecorder.ondataavailable = e => {
        if (e.data.size > 0) recordedChunks.push(e.data);
    };
    mediaRecorder.onstop = () => {
        const blob = new Blob(recordedChunks, { type: "video/webm" });
        const url = URL.createObjectURL(blob);

        downloadLink.href = url;
        downloadLink.download = "Wincorder Video.webm";
        downloadLink.textContent = "Download Video";
        downloadLink.style.display = "inline";
    };

    mediaRecorder.start();
    startBtn.disabled = true;
    stopBtn.disabled = false;

    // Stop if user stops sharing screen
    screenStream.getVideoTracks()[0].addEventListener("ended", stopRecording);
}

function stopRecording() {
    if (mediaRecorder && mediaRecorder.state !== "inactive") {
        mediaRecorder.stop();
    }
    if (canvasStream) {
        canvasStream.getTracks().forEach(track => track.stop());
    }
    if (videoElement) {
        videoElement.pause();
        videoElement.srcObject = null;
    }
    startBtn.disabled = false;
    stopBtn.disabled = true;
}

startBtn.addEventListener("click", startScreenRecording);
stopBtn.addEventListener("click", stopRecording);
</script>

</body>
</html>
