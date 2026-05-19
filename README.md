[cadence_graph.html](https://github.com/user-attachments/files/27993660/cadence_graph.html)
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>ケイデンス（RPM）計測ツール</title>
    <!-- グラフ描画用とExcel保存用の外部ライブラリ -->
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>
    <style>
        body { font-family: 'Segoe UI', sans-serif; text-align: center; background: #121212; color: #e0e0e0; margin: 0; padding: 20px; }
        .container { max-width: 600px; margin: 0 auto; background: #1e1e1e; padding: 30px; border-radius: 20px; box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
        #rpmDisplay { font-size: 5rem; font-weight: bold; color: #00ffcc; margin: 20px 0; }
        #status { margin-bottom: 20px; color: #aaa; font-size: 1.1rem; }
        button { padding: 12px 24px; font-size: 1rem; border-radius: 10px; border: none; cursor: pointer; font-weight: bold; margin: 5px; transition: 0.2s; }
        #btnConnect { background: #007bff; color: white; }
        #btnConnect:disabled { background: #333; color: #666; cursor: not-allowed; }
        #btnSave { background: #28a745; color: white; }
        #btnSave:disabled { background: #333; color: #666; cursor: not-allowed; }
        .chart-container { height: 200px; margin-top: 25px; }
    </style>
</head>
<body>

<div class="container">
    <h1>ケイデンス計測</h1>
    <div id="status">センサー待機中</div>
    
    <div id="rpmDisplay">0<span style="font-size: 1.5rem; color: #888;"> RPM</span></div>
    
    <div class="controls">
        <button id="btnConnect">センサーに接続</button>
        <button id="btnSave" disabled>Excel保存</button>
    </div>
    
    <div class="chart-container">
        <canvas id="liveChart"></canvas>
    </div>
</div>

<script>
    // --- 状態管理変数 ---
    let lastReceivedTime = Date.now();
    let startTime = null;
    let currentRPM = 0;
    let lastCrankRev = -1, lastCrankTime = -1;
    let recordedData = [];
    let chart = null;

    // --- 1. 定期チェック（切断対策など） ---
    // 2秒以上データが来なければRPM表示を0にする
    setInterval(() => {
        const now = Date.now();
        if (now - lastReceivedTime > 2000 && currentRPM !== 0) {
            currentRPM = 0;
            document.getElementById('rpmDisplay').innerHTML = `0<span style="font-size: 1.5rem; color: #888;"> RPM</span>`;
        }
    }, 1000);

    // --- 2. Bluetooth通信・データ処理 ---
    document.getElementById('btnConnect').addEventListener('click', async () => {
        try {
            const device = await navigator.bluetooth.requestDevice({ 
                filters: [{ services: ['cycling_speed_and_cadence'] }] 
            });
            const server = await device.gatt.connect();
            const service = await server.getPrimaryService('cycling_speed_and_cadence');
            const characteristic = await service.getCharacteristic('csc_measurement');
            
            await characteristic.startNotifications();
            characteristic.addEventListener('characteristicvaluechanged', handleData);
            
            document.getElementById('btnConnect').disabled = true;
            document.getElementById('status').innerText = "接続完了: " + device.name;
            
            initChart();
        } catch (e) { 
            alert("接続に失敗しました: " + e); 
        }
    });

    function handleData(event) {
        lastReceivedTime = Date.now();
        const data = event.target.value;
        let offset = 1;
        const flags = data.getUint8(0);
        
        if (flags & 0x01) offset += 6; // ホイールデータ（速度）をスキップ
        if (flags & 0x02) {            // クランクデータ（ケイデンス）がある場合
            const rev = data.getUint16(offset, true);
            const time = data.getUint16(offset + 2, true);
            
            if (lastCrankTime !== -1 && time !== lastCrankTime) {
                let rD = rev - lastCrankRev; 
                let tD = time - lastCrankTime;
                
                // 16ビットのオーバーフロー対応
                if (rD < 0) rD += 65536; 
                if (tD < 0) tD += 65536;
                
                // RPMの計算 (時間が1/1024秒単位であるため1024を掛ける)
                const rpm = Math.round((rD * 1024 * 60) / tD);
                if (rpm < 250) updateApp(rpm);
            }
            lastCrankRev = rev; 
            lastCrankTime = time;
        }
    }

    // --- 3. 画面表示・ログ記録更新 ---
    function updateApp(rpm) {
        currentRPM = rpm;
        
        // 最初に漕ぎ始めたタイミングを計測開始時間とする
        if (startTime === null && rpm > 0) startTime = Date.now(); 
        
        // 数値表示更新
        document.getElementById('rpmDisplay').innerHTML = `${rpm}<span style="font-size: 1.5rem; color: #888;"> RPM</span>`;
        
        // グラフ更新（一番左のデータを消して、最新のデータを右に追加）
        if (chart) { 
            chart.data.datasets[0].data.push(rpm); 
            chart.data.datasets[0].data.shift(); 
            chart.update('none'); 
        }
        
        // エクセル用ログデータの保存（経過時間とRPM）
        const elapsed = startTime ? parseFloat(((Date.now() - startTime) / 1000).toFixed(1)) : 0;
        recordedData.push({ "Time(s)": elapsed, "RPM": rpm });
        
        // 保存ボタンを有効化
        document.getElementById('btnSave').disabled = false;
    }

    // --- 4. グラフの初期化 ---
    function initChart() {
        const ctx = document.getElementById('liveChart').getContext('2d');
        chart = new Chart(ctx, {
            type: 'line',
            data: { 
                labels: Array(30).fill(''), // 直近30回分のダミーラベル
                datasets: [{ 
                    data: Array(30).fill(0), 
                    borderColor: '#00ffcc', 
                    borderWidth: 2, 
                    fill: false, 
                    pointRadius: 0 
                }] 
            },
            options: { 
                responsive: true, 
                maintainAspectRatio: false, 
                animation: false, 
                scales: { 
                    y: { min: 0, max: 150, ticks: { color: '#aaa' } }, 
                    x: { display: false } 
                }, 
                plugins: { legend: { display: false } } 
            }
        });
    }

    // --- 5. Excel保存機能 ---
    document.getElementById('btnSave').addEventListener('click', () => {
        if (recordedData.length === 0) return;
        const wb = XLSX.utils.book_new();
        const ws = XLSX.utils.json_to_sheet(recordedData);
        XXLSX.utils.book_append_sheet(wb, ws, "CadenceLog");
        XLSX.writeFile(wb, `CadenceLog_${new Date().getTime()}.xlsx`);
    });
</script>
</body>
</html>
