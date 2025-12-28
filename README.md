<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>WinCorder TGR</title>
<style>
  body { font-family: sans-serif; padding: 20px; }
  label { display: block; margin-top: 10px; }
  video, canvas { margin-top: 20px; border: 1px solid black; max-width: 100%; }
</style>
</head>
<body>
<h1>WinCorder TGR - WebM Recorder with Watermark</h1>

<label>
  Resolution:
  <select id="resolution">
    <option value="640x480">640x480</option>
    <option value="1280x720">1280x720</option>
    <option value="1920x1080">1920x1080</option>
  </select>
</label>

<label>
  FPS:
  <input type="number" id="fps" value="30" min="1" max="60">
</label>

<label>
  Audio:
  <input type="checkbox" id="microphone"> Microphone
  <input type="checkbox" id="systemAudio"> System Audio (Tab/Screen)
</label>

<label>
  Preset:
  <select id="preset">
    <option value="">Custom</option>
    <option value="ntsc">NTSC (1280x720, 30fps)</option>
    <option value="pal">PAL (1280x720, 25fps)</option>
    <option value="tv">TV (640x480, 15fps)</option>
  </select>
</label>

<button id="start">Start Recording</button>
<button id="stop" disabled>Stop Recording</button>
<a id="download" style="display:none">Download</a>

<video id="preview" autoplay muted style="display:none"></video>
<canvas id="canvasPreview"></canvas>

<script>
let mediaRecorder;
let recordedChunks = [];
let animationFrameId;

const startButton = document.getElementById('start');
const stopButton = document.getElementById('stop');
const downloadLink = document.getElementById('download');
const preview = document.getElementById('preview');
const canvas = document.getElementById('canvasPreview');
const ctx = canvas.getContext('2d');

function applyPreset() {
  const preset = document.getElementById('preset').value;
  const resSelect = document.getElementById('resolution');
  const fpsInput = document.getElementById('fps');

  switch(preset) {
    case 'ntsc':
      resSelect.value = "1280x720";
      fpsInput.value = 30;
      break;
    case 'pal':
      resSelect.value = "1280x720";
      fpsInput.value = 25;
      break;
    case 'tv':
      resSelect.value = "640x480";
      fpsInput.value = 15;
      break;
  }
}

document.getElementById('preset').addEventListener('change', applyPreset);

function drawWatermark() {
  const text = "www.wincorder.tgr";
  ctx.font = "bold 40px Arial";
  ctx.textAlign = "center";
  ctx.textBaseline = "top";
  ctx.fillStyle = "rgba(255,255,255,0.8)";
  ctx.strokeStyle = "rgba(0,0,0,0.8)";
  ctx.lineWidth = 2;
  const x = canvas.width / 2;
  const y = 10;
  ctx.strokeText(text, x, y);
  ctx.fillText(text, x, y);
}

function renderFrame(video) {
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
  drawWatermark();
  animationFrameId = requestAnimationFrame(() => renderFrame(video));
}

startButton.addEventListener('click', async () => {
  startButton.disabled = true;
  stopButton.disabled = false;

  const resolution = document.getElementById('resolution').value.split('x');
  const width = parseInt(resolution[0]);
  const height = parseInt(resolution[1]);
  const fps = parseInt(document.getElementById('fps').value);
  const useMic = document.getElementById('microphone').checked;
  const useSystemAudio = document.getElementById('systemAudio').checked;

  canvas.width = width;
  canvas.height = height;

  // Capture display (screen/tab)
  const displayStream = await navigator.mediaDevices.getDisplayMedia({
    video: { width, height, frameRate: fps },
    audio: useSystemAudio
  });

  if (useMic) {
    const micStream = await navigator.mediaDevices.getUserMedia({ audio: true });
    micStream.getAudioTracks().forEach(track => displayStream.addTrack(track));
  }

  preview.srcObject = displayStream;

  // Render to canvas with watermark
  renderFrame(preview);

  // Capture canvas stream
  const canvasStream = canvas.captureStream(fps);
  const combinedStream = new MediaStream([
    ...canvasStream.getVideoTracks(),
    ...displayStream.getAudioTracks()
  ]);

  mediaRecorder = new MediaRecorder(combinedStream, { mimeType: 'video/webm;codecs=vp9,opus' });

  mediaRecorder.ondataavailable = e => {
    if (e.data.size > 0) recordedChunks.push(e.data);
  };

  mediaRecorder.onstop = () => {
    cancelAnimationFrame(animationFrameId);
    const blob = new Blob(recordedChunks, { type: 'video/webm' });
    recordedChunks = [];
    const url = URL.createObjectURL(blob);
    downloadLink.href = url;
    downloadLink.download = 'recording.webm';
    downloadLink.style.display = 'inline';
    downloadLink.textContent = 'Download Recording';
  };

  mediaRecorder.start();
});

stopButton.addEventListener('click', () => {
  mediaRecorder.stop();
  startButton.disabled = false;
  stopButton.disabled = true;
});
</script>
</body>
</html>
