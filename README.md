
<html lang="de">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Lieferschein Scanner</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background: #f2f2f2;
      padding: 1rem;
      max-width: 500px;
      margin: auto;
      color: #333;
    }

    h1 {
      text-align: center;
      color: #2c3e50;
    }

    button, input {
      margin: 0.4rem 0;
      padding: 0.8rem;
      width: 100%;
      font-size: 1rem;
      border: none;
      border-radius: 5px;
      box-shadow: 0 2px 5px rgba(0,0,0,0.1);
      transition: background-color 0.2s ease;
    }

    input {
      border: 1px solid #ccc;
    }

    button {
      background-color: #3498db;
      color: white;
      cursor: pointer;
    }

    button:hover {
      background-color: #2980b9;
    }

    #stopBtn { background-color: #c0392b; }
    #stopBtn:hover { background-color: #a93226; }

    #shareBtn { background-color: #27ae60; }
    #shareBtn:hover { background-color: #1e8449; }

    #copyBtn { background-color: #f39c12; }
    #copyBtn:hover { background-color: #d68910; }

    #undoBtn { background-color: #e67e22; }
    #undoBtn:hover { background-color: #ca6f1e; }

    #clearBtn { background-color: #95a5a6; }
    #clearBtn:hover { background-color: #7f8c8d; }

    #videoContainer {
      position: relative;
      border: 2px solid #ccc;
      border-radius: 6px;
      overflow: hidden;
      margin: 1rem 0;
      background-color: #000;
    }

    video {
      width: 100%;
    }

    #scannerFrame {
      position: absolute;
      top: 25%;
      left: 25%;
      width: 50%;
      height: 30%;
      border: 2px dashed red;
      pointer-events: none;
    }

    #flash {
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      background: white;
      opacity: 0;
      pointer-events: none;
      transition: opacity 0.2s ease;
    }

    #notification {
      text-align: center;
      color: red;
      min-height: 1.2rem;
      margin-top: 0.5rem;
    }

    ul {
      list-style: disc;
      background: white;
      padding-left: 1.2rem;
      max-height: 200px;
      overflow-y: auto;
      border: 1px solid #ccc;
      border-radius: 5px;
      margin-top: 0.5rem;
    }

    li {
      padding: 0.3rem 0.5rem;
      border-bottom: 1px solid #eee;
    }
  </style>
</head>
<body>

<h1>Lieferschein Scanner</h1>

<button id="startBtn">📷 Scanner starten</button>
<button id="stopBtn" disabled>⛔ Scanner stoppen</button>
<button id="shareBtn">📤 Teilen</button>
<button id="copyBtn">📋 In Zwischenablage kopieren</button>
<button id="undoBtn">↩️ Letzten Scan löschen</button>
<button id="clearBtn">🗑️ Alle Scans löschen</button>

<div id="notification"></div>

<div id="videoContainer">
  <video id="video" autoplay muted playsinline></video>
  <div id="scannerFrame"></div>
</div>

<div id="flash"></div>

<h2>Gespeicherte Barcodes:</h2>
<ul id="barcodeList"></ul>

<!-- ZXing Barcode Library -->
<script src="https://cdn.jsdelivr.net/npm/@zxing/library@0.18.6/umd/index.min.js"></script>
<script>
  const startBtn = document.getElementById('startBtn');
  const stopBtn = document.getElementById('stopBtn');
  const shareBtn = document.getElementById('shareBtn');
  const copyBtn = document.getElementById('copyBtn');
  const undoBtn = document.getElementById('undoBtn');
  const clearBtn = document.getElementById('clearBtn');
  const video = document.getElementById('video');
  const barcodeListEl = document.getElementById('barcodeList');
  const notification = document.getElementById('notification');
  const flash = document.getElementById('flash');

  let barcodes = [];
  let codeReader = null;
  let scanning = false;
  let selectedDeviceId = null;

  const beep = () => {
    const ctx = new AudioContext();
    const osc = ctx.createOscillator();
    osc.type = 'sine';
    osc.frequency.value = 1000;
    osc.connect(ctx.destination);
    osc.start();
    setTimeout(() => {
      osc.stop();
      ctx.close();
    }, 150);
  };

  const vibrate = () => {
    if (navigator.vibrate) navigator.vibrate(150);
  };

  const flashScreen = () => {
    flash.style.opacity = 1;
    setTimeout(() => {
      flash.style.opacity = 0;
    }, 100);
  };

  const showNotification = msg => {
    notification.textContent = msg;
    setTimeout(() => {
      if (notification.textContent === msg) notification.textContent = '';
    }, 3000);
  };

  const renderList = () => {
    barcodeListEl.innerHTML = '';
    barcodes.forEach(code => {
      const li = document.createElement('li');
      li.textContent = code;
      barcodeListEl.appendChild(li);
    });
  };

  const startScanner = async () => {
    if (scanning) return;

    codeReader = new ZXing.BrowserBarcodeReader();

    const devices = await codeReader.listVideoInputDevices();
    const backCamera = devices.find(d => d.label.toLowerCase().includes('back')) || devices[0];
    selectedDeviceId = backCamera.deviceId;

    const constraints = {
      video: {
        deviceId: { exact: selectedDeviceId },
        facingMode: 'environment',
        focusMode: 'continuous'
      }
    };

    try {
      scanning = true;
      startBtn.disabled = true;
      stopBtn.disabled = false;

      const stream = await navigator.mediaDevices.getUserMedia(constraints);
      video.srcObject = stream;

      await codeReader.decodeFromVideoDevice(selectedDeviceId, video, (result, err) => {
        if (result) {
          const code = result.text.trim();
          if (!barcodes.includes(code)) {
            barcodes.push(code);
            renderList();
            beep();
            vibrate();
            flashScreen();
            showNotification('Barcode gespeichert: ' + code);
          } else {
            showNotification('Bereits gescannt!');
          }
        }
        if (err && !(err instanceof ZXing.NotFoundException)) {
          console.error(err);
        }
      });

    } catch (err) {
      scanning = false;
      startBtn.disabled = false;
      stopBtn.disabled = true;
      showNotification('Fehler: ' + err.message);
    }
  };

  startBtn.addEventListener('click', startScanner);

  stopBtn.addEventListener('click', () => {
    if (!scanning) return;
    codeReader.reset();
    const tracks = video.srcObject?.getTracks();
    if (tracks) tracks.forEach(track => track.stop());
    video.srcObject = null;
    scanning = false;
    startBtn.disabled = false;
    stopBtn.disabled = true;
    showNotification('Scanner gestoppt.');
  });

  shareBtn.addEventListener('click', () => {
    if (barcodes.length === 0) return showNotification('Keine Barcodes zum Teilen.');
    const text = barcodes.join('\n');
    if (navigator.share) {
      navigator.share({ title: 'Barcodes', text })
        .then(() => showNotification('Geteilt.'))
        .catch(() => showNotification('Teilen abgebrochen.'));
    } else {
      showNotification('Teilen wird nicht unterstützt.');
    }
  });

  copyBtn.addEventListener('click', async () => {
    if (barcodes.length === 0) return showNotification('Keine Barcodes.');
    try {
      await navigator.clipboard.writeText(barcodes.join('\n'));
      showNotification('Kopiert!');
    } catch (e) {
      showNotification('Fehler beim Kopieren.');
    }
  });

  undoBtn.addEventListener('click', () => {
    if (barcodes.length > 0) {
      barcodes.pop();
      renderList();
      showNotification('Letzter Scan gelöscht.');
    }
  });

  clearBtn.addEventListener('click', () => {
    barcodes = [];
    renderList();
    showNotification('Alle gelöscht.');
  });

  renderList();
</script>
</body>
</html>
