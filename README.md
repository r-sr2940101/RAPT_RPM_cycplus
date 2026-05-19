[cadence_graph.html](https://github.com/user-attachments/files/27993660/cadence_graph.html)
# RAPT_RPM_cycplus
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>C3 Flight Simulator - Connection Persistent</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>
    <style>
        body { font-family: 'Segoe UI', sans-serif; text-align: center; background: #121212; color: #e0e0e0; margin: 0; padding: 20px; overflow-x: hidden; }
        .flight-area {
            position: relative; width: 100%; max-width: 1000px; height: 500px; 
            margin: 0 auto 20px; background: linear-gradient(to bottom, #1e5799 0%, #7db9e8 100%);
            border-radius: 15px; overflow: hidden; border: 3px solid #333;
            box-shadow: 0 20px 50px rgba(0,0,0,0.5);
        }
        .cloud { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%) scale(0); opacity: 0; font-size: 80px; pointer-events: none; }
        #flightTimer { position: absolute; top: 20px; left: 20px; font-family: 'Courier New', monospace; font-size: 2rem; font-weight: bold; color: #ffffff; text-shadow: 2px 2px 4px rgba(0,0,0,0.8); z-index: 30; }
        #alertMsgGroup { position: absolute; top: 20px; right: 120px; text-align: right; z-index: 20; }
        .alert-text { font-weight: bold; font-size: 20px; text-shadow: 2px 2px 4px #000; margin-bottom: 5px; display: none; }
        #alertAngle { color: #ff3300; }
        #alertHeight { color: #ffcc00; }
        #windIndicator { position: absolute; top: 20px; right: 20px; background: rgba(0,0,0,0.4); padding: 10px; border-radius: 50%; width: 80px; height: 80px; border: 2px solid #00ffcc; z-index: 10; }
        #windArrow { position: absolute; top: 50%; left: 50%; width: 4px; height: 40px; background: #ff3300; transform-origin: top center; }
        #aircraftSystem { position: absolute; left: 50%; top: 50%; transform: translate(-50%, -50%); width: 600px; height: 400px; z-index: 5; }
        .aircraft-part { position: absolute; left: 50%; transform: translateX(-50%); }
        #aircraftBody { top: 5%; width: 600px; z-index: 7; }
        #propeller { top: 26%; width: 280px; z-index: 8; }
        #backBody { top: 34%; width: 220px; z-index: 6; }
        #gameOverScreen { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.85); display: none; flex-direction: column; justify-content: center; align-items: center; z-index: 1000; }
        .card { max-width: 600px; margin: 0 auto; background: #1e1e1e; padding: 20px; border-radius: 20px; box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
        #rpmDisplay { font-size: 4rem; font-weight: bold; color: #00ffcc; }
        button { padding: 12px 24px; font-size: 1rem; border-radius: 10px; border: none; cursor: pointer; font-weight: bold; margin: 5px; transition: 0.2s; }
        #btnConnect { background: #007bff; color: white; }
        #btnSave { background: #28a745; color: white; }
        #btnSave:disabled { background: #333; color: #666; cursor: not-allowed; }
        #btnRetry { background: #ffc107; color: #000; }
    </style>
</head>
<body>

<!--<div class="flight-area">
    <div id="flightTimer">00:00.0</div>
    <div id="cloudLayer"></div>
    <div id="alertMsgGroup">
        <div id="alertAngle" class="alert-text">⚠️ 傾き制限超過</div>
        <div id="alertHeight" class="alert-text">⚠️ 高度注意</div>
    </div>
    <div id="windIndicator">
        <div id="windArrow"></div>
        <div style="position: absolute; bottom: -25px; left: 50%; transform: translateX(-50%); font-size: 11px; color: #00ffcc; white-space: nowrap;">WIND: <span id="windPower">0</span></div>
    </div>
    <div id="aircraftSystem">
        <img src="aircraft_body.png" id="aircraftBody" class="aircraft-part">
        <img src="propeller.png" id="propeller" class="aircraft-part">
        <img src="backBody.png" id="backBody" class="aircraft-part">
    </div>
    <div id="gameOverScreen">
        <h1 style="color: #ff3300; font-size: 3rem; margin: 0;">MISSION FAILED</h1>
        <p id="failReason" style="color: white; font-size: 1.2rem; margin: 20px 0;"></p>
        <button id="btnRetry">センサー維持したままリトライ</button>
        <button onclick="location.reload()" style="background:none; color:#888; font-size:0.8rem; text-decoration:underline;">完全に再読み込み(接続も切れます)</button>
    </div>
</div>-->

<div class="card">
    <div id="rpmDisplay">0<span style="font-size: 1rem; color: #888;">RPM</span></div>
    <div style="margin-bottom: 10px; color: #aaa;" id="status">センサー・コントローラー待機中</div>
    <div class="controls">
        <button id="btnConnect">センサーに接続</button>
        <button id="btnSave" disabled>Excel保存</button>
    </div>
    <div style="height: 120px; margin-top: 15px;"><canvas id="liveChart"></canvas></div>
</div>

<script>
    // --- 状態管理変数 ---
    let isSimulationActive = true;
    let lastReceivedTime = Date.now();
    let startTime = null;
    let currentRPM = 0;
    let lastCrankRev = -1, lastCrankTime = -1;
    let recordedData = [];
    let chart = null;

    // 飛行パラメータ
    let planeY = 0, planeBank = 0, propellerAngle = 0;
    let windNoiseX = 0, windNoiseY = 0;
    let overAngleTime = 0;

    // 設定値
    const LIMIT_ANGLE_SOFT = 25; 
    const LIMIT_ANGLE_HARD = 50; 
    const WARN_ANGLE = 20;       
    const LIMIT_Y = 150;         
    const WARN_Y = 100;          

    // --- 1. 雲の初期化 ---
    const cloudLayer = document.getElementById('cloudLayer');
    const clouds = [];
    for (let i = 0; i < 8; i++) {
        const el = document.createElement('div');
        el.className = 'cloud'; el.innerText = "☁️";
        cloudLayer.appendChild(el);
        clouds.push({ el: el, depth: Math.random(), xAngle: (Math.random() - 0.5) * Math.PI * 0.8, yOffset: (Math.random() - 0.5) * 150, speed: Math.random() * 0.008 + 0.004 });
    }

    // --- 2. リセット機能 (ここが重要！) ---
    function resetSimulation() {
        // パラメータを初期値に戻す
        planeY = 0;
        planeBank = 0;
        overAngleTime = 0;
        windNoiseX = 0;
        windNoiseY = 0;
        startTime = null; // タイマーもリセット（漕ぎ始めたら再開）
        
        // 画面表示を戻す
        document.getElementById('gameOverScreen').style.display = 'none';
        document.getElementById('flightTimer').innerText = "00:00.0";
        document.getElementById('alertAngle').style.display = 'none';
        document.getElementById('alertHeight').style.display = 'none';
        
        // シミュレーション再開
        isSimulationActive = true;
        animate(); 
    }

    // --- 3. アニメーションループ ---
    function animate() {
        if (!isSimulationActive) return;

        const now = Date.now();
        if (now - lastReceivedTime > 2000 && currentRPM !== 0) {
            currentRPM = 0;
            document.getElementById('rpmDisplay').innerHTML = `0<span style="font-size: 1rem; color: #888;">RPM</span>`;
        }

        if (startTime) {
            const elapsed = now - startTime;
            const m = Math.floor(elapsed / 60000).toString().padStart(2, '0');
            const s = Math.floor((elapsed % 60000) / 1000).toString().padStart(2, '0');
            const ms = Math.floor((elapsed % 1000) / 100).toString();
            document.getElementById('flightTimer').innerText = `${m}:${s}.${ms}`;
        }

        const flightSpeed = 1.0 + (currentRPM * 0.04);
        clouds.forEach(c => {
            c.depth += c.speed * flightSpeed;
            if (c.depth >= 1.0) { c.depth = 0; c.xAngle = (Math.random() - 0.5) * Math.PI * 0.8; }
            const scale = 0.1 + (c.depth * c.depth * 2.5);
            const x = Math.sin(c.xAngle) * (450 * Math.pow(c.depth, 1.5));
            const y = (Math.cos(c.xAngle) * 100 * Math.pow(c.depth, 1.2)) + c.yOffset - (planeY * 0.3 * c.depth);
            const opacity = c.depth < 0.1 ? c.depth * 8 : (c.depth > 0.8 ? (1-c.depth)*4 : 0.7);
            c.el.style.transform = `translate(-50%, -50%) translate(${x}px, ${y}px) scale(${scale})`;
            c.el.style.opacity = opacity;
        });

        let windFactor = Math.max(0, 1.3 - (currentRPM * 0.008));
        windNoiseX += (Math.random() - 0.5) * 0.15 * windFactor;
        windNoiseY += (Math.random() - 0.5) * 0.15 * windFactor;
        windNoiseX *= 0.99; windNoiseY *= 0.99;

        const gps = navigator.getGamepads();
        const gp = Array.from(gps).find(p => p !== null);
        if (gp) {
            const stickX = Math.abs(gp.axes[2]) > 0.1 ? gp.axes[2] : 0;
            const stickY = Math.abs(gp.axes[3]) > 0.1 ? gp.axes[3] : 0;
            planeY += (stickY * 4) + windNoiseY;
            const stab = 0.5 + Math.abs(planeBank / 90) * 2.5;
            planeBank += (stickX * stab * 2.0) + (windNoiseX * 1.5);
            if (stickX === 0) planeBank *= 0.98;
        }

        const absB = Math.abs(planeBank);
        document.getElementById('alertAngle').style.display = absB >= WARN_ANGLE ? 'block' : 'none';
        if (absB >= LIMIT_ANGLE_HARD) return endSimulation("機体構造限界（50度超過）");
        if (absB >= LIMIT_ANGLE_SOFT) {
            overAngleTime += 1/60;
            if (overAngleTime >= 2.0) return endSimulation("失速：25度以上の傾斜が2秒継続");
        } else { overAngleTime = 0; }

        if (Math.abs(planeY) >= WARN_Y) {
            const el = document.getElementById('alertHeight');
            el.style.display = 'block';
            el.innerText = planeY > 0 ? "⚠️ 低高度注意" : "⚠️ 高高度注意";
        } else { document.getElementById('alertHeight').style.display = 'none'; }

        if (planeY >= LIMIT_Y) return endSimulation("地上激突");
        if (planeY <= -LIMIT_Y) return endSimulation("高度制限超過（失速）");

        const sys = document.getElementById('aircraftSystem');
        const prop = document.getElementById('propeller');
        if (sys) {
            planeY = Math.max(-LIMIT_Y, Math.min(LIMIT_Y, planeY));
            sys.style.top = `calc(50% + ${planeY}px)`;
            sys.style.transform = `translate(-50%, -50%) rotate(${planeBank}deg)`;
        }
        if (prop && currentRPM > 0) {
            propellerAngle += (currentRPM * 0.15);
            prop.style.transform = `translateX(-50%) rotate(${propellerAngle}deg)`;
        }

        const wP = Math.sqrt(windNoiseX**2 + windNoiseY**2);
        const wA = Math.atan2(windNoiseY, windNoiseX) * (180/Math.PI);
        document.getElementById('windArrow').style.transform = `translate(-50%, 0) rotate(${wA+90}deg)`;
        document.getElementById('windPower').innerText = (wP * 10).toFixed(1);

        requestAnimationFrame(animate);
    }

    function endSimulation(reason) {
        isSimulationActive = false;
        document.getElementById('gameOverScreen').style.display = 'flex';
        document.getElementById('failReason').innerText = reason;
        document.getElementById('rpmDisplay').innerHTML = `0<span style="font-size: 1rem; color: #888;">RPM</span>`;
    }

    // --- 4. 通信・データ処理 ---
    document.getElementById('btnConnect').addEventListener('click', async () => {
        try {
            const device = await navigator.bluetooth.requestDevice({ filters: [{ services: ['cycling_speed_and_cadence'] }] });
            const server = await device.gatt.connect();
            const service = await server.getPrimaryService('cycling_speed_and_cadence');
            const characteristic = await service.getCharacteristic('csc_measurement');
            await characteristic.startNotifications();
            characteristic.addEventListener('characteristicvaluechanged', handleData);
            document.getElementById('btnConnect').disabled = true;
            document.getElementById('status').innerText = "接続完了: " + device.name;
            initChart();
        } catch (e) { alert("接続に失敗しました: " + e); }
    });

    function handleData(event) {
        lastReceivedTime = Date.now();
        const data = event.target.value;
        let offset = 1;
        const flags = data.getUint8(0);
        if (flags & 0x01) offset += 6;
        if (flags & 0x02) {
            const rev = data.getUint16(offset, true);
            const time = data.getUint16(offset + 2, true);
            if (lastCrankTime !== -1 && time !== lastCrankTime) {
                let rD = rev - lastCrankRev; let tD = time - lastCrankTime;
                if (rD < 0) rD += 65536; if (tD < 0) tD += 65536;
                const rpm = Math.round((rD * 1024 * 60) / tD);
                if (rpm < 250) updateApp(rpm);
            }
            lastCrankRev = rev; lastCrankTime = time;
        }
    }

    function updateApp(rpm) {
        currentRPM = rpm;
        if (startTime === null && rpm > 0) startTime = Date.now(); // 漕ぎ始めたらタイマー始動
        document.getElementById('rpmDisplay').innerHTML = `${rpm}<span style="font-size: 1rem; color: #888;">RPM</span>`;
        if (chart) { chart.data.datasets[0].data.push(rpm); chart.data.datasets[0].data.shift(); chart.update('none'); }
        const elapsed = startTime ? parseFloat(((Date.now() - startTime) / 1000).toFixed(1)) : 0;
        recordedData.push({ "Time(s)": elapsed, "RPM": rpm });
        document.getElementById('btnSave').disabled = false;
    }

    function initChart() {
        const ctx = document.getElementById('liveChart').getContext('2d');
        chart = new Chart(ctx, {
            type: 'line',
            data: { labels: Array(30).fill(''), datasets: [{ data: Array(30).fill(0), borderColor: '#00ffcc', borderWidth: 2, fill: false, pointRadius: 0 }] },
            options: { responsive: true, maintainAspectRatio: false, animation: false, scales: { y: { min: 0, max: 150 }, x: { display: false } }, plugins: { legend: { display: false } } }
        });
    }

    document.getElementById('btnSave').addEventListener('click', () => {
        const wb = XLSX.utils.book_new();
        const ws = XLSX.utils.json_to_sheet(recordedData);
        XLSX.utils.book_append_sheet(wb, ws, "FlightLog");
        XLSX.writeFile(wb, `FlightLog_${new Date().getTime()}.xlsx`);
    });

    // リトライボタンにリセット機能を紐付け
    document.getElementById('btnRetry').addEventListener('click', resetSimulation);

    animate();
</script>
</body>
</html>
