# Haspera-mvp
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Idea → Video (Zero Budget MVP)</title>
  <style>
    body { font-family: Inter, system-ui, sans-serif; background:#0b0b0b; color:#fff; margin:0; padding:20px; }
    .container { max-width:900px; margin:0 auto; }
    header { text-align:left; margin-bottom:18px; }
    h1 { font-size:32px; margin:0 0 8px 0; }
    .row { display:flex; gap:12px; align-items:center; margin-bottom:12px; }
    input, textarea, select, button { padding:10px; border-radius:8px; border:1px solid #222; background:#111; color:#fff; }
    textarea { width:100%; min-height:120px; }
    .scene { border:1px dashed #222; padding:8px; margin-bottom:8px; border-radius:8px; }
    #previewCanvas { background:#222; width:800px; height:450px; display:block; margin:12px 0; }
    .small { font-size:13px; color:#bbb; }
    .controls { display:flex; gap:8px; flex-wrap:wrap; }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>From idea → video (zero budget MVP)</h1>
      <div class="small">Enter a short idea, auto-generate a script, attach images & record voice, then export a WebM.</div>
    </header>

    <!-- Idea -->
    <section>
      <div class="row">
        <input id="ideaInput" placeholder="A 30s explainer about switching to LED lights" style="flex:1" />
        <select id="toneSelect">
          <option value="neutral">Neutral</option>
          <option value="funny">Funny</option>
          <option value="serious">Serious</option>
        </select>
        <select id="lengthSelect">
          <option value="15">15s</option>
          <option value="30" selected>30s</option>
          <option value="60">60s</option>
        </select>
        <button id="genScriptBtn">Auto-Script</button>
      </div>
    </section>

    <!-- Script editor -->
    <section>
      <label class="small">Script (editable)</label>
      <textarea id="scriptArea"></textarea>
      <div class="controls">
        <button id="splitScenesBtn">Split into scenes</button>
        <button id="fetchImagesBtn">Auto-fetch images</button>
      </div>
    </section>

    <!-- Scenes -->
    <section>
      <h3>Scenes</h3>
      <div id="scenesContainer"></div>
    </section>

    <!-- Audio recorder -->
    <section>
      <h3>Voice / Audio</h3>
      <div class="row">
        <button id="startRecBtn">Record Voice</button>
        <button id="stopRecBtn" disabled>Stop</button>
        <button id="ttsBtn">TTS (speechSynthesis)</button>
        <audio id="audioPreview" controls></audio>
      </div>
      <div class="small">Recording stored locally; you can re-record or use TTS as fallback.</div>
    </section>

    <!-- Preview & Export -->
    <section>
      <h3>Preview & Export</h3>
      <canvas id="previewCanvas" width="800" height="450"></canvas>
      <div class="controls">
        <button id="createVideoBtn">Create Video (Record Canvas → WebM)</button>
        <a id="downloadLink" style="display:inline-block; padding:10px; background:#144; border-radius:8px; color:#fff; text-decoration:none;">Download</a>
      </div>
    </section>

    <footer style="margin-top:28px;">
      <div class="small">Notes: Unsplash images use source.unsplash.com (no API key). For MP4 replace the canvas-record logic with ffmpeg.wasm if needed later.</div>
    </footer>
  </div>

<script>
/* ---------- Simple script generator (template-based) ---------- */
function generateScriptFromIdea(idea, tone, lengthSec) {
  // naive template: split into 3 acts based on length
  const acts = Math.max(2, Math.round((lengthSec/15) + 1)); // 2..5 scenes
  const sentences = [];
  for (let i=0;i<acts;i++) {
    if (i===0) sentences.push(\`Intro: \${idea} — why it matters.\`);
    else if (i===acts-1) sentences.push(\`CTA: What should viewer do next? (eg. try it, share)\`);
    else sentences.push(\`Point \${i}: quick actionable line about \${idea}.\`);
  }
  // tone adjustments (tiny)
  const tonePreface = tone === 'funny' ? 'Quick laugh: ' : tone === 'serious' ? 'Fact: ' : '';
  return sentences.map(s => tonePreface + s).join('\\n\\n');
}

/* ---------- UI references ---------- */
const ideaInput = document.getElementById('ideaInput');
const toneSelect = document.getElementById('toneSelect');
const lengthSelect = document.getElementById('lengthSelect');
const genScriptBtn = document.getElementById('genScriptBtn');
const scriptArea = document.getElementById('scriptArea');
const splitScenesBtn = document.getElementById('splitScenesBtn');
const scenesContainer = document.getElementById('scenesContainer');
const fetchImagesBtn = document.getElementById('fetchImagesBtn');

genScriptBtn.onclick = () => {
  const idea = ideaInput.value.trim() || 'Why switch to LED lights';
  const tone = toneSelect.value;
  const length = parseInt(lengthSelect.value,10);
  scriptArea.value = generateScriptFromIdea(idea, tone, length);
}

/* ---------- Scene management ---------- */
function splitScriptToScenes(scriptText) {
  const blocks = scriptText.split(/\\n\\n+/).map(s=>s.trim()).filter(Boolean);
  return blocks.map((text, idx) => ({
    id: 'scene-' + idx,
    text,
    image: null,
    imageUrl: null,
    duration:  Math.max(3, Math.round((parseInt(lengthSelect.value,10) || 30)/Math.max(1,blocks.length)))
  }));
}

let scenes = [];
splitScenesBtn.onclick = () => {
  scenes = splitScriptToScenes(scriptArea.value || '');
  renderScenes();
}

function renderScenes(){
  scenesContainer.innerHTML = '';
  scenes.forEach((scene, i) => {
    const el = document.createElement('div'); el.className='scene';
    el.innerHTML = \`
      <div><strong>Scene \${i+1} (\${scene.duration}s)</strong></div>
      <div style="margin:8px 0">\${scene.text}</div>
      <div style="display:flex; gap:8px; align-items:center">
        <img id="img-\${scene.id}" src="\${scene.imageUrl||''}" alt="" width="200" height="112" style="object-fit:cover; background:#222;"/>
        <div style="flex:1">
          <input placeholder="Upload image or use auto-fetch" type="file" accept="image/*" data-scene="\${scene.id}"/>
          <div style="margin-top:8px;">
            <button data-action="fetch" data-scene="\${scene.id}">Fetch Unsplash</button>
            <button data-action="remove" data-scene="\${scene.id}">Remove</button>
          </div>
        </div>
      </div>
    \`;
    scenesContainer.appendChild(el);
  });
}

// handle file uploads and buttons
scenesContainer.addEventListener('change', (ev)=>{
  const fileInput = ev.target;
  if (fileInput && fileInput.files && fileInput.files[0]) {
    const id = fileInput.dataset.scene;
    const scene = scenes.find(s=>s.id===id);
    const file = fileInput.files[0];
    scene.image = file;
    scene.imageUrl = URL.createObjectURL(file);
    document.getElementById('img-'+id).src = scene.imageUrl;
  }
});

scenesContainer.addEventListener('click', (ev)=>{
  const btn = ev.target.closest('button');
  if (!btn) return;
  const id = btn.dataset.scene;
  const action = btn.dataset.action;
  const scene = scenes.find(s=>s.id===id);
  if (!scene) return;
  if (action === 'fetch') {
    fetchImageForScene(scene);
  } else if (action === 'remove') {
    scene.image = null; scene.imageUrl = null;
    document.getElementById('img-'+id).src = '';
  }
});

/* ---------- Unsplash lightweight fetch (no key) ---------- */
function fetchImageForScene(scene) {
  // Use source.unsplash.com which returns a random image matching the query
  // Build query from scene.text (simple keyword extraction)
  const q = encodeURIComponent(scene.text.split(/\\W+/).slice(0,6).join(',') || 'nature');
  const url = \`https://source.unsplash.com/800x450/?\${q}\`;
  // Assign url (these are direct image endpoints; each request returns one image)
  scene.imageUrl = url + '&' + Date.now(); // cache-bust
  renderScenes(); // update UI
}

/* ---------- Audio recording (getUserMedia + MediaRecorder) ---------- */
let mediaRecorder, audioChunks = [], audioBlob = null;
const startRecBtn = document.getElementById('startRecBtn');
const stopRecBtn = document.getElementById('stopRecBtn');
const audioPreview = document.getElementById('audioPreview');

startRecBtn.onclick = async () => {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    mediaRecorder = new MediaRecorder(stream);
    audioChunks = [];
    mediaRecorder.ondataavailable = e => audioChunks.push(e.data);
    mediaRecorder.onstop = () => {
      audioBlob = new Blob(audioChunks, { type: 'audio/webm' });
      audioPreview.src = URL.createObjectURL(audioBlob);
      audioPreview.controls = true;
    };
    mediaRecorder.start();
    startRecBtn.disabled = true;
    stopRecBtn.disabled = false;
  } catch (err) {
    alert('Microphone access denied or unsupported: ' + err.message);
  }
}

stopRecBtn.onclick = () => {
  if (mediaRecorder && mediaRecorder.state !== 'inactive') mediaRecorder.stop();
  startRecBtn.disabled = false;
  stopRecBtn.disabled = true;
}

/* ---------- Simple TTS fallback (speechSynthesis) ---------- */
const ttsBtn = document.getElementById('ttsBtn');
ttsBtn.onclick = () => {
  const script = scriptArea.value || '';
  if (!script.trim()) return alert('No script to TTS');
  // We'll produce a single audio blob by using SpeechSynthesis and capture via MediaStream (experimental)
  // Simpler UX: just play TTS and offer user to record from system if they want; here we just play.
  const utter = new SpeechSynthesisUtterance(script);
  speechSynthesis.speak(utter);
  alert('TTS playing in browser. To include TTS in the exported video, record your voice while TTS plays (or implement server-side TTS later).');
}

/* ---------- Canvas-based stitch + MediaRecorder (creates WebM) ---------- */
const previewCanvas = document.getElementById('previewCanvas');
const canvasCtx = previewCanvas.getContext('2d');

async function createVideoFromScenes() {
  if (!scenes.length) return alert('No scenes defined');
  if (!audioBlob) {
    if (!confirm('No recorded voice. Continue with silent video?')) return;
  }

  // Preload scene images
  const loadedImages = await Promise.all(scenes.map(s=>loadImage(s.imageUrl)));

  // Create an audio element to play the recorded audio (if present)
  let audioEl = null;
  let audioDuration = 0;
  if (audioBlob) {
    audioEl = document.createElement('audio');
    audioEl.src = URL.createObjectURL(audioBlob);
    await audioEl.play().catch(()=>{}); // may be blocked; we will control playback later
    audioEl.pause();
    audioDuration = audioEl.duration || scenes.reduce((a,b)=>a+b.duration,0);
  }

  // Build timeline: scenes will display for their duration; if audio exists, we sync to audio length by scaling durations
  const totalSceneDur = scenes.reduce((a,b)=>a+b.duration,0);
  const targetTotal = audioDuration > 0 ? audioDuration : totalSceneDur;
  // compute scaled durations in ms
  const durationsMs = scenes.map(s => (s.duration/totalSceneDur) * targetTotal * 1000);

  // Prepare canvas capture stream + recorder
  const stream = previewCanvas.captureStream(30); // 30 fps
  // Optionally mix audio into the stream: create an AudioContext and connect audio element to stream via MediaStreamDestination
  if (audioEl) {
    const ac = new (window.AudioContext || window.webkitAudioContext)();
    const src = ac.createMediaElementSource(audioEl);
    const dest = ac.createMediaStreamDestination();
    src.connect(ac.destination); // to hear
    src.connect(dest); // to push into a MediaStream
    // combine canvas stream + audio tracks
    dest.stream.getAudioTracks().forEach(track => stream.addTrack(track));
  }

  const options = { mimeType: 'video/webm; codecs=vp8,opus' };
  const recorder = new MediaRecorder(stream, options);
  const chunks = [];
  recorder.ondataavailable = e => { if (e.data && e.data.size) chunks.push(e.data); };

  recorder.start();
  // start audio playback slightly after recorder start
  if (audioEl) { audioEl.currentTime = 0; await audioEl.play().catch(()=>{}); }

  // render each scene sequentially
  for (let i=0;i<scenes.length;i++){
    const img = loadedImages[i];
    const text = scenes[i].text;
    const duration = durationsMs[i];
    // draw frame repeatedly during this scene to keep motion for the recorder
    const fps = 30;
    const frames = Math.max(1, Math.round((duration/1000) * fps));
    for (let f=0; f<frames; f++){
      // clear
      canvasCtx.fillStyle = '#0b0b0b';
      canvasCtx.fillRect(0,0, previewCanvas.width, previewCanvas.height);
      // draw image centered/cover
      if (img) {
        // cover behavior
        const iw = img.width, ih = img.height, cw = previewCanvas.width, ch = previewCanvas.height;
        const ir = iw/ih, cr = cw/ch;
        let dw, dh, dx, dy;
        if (ir > cr) {
          dh = ch; dw = ch * ir; dx = (cw - dw)/2; dy = 0;
        } else {
          dw = cw; dh = cw / ir; dx = 0; dy = (ch - dh)/2;
        }
        canvasCtx.drawImage(img, dx, dy, dw, dh);
      } else {
        // placeholder
        canvasCtx.fillStyle = '#222';
        canvasCtx.fillRect(0,0, previewCanvas.width, previewCanvas.height);
      }
      // overlay semi-transparent strip + text
      canvasCtx.fillStyle = 'rgba(0,0,0,0.4)';
      canvasCtx.fillRect(0, previewCanvas.height - 110, previewCanvas.width, 110);
      canvasCtx.fillStyle = '#fff';
      canvasCtx.font = '22px sans-serif';
      wrapText(canvasCtx, text, 20, previewCanvas.height - 70, previewCanvas.width - 40, 24);
      await sleep(1000 / fps);
    }
  }

  // finish
  if (audioEl) { audioEl.pause(); }
  recorder.stop();

  // wait until dataavailable done
  const recordedBlob = await new Promise(resolve => {
    recorder.onstop = () => resolve(new Blob(chunks, { type: 'video/webm' }));
  });

  // provide download link
  const url = URL.createObjectURL(recordedBlob);
  const a = document.getElementById('downloadLink');
  a.href = url; a.download = 'haspera_mvp.webm';
  a.textContent = 'Download video (WebM)';
  alert('Video ready — click Download.');
}

/* ---------- helpers ---------- */
function loadImage(src){
  return new Promise(resolve => {
    if (!src) { resolve(null); return; }
    const img = new Image();
    img.crossOrigin = 'anonymous';
    img.onload = () => resolve(img);
    img.onerror = () => resolve(null);
    img.src = src;
  });
}
function sleep(ms){ return new Promise(res => setTimeout(res, ms)); }

function wrapText(ctx, text, x, y, maxWidth, lineHeight) {
  const words = text.split(' ');
  let line = '';
  let testY = y;
  for (let n = 0; n < words.length; n++) {
    const testLine = line + words[n] + ' ';
    const metrics = ctx.measureText(testLine);
    const testWidth = metrics.width;
    if (testWidth > maxWidth && n > 0) {
      ctx.fillText(line, x, testY);
      line = words[n] + ' ';
      testY += lineHeight;
    } else {
      line = testLine;
    }
  }
  ctx.fillText(line, x, testY);
}

/* ---------- UI wiring ---------- */
document.getElementById('fetchImagesBtn').onclick = () => {
  scenes.forEach(s => fetchImageForScene(s));
  renderScenes();
}

document.getElementById('createVideoBtn').onclick = async () => {
  document.getElementById('createVideoBtn').disabled = true;
  try {
    await createVideoFromScenes();
  } catch (e) {
    console.error(e); alert('Error creating video: ' + e.message);
  } finally {
    document.getElementById('createVideoBtn').disabled = false;
  }
}

</script>
</body>
</html>
