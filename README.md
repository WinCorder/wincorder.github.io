<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>WinCorder TGR</title>
<style>
  body { font-family: sans-serif; padding: 20px; }
  label { display: block; margin-top: 10px; }
  video, canvas {
    margin-top: 20px;
    border: 1px solid black;
    max-width: 100%;
  }
</style>
</head>
<body>

<h1>WinCorder</h1>

<label>
  Resolution:
  <select id="resolution">
    <option value="640x480">640x480</option>
    <option value="854x480">854x480</option>
    <option value="960x720">960x720</option>
    <option value="1280x720">1280x720</option>
    <option value="1440x1080">1440x1080</option>
    <option value="1920x1080">1920x1080</option>
  </select>
</label>

<label>
  FPS:
  <input type="number" id="fps" value="30" min="1" max="240">
</label>

<label>
  Audio:
  <input type="checkbox" id="microphone"> Microphone
  <input type="checkbox" id="systemAudio"> System Audio (Tab/Screen)
</label>

<label>
  <input type="checkbox" id="fixMacAudio">
  Fix macOS audio (48kHz Opus – recommended on Mac)
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
<a id="download" style="display:none"></a>

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
  const res = document.getElementById('resolution');
  const fps = document.getElementById('fps');

  if (preset === 'ntsc') { res.value = "1280x720"; fps.value = 30; }
  if (preset === 'pal')  { res.value = "1280x720"; fps.value = 25; }
  if (preset === 'tv')   { res.value = "640x480"; fps.value = 15; }
}
document.getElementById('preset').addEventListener('change', applyPreset);

// Auto-enable macOS fix
if (navigator.platform.includes("Mac")) {
  document.getElementById('fixMacAudio').checked = true;
}

function drawWatermark() {
  const text = "wincorder.github.io";
  ctx.font = "bold 40px Arial";
  ctx.textAlign = "center";
  ctx.textBaseline = "top";
  ctx.fillStyle = "rgba(255,255,255,0.8)";
  ctx.strokeStyle = "rgba(0,0,0,0.8)";
  ctx.lineWidth = 2;
  ctx.strokeText(text, canvas.width / 2, 10);
  ctx.fillText(text, canvas.width / 2, 10);
}

function renderFrame(video) {
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
  drawWatermark();
  animationFrameId = requestAnimationFrame(() => renderFrame(video));
}

startButton.addEventListener('click', async () => {
  startButton.disabled = true;
  stopButton.disabled = false;

  const [width, height] = document.getElementById('resolution').value.split('x').map(Number);
  const fps = Number(document.getElementById('fps').value);
  const useMic = document.getElementById('microphone').checked;
  const useSystemAudio = document.getElementById('systemAudio').checked;
  const fixMacAudio = document.getElementById('fixMacAudio').checked;

  canvas.width = width;
  canvas.height = height;

  let displayStream;
  try {
    displayStream = await navigator.mediaDevices.getDisplayMedia({
      video: { width, height, frameRate: fps },
      audio: useSystemAudio
    });
  } catch (err) {
    alert("Display capture failed: " + err);
    startButton.disabled = false;
    stopButton.disabled = true;
    return;
  }

  preview.srcObject = displayStream;
  await preview.play();
  renderFrame(preview);

  const canvasStream = canvas.captureStream(fps);
  let finalAudioTracks = [];

  if (useMic || useSystemAudio) {
    if (fixMacAudio) {
      const audioContext = new AudioContext({ sampleRate: 48000 });
      const destination = audioContext.createMediaStreamDestination();

      if (useSystemAudio) {
        displayStream.getAudioTracks().forEach(track => {
          const src = audioContext.createMediaStreamSource(new MediaStream([track]));
          src.connect(destination);
        });
      }

      if (useMic) {
        const micStream = await navigator.mediaDevices.getUserMedia({
          audio: {
            sampleRate: 48000,
            channelCount: 2,
            echoCancellation: false,
            noiseSuppression: false,
            autoGainControl: false
          }
        });

        micStream.getAudioTracks().forEach(track => {
          const src = audioContext.createMediaStreamSource(new MediaStream([track]));
          src.connect(destination);
        });
      }

      finalAudioTracks = destination.stream.getAudioTracks();
    } else {
      if (useSystemAudio) {
        displayStream.getAudioTracks().forEach(t => finalAudioTracks.push(t));
      }
      if (useMic) {
        const micStream = await navigator.mediaDevices.getUserMedia({ audio: true });
        micStream.getAudioTracks().forEach(t => finalAudioTracks.push(t));
      }
    }
  }

  const combinedStream = new MediaStream([
    ...canvasStream.getVideoTracks(),
    ...finalAudioTracks
  ]);

  mediaRecorder = new MediaRecorder(combinedStream, {
    mimeType: 'video/webm;codecs=vp9,opus',
    audioBitsPerSecond: 192000,
    videoBitsPerSecond: 6000000
  });

  mediaRecorder.ondataavailable = e => {
    if (e.data.size) recordedChunks.push(e.data);
  };

  mediaRecorder.onstop = () => {
    cancelAnimationFrame(animationFrameId);
    const blob = new Blob(recordedChunks, { type: 'video/webm' });
    recordedChunks = [];
    const url = URL.createObjectURL(blob);
    downloadLink.href = url;
    downloadLink.download = 'recording.webm';
    downloadLink.textContent = 'Download Recording';
    downloadLink.style.display = 'inline';
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
