<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>学霸抽卡番茄钟</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        /* --- 皮肤变量定义 --- */
        :root { 
            --primary: #ff5e57; --secondary: #54a0ff; --bg: #f8f9fa; 
            --card-bg: white; --text: #333; --border: #ddd;
        }

        body.theme-mint { 
            --primary: #3dcb9d; --secondary: #6d4c41; --bg: #e8f5e9; 
            --card-bg: #ffffff; --text: #2e7d32; --border: #c8e6c9;
        }

        body.theme-pink { 
            --primary: #ff8a8a; --secondary: #f48fb1; --bg: #fff5f8; 
            --card-bg: #ffffff; --text: #ad1457; --border: #f8bbd0;
        }

        body.theme-dark { 
            --primary: #5c7cfa; --secondary: #339af0; --bg: #101113; 
            --card-bg: #1a1b1e; --text: #c1c2c5; --border: #373a40;
        }

        body.theme-gold { 
            --primary: #fab005; --secondary: #e67e22; --bg: #fff9db; 
            --card-bg: #ffffff; --text: #862e00; --border: #ffe066;
        }

        body.theme-matrix { 
            --primary: #2ecc71; --secondary: #27ae60; --bg: #000000; 
            --card-bg: #0d0d0d; --text: #2ecc71; --border: #003300;
        }

        /* --- 基础样式 --- */
        body { font-family: -apple-system, sans-serif; background: var(--bg); color: var(--text); margin: 0; padding: 15px; display: flex; flex-direction: column; align-items: center; transition: 0.5s; }
        .card { background: var(--card-bg); width: 100%; max-width: 400px; border-radius: 24px; padding: 20px; box-shadow: 0 10px 30px rgba(0,0,0,0.08); margin-bottom: 15px; box-sizing: border-box; border: 1px solid var(--border); }
        
        #timer { font-size: 70px; font-weight: 800; color: var(--primary); text-align: center; margin: 10px 0; font-variant-numeric: tabular-nums; text-shadow: 0 4px 10px rgba(0,0,0,0.05); }
        #status { text-align: center; font-size: 14px; opacity: 0.8; margin-bottom: 10px; }

        .input-group { display: flex; flex-direction: column; gap: 10px; margin-bottom: 20px; }
        input, select { padding: 12px; border: 1px solid var(--border); border-radius: 12px; font-size: 16px; outline: none; background: var(--card-bg); color: var(--text); }
        
        button { padding: 15px; border: none; border-radius: 14px; font-weight: bold; cursor: pointer; transition: 0.2s; font-size: 14px; }
        button:active { transform: scale(0.96); }
        
        .btn-start { background: var(--primary); color: white; width: 100%; font-size: 18px; margin-bottom: 10px; }
        .btn-save { background: #2ecc71; color: white; flex: 1; }
        .btn-reset { background: var(--border); color: var(--text); flex: 0.5; }

        /* --- 抽卡区域 --- */
        .gacha-box { border: 2px dashed var(--primary); background: rgba(255,255,255,0.3); }
        .credit-num { font-size: 20px; font-weight: 800; color: var(--primary); }
        .wardrobe-list { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 10px; }
        .skin-chip { padding: 6px 12px; border-radius: 20px; font-size: 12px; border: 1px solid var(--primary); cursor: pointer; }
        .skin-chip.active { background: var(--primary); color: white; }

        /* --- 日历弹窗 --- */
        #calendarModal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: var(--bg); z-index: 100; overflow-y: auto; padding: 20px; box-sizing: border-box; }
        .calendar-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 5px; text-align: center; margin: 15px 0; }
        .calendar-day { padding: 12px 0; border-radius: 10px; background: var(--card-bg); font-size: 13px; position: relative; border: 1px solid var(--border); }
        .has-data::after { content: ''; position: absolute; bottom: 4px; left: 50%; transform: translateX(-50%); width: 4px; height: 4px; background: var(--primary); border-radius: 50%; }
    </style>
</head>
<body>

    <div class="card">
        <div id="status">今天也要加油鸭！</div>
        <div id="timer">25:00</div>
        
        <div class="input-group">
            <input type="text" id="taskName" placeholder="要做什么？(例如: 刷数学题)">
            <select id="categorySelect"></select>
        </div>

        <div class="btns">
            <button class="btn-start" id="startBtn" onclick="toggleTimer()">开始专注</button>
            <div style="display: flex; gap: 10px; margin-bottom: 10px;">
                <button class="btn-save" onclick="saveRecord()">结算保存</button>
                <button class="btn-reset" onclick="resetTimer()">放弃</button>
            </div>
            <button onclick="openCalendar()" style="width:100%; background: var(--secondary); color: white;">查看历史记录 & 统计</button>
        </div>
    </div>

    <!-- 抽卡系统卡片 -->
    <div class="card gacha-box">
        <div style="display:flex; justify-content: space-between; align-items:center;">
            <div>
                <div style="font-size:12px; opacity:0.8;">当前学分余额</div>
                <div class="credit-num" id="creditDisplay">0</div>
            </div>
            <button onclick="drawCard()" style="background:var(--primary); color:white; padding: 10px 20px;">🎲 50分抽盲盒</button>
        </div>
        <div style="margin-top:15px;">
            <div style="font-size:12px; opacity:0.6; margin-bottom:8px;">已解锁皮肤：</div>
            <div class="wardrobe-list" id="wardrobe"></div>
        </div>
    </div>

    <!-- 日历弹窗 -->
    <div id="calendarModal">
        <div style="display:flex; justify-content: space-between; align-items:center;">
            <button onclick="changeMonth(-1)" style="padding:5px 15px;">上月</button>
            <h3 id="monthDisplay"></h3>
            <button onclick="changeMonth(1)" style="padding:5px 15px;">下月</button>
        </div>
        <div class="calendar-grid" id="calendarGrid"></div>
        <div id="dayDetail" style="margin-top:20px; display:none; background:var(--card-bg); padding:15px; border-radius:15px; border:1px solid var(--border);">
            <h4 id="detailDate"></h4>
            <p>累计专注: <span id="totalMin" style="font-size:20px; font-weight:bold; color:var(--primary);">0</span> 分钟</p>
            <canvas id="dayChart"></canvas>
            <div id="detailLogs" style="margin-top:10px; font-size:13px; opacity:0.8;"></div>
        </div>
        <button onclick="closeCalendar()" style="width:100%; margin-top:20px; background:#333; color:white;">关闭日历</button>
    </div>

    <script>
        // --- 初始化变量 ---
        let timerId = null, startTime = null, accumulatedSeconds = 0, totalSeconds = 0, timeLeft = 25 * 60, isOvertime = false;
        let currentViewDate = new Date(), dayChart = null;
        
        // 游戏数据
        const ALL_THEMES = [
            { id: 'default', name: '经典西红柿', color: '#ff5e57' },
            { id: 'mint', name: '薄荷巧克力', color: '#3dcb9d' },
            { id: 'pink', name: '樱花粉色', color: '#ff8a8a' },
            { id: 'dark', name: '深海潜航', color: '#5c7cfa' },
            { id: 'gold', name: '黄金沙漠', color: '#fab005' },
            { id: 'matrix', name: '黑客帝国', color: '#2ecc71' }
        ];

        let userData = JSON.parse(localStorage.getItem('pomodoro_game_v1') || '{ "credits": 0, "unlocked": ["default"], "current": "default" }');
        let categories = JSON.parse(localStorage.getItem('my_categories') || '["数学", "英语", "语文", "物理", "其他"]');

        // --- 计时器核心逻辑 ---
        function toggleTimer() {
            const btn = document.getElementById('startBtn');
            const status = document.getElementById('status');
            if (timerId) {
                clearInterval(timerId); timerId = null;
                accumulatedSeconds += Math.floor((Date.now() - startTime) / 1000);
                btn.innerText = "继续专注"; status.innerText = "专注已暂停";
            } else {
                startTime = Date.now();
                btn.innerText = "暂停计时"; status.innerText = "进入深度专注中...";
                timerId = setInterval(() => {
                    totalSeconds = accumulatedSeconds + Math.floor((Date.now() - startTime) / 1000);
                    if (totalSeconds < 25 * 60) {
                        isOvertime = false; timeLeft = 25 * 60 - totalSeconds;
                    } else {
                        isOvertime = true; timeLeft = totalSeconds - 25 * 60;
                        status.innerText = "目标达成！正在冲刺额外学分...";
                    }
                    updateDisplay();
                }, 200);
            }
        }

        function updateDisplay() {
            const m = Math.floor(Math.abs(timeLeft) / 60), s = Math.floor(Math.abs(timeLeft) % 60);
            document.getElementById('timer').textContent = (isOvertime ? "+" : "") + `${m.toString().padStart(2,'0')}:${s.toString().padStart(2,'0')}`;
            document.getElementById('timer').style.color = isOvertime ? "var(--primary)" : "var(--primary)"; 
        }

        function saveRecord() {
            if (totalSeconds < 10) { alert("太短了，学分不涨哦！"); return; }
            if (!confirm(`本次专注 ${Math.floor(totalSeconds/60)} 分钟，结算学分吗？`)) return;

            // 1. 记录历史
            const now = new Date(), dStr = `${now.getFullYear()}-${now.getMonth()+1}-${now.getDate()}`;
            const history = JSON.parse(localStorage.getItem('pomodoro_v2') || '{}');
            if (!history[dStr]) history[dStr] = [];
            history[dStr].push({
                cat: document.getElementById('categorySelect').value,
                task: document.getElementById('taskName').value || "专注任务",
                duration: Math.round(totalSeconds / 60 * 10) / 10,
                time: now.toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'})
            });
            localStorage.setItem('pomodoro_v2', JSON.stringify(history));

            // 2. 增加学分 (1分钟 = 1学分)
            const earned = Math.floor(totalSeconds / 60);
            userData.credits += earned;
            saveGame();
            
            alert(`结算成功！获得 ${earned} 学分。`);
            resetAll();
        }

        // --- 抽卡系统 ---
        function drawCard() {
            if (userData.credits < 50) { alert("学分不足 50，再学一会吧！"); return; }
            const locked = ALL_THEMES.filter(t => !userData.unlocked.includes(t.id));
            if (locked.length === 0) { alert("全图鉴达成！你是最强学霸！"); return; }

            userData.credits -= 50;
            const prize = locked[Math.floor(Math.random() * locked.length)];
            userData.unlocked.push(prize.id);
            saveGame();
            alert(`✨ 恭喜抽中皮肤：【${prize.name}】！`);
            updateGameUI();
        }

        function changeTheme(id) {
            userData.current = id;
            saveGame();
            applyTheme(id);
            updateGameUI();
        }

        function applyTheme(id) {
            document.body.className = id === 'default' ? '' : 'theme-' + id;
        }

        function saveGame() { localStorage.setItem('pomodoro_game_v1', JSON.stringify(userData)); }

        function updateGameUI() {
            document.getElementById('creditDisplay').innerText = userData.credits;
            document.getElementById('wardrobe').innerHTML = userData.unlocked.map(tId => {
                const t = ALL_THEMES.find(x => x.id === tId);
                const active = userData.current === tId ? 'active' : '';
                return `<div class="skin-chip ${active}" onclick="changeTheme('${tId}')">${t.name}</div>`;
            }).join('');
        }

        // --- 其他功能 (日历、重置等) ---
        function resetAll() {
            clearInterval(timerId); timerId = null; startTime = null; accumulatedSeconds = 0; totalSeconds = 0; 
            timeLeft = 25 * 60; isOvertime = false; updateDisplay();
            document.getElementById('status').innerText = "今天也要加油鸭！";
            document.getElementById('startBtn').innerText = "开始专注";
        }

        function resetTimer() { if(confirm("放弃进度吗？")) resetAll(); }

        function openCalendar() { document.getElementById('calendarModal').style.display = 'block'; renderCalendar(); }
        function closeCalendar() { document.getElementById('calendarModal').style.display = 'none'; }

        function renderCalendar() {
            const grid = document.getElementById('calendarGrid'), history = JSON.parse(localStorage.getItem('pomodoro_v2') || '{}');
            grid.innerHTML = '';
            const y = currentViewDate.getFullYear(), m = currentViewDate.getMonth();
            document.getElementById('monthDisplay').innerText = `${y}年 ${m+1}月`;
            const firstDay = new Date(y, m, 1).getDay(), daysInMonth = new Date(y, m+1, 0).getDate();
            for (let i = 0; i < firstDay; i++) grid.innerHTML += '<div></div>';
            for (let d = 1; d <= daysInMonth; d++) {
                const dStr = `${y}-${m+1}-${d}`, hasData = history[dStr] ? 'has-data' : '';
                grid.innerHTML += `<div class="calendar-day ${hasData}" onclick="showDetail('${dStr}')">${d}</div>`;
            }
        }

        function showDetail(dStr) {
            const history = JSON.parse(localStorage.getItem('pomodoro_v2') || '{}'), records = history[dStr] || [];
            document.getElementById('dayDetail').style.display = 'block';
            document.getElementById('detailDate').innerText = dStr;
            let total = 0; const sum = {};
            records.forEach(r => { total += r.duration; sum[r.cat] = (sum[r.cat] || 0) + r.duration; });
            document.getElementById('totalMin').innerText = total.toFixed(1);
            const ctx = document.getElementById('dayChart').getContext('2d');
            if (dayChart) dayChart.destroy();
            dayChart = new Chart(ctx, {
                type: 'doughnut',
                data: { labels: Object.keys(sum), datasets: [{ data: Object.values(sum), backgroundColor: ['#ff5e57', '#54a0ff', '#1dd1a1', '#fab005', '#ac71ff'] }] },
                options: { plugins: { legend: { position: 'bottom', labels: { color: getComputedStyle(document.body).getPropertyValue('--text') } } } }
            });
            document.getElementById('detailLogs').innerHTML = records.map(r => `<div>• [${r.time}] ${r.cat}: ${r.task} (${r.duration}m)</div>`).join('');
        }

        function changeMonth(dir) { currentViewDate.setMonth(currentViewDate.getMonth() + dir); renderCalendar(); }

        // 启动！
        document.getElementById('categorySelect').innerHTML = categories.map(c => `<option value="${c}">${c}</option>`).join('');
        applyTheme(userData.current);
        updateGameUI();
        updateDisplay();
    </script>
</body>
</html>
