<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>学霸全能看板 Pro</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root { --primary: #ff5e57; --secondary: #54a0ff; --bg: #f8f9fa; }
        body { font-family: -apple-system, sans-serif; background: var(--bg); margin: 0; padding: 15px; display: flex; flex-direction: column; align-items: center; }
        .card { background: white; width: 100%; max-width: 400px; border-radius: 20px; padding: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.05); margin-bottom: 15px; box-sizing: border-box; }
        #timer { font-size: 70px; font-weight: 800; color: var(--primary); text-align: center; margin: 10px 0; font-variant-numeric: tabular-nums; }
        .input-group { display: flex; flex-direction: column; gap: 10px; margin-bottom: 20px; }
        input, select { padding: 12px; border: 1px solid #ddd; border-radius: 10px; font-size: 16px; outline: none; }
        .category-row { display: flex; gap: 5px; }
        .category-row input { flex: 1; }
        .btn-add { background: var(--secondary); color: white; border: none; padding: 0 15px; border-radius: 10px; font-weight: bold; }
        .btns { display: flex; flex-direction: column; gap: 10px; width: 100%; }
        button { padding: 15px; border: none; border-radius: 12px; font-weight: bold; cursor: pointer; transition: 0.2s; }
        .btn-start { background: var(--primary); color: white; width: 100%; font-size: 18px; }
        .btn-save { background: #2ecc71; color: white; flex: 1; }
        .btn-reset { background: #eee; color: #666; flex: 0.5; }
        #calendarModal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: white; z-index: 100; overflow-y: auto; padding: 20px; box-sizing: border-box; }
        .calendar-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 5px; text-align: center; margin-top: 10px; }
        .calendar-day { padding: 12px 0; border-radius: 10px; background: #f9f9f9; font-size: 14px; position: relative; }
        .has-data::after { content: ''; position: absolute; bottom: 4px; left: 50%; transform: translateX(-50%); width: 4px; height: 4px; background: var(--primary); border-radius: 50%; }
        .today { border: 2px solid var(--primary); }
    </style>
</head>
<body>

    <div class="card">
        <div id="status" style="text-align: center; color: #666;">准备专注了吗？</div>
        <div id="timer">25:00</div>
        
        <div class="input-group">
            <input type="text" id="taskName" placeholder="任务名称 (如: 英语听力)">
            <select id="categorySelect"></select>
            <div class="category-row">
                <input type="text" id="newCatInput" placeholder="新增分类">
                <button class="btn-add" onclick="addNewCategory()">添加</button>
            </div>
        </div>

        <div class="btns">
            <button class="btn-start" id="startBtn" onclick="toggleTimer()">开始专注</button>
            <div style="display: flex; gap: 10px;">
                <button class="btn-save" onclick="saveRecord()">完成结算</button>
                <button class="btn-reset" onclick="resetTimer()">放弃</button>
            </div>
            <button onclick="openCalendar()" style="background: var(--secondary); color: white;">查看历史日历</button>
        </div>
    </div>

    <!-- 日历弹窗 -->
    <div id="calendarModal">
        <div style="display:flex; justify-content: space-between; align-items:center;">
            <button onclick="changeMonth(-1)">上月</button>
            <h3 id="monthDisplay"></h3>
            <button onclick="changeMonth(1)">下月</button>
        </div>
        <div class="calendar-grid" id="calendarGrid"></div>
        <div id="dayDetail" style="margin-top:20px; display:none; background:#f0f7ff; padding:15px; border-radius:15px;">
            <h4 id="detailDate"></h4>
            <p>总计时: <span id="totalMin" style="font-size:20px; font-weight:bold; color:var(--secondary);">0</span> 分钟</p>
            <canvas id="dayChart"></canvas>
            <div id="detailLogs"></div>
        </div>
        <button onclick="closeCalendar()" style="width:100%; margin-top:20px; background:#333; color:white; padding:15px; border-radius:12px; border:none;">关闭</button>
    </div>

    <script>
        // --- 变量定义 ---
        let timerId = null;
        let startTime = null;
        let accumulatedSeconds = 0;
        let totalSeconds = 0;
        let timeLeft = 25 * 60;
        let isOvertime = false;
        let categories = JSON.parse(localStorage.getItem('my_categories') || '["数学", "英语", "其他"]');
        let currentViewDate = new Date();
        let dayChart = null;

        // --- 计时逻辑 (修复切屏问题) ---
        function toggleTimer() {
            const btn = document.getElementById('startBtn');
            const status = document.getElementById('status');

            if (timerId) {
                // 暂停
                clearInterval(timerId);
                timerId = null;
                accumulatedSeconds += Math.floor((Date.now() - startTime) / 1000);
                btn.innerText = "继续专注";
                btn.style.background = "#2ecc71";
                status.innerText = "已暂停";
            } else {
                // 开始
                startTime = Date.now();
                btn.innerText = "暂停计时";
                btn.style.background = "#f1c40f";
                status.innerText = "正在专注中...";
                
                timerId = setInterval(() => {
                    const currentSegment = Math.floor((Date.now() - startTime) / 1000);
                    totalSeconds = accumulatedSeconds + currentSegment;

                    const baseTime = 25 * 60;
                    if (totalSeconds < baseTime) {
                        isOvertime = false;
                        timeLeft = baseTime - totalSeconds;
                        document.getElementById('timer').style.color = "var(--primary)";
                    } else {
                        isOvertime = true;
                        timeLeft = totalSeconds - baseTime;
                        document.getElementById('timer').style.color = "#2ecc71";
                        status.innerText = "目标达成！冲刺中...";
                    }
                    updateDisplay();
                }, 200); 
            }
        }

        function updateDisplay() {
            const m = Math.floor(Math.abs(timeLeft) / 60);
            const s = Math.floor(Math.abs(timeLeft) % 60);
            const prefix = isOvertime ? "+" : "";
            document.getElementById('timer').textContent = `${prefix}${m.toString().padStart(2,'0')}:${s.toString().padStart(2,'0')}`;
        }

        function saveRecord() {
            if (totalSeconds < 5) { alert("时间太短，不予记录"); return; }
            if (!confirm(`本次专注了 ${Math.floor(totalSeconds/60)} 分钟，存入历史吗？`)) return;

            const now = new Date();
            const dateStr = `${now.getFullYear()}-${now.getMonth()+1}-${now.getDate()}`;
            const history = JSON.parse(localStorage.getItem('pomodoro_v2') || '{}');
            if (!history[dateStr]) history[dateStr] = [];

            history[dateStr].push({
                cat: document.getElementById('categorySelect').value,
                task: document.getElementById('taskName').value || "专注任务",
                duration: Math.round(totalSeconds / 60 * 10) / 10,
                time: now.toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'})
            });

            localStorage.setItem('pomodoro_v2', JSON.stringify(history));
            resetAll();
            alert("保存成功！");
        }

        function resetTimer() {
            if (confirm("确定放弃本次进度吗？")) resetAll();
        }

        function resetAll() {
            clearInterval(timerId);
            timerId = null;
            startTime = null;
            accumulatedSeconds = 0;
            totalSeconds = 0;
            timeLeft = 25 * 60;
            isOvertime = false;
            updateDisplay();
            document.getElementById('timer').style.color = "var(--primary)";
            document.getElementById('startBtn').innerText = "开始专注";
            document.getElementById('startBtn').style.background = "var(--primary)";
            document.getElementById('status').innerText = "准备专注了吗？";
        }

        // --- 分类与日历 (保持之前功能) ---
        function addNewCategory() {
            const val = document.getElementById('newCatInput').value.trim();
            if (val && !categories.includes(val)) {
                categories.push(val);
                localStorage.setItem('my_categories', JSON.stringify(categories));
                renderCats();
                document.getElementById('newCatInput').value = "";
            }
        }

        function renderCats() {
            document.getElementById('categorySelect').innerHTML = categories.map(c => `<option value="${c}">${c}</option>`).join('');
        }

        function openCalendar() {
            document.getElementById('calendarModal').style.display = 'block';
            renderCalendar();
        }

        function closeCalendar() { document.getElementById('calendarModal').style.display = 'none'; }

        function renderCalendar() {
            const grid = document.getElementById('calendarGrid');
            const history = JSON.parse(localStorage.getItem('pomodoro_v2') || '{}');
            grid.innerHTML = '';
            const y = currentViewDate.getFullYear(), m = currentViewDate.getMonth();
            document.getElementById('monthDisplay').innerText = `${y}年 ${m+1}月`;
            const firstDay = new Date(y, m, 1).getDay();
            const daysInMonth = new Date(y, m+1, 0).getDate();

            for (let i = 0; i < firstDay; i++) grid.innerHTML += '<div></div>';
            for (let d = 1; d <= daysInMonth; d++) {
                const dStr = `${y}-${m+1}-${d}`;
                const hasData = history[dStr] ? 'has-data' : '';
                grid.innerHTML += `<div class="calendar-day ${hasData}" onclick="showDetail('${dStr}')">${d}</div>`;
            }
        }

        function showDetail(dStr) {
            const history = JSON.parse(localStorage.getItem('pomodoro_v2') || '{}');
            const records = history[dStr] || [];
            document.getElementById('dayDetail').style.display = 'block';
            document.getElementById('detailDate').innerText = dStr;
            
            let total = 0; const sum = {};
            records.forEach(r => { total += r.duration; sum[r.cat] = (sum[r.cat] || 0) + r.duration; });
            document.getElementById('totalMin').innerText = total.toFixed(1);

            const ctx = document.getElementById('dayChart').getContext('2d');
            if (dayChart) dayChart.destroy();
            dayChart = new Chart(ctx, {
                type: 'doughnut',
                data: { labels: Object.keys(sum), datasets: [{ data: Object.values(sum), backgroundColor: ['#ff5e57', '#54a0ff', '#1dd1a1', '#ffbe76', '#a29bfe'] }] }
            });
            document.getElementById('detailLogs').innerHTML = records.map(r => `<div style="font-size:12px; margin-top:5px;">[${r.time}] ${r.cat}: ${r.task} (${r.duration}m)</div>`).join('');
        }

        function changeMonth(dir) { currentViewDate.setMonth(currentViewDate.getMonth() + dir); renderCalendar(); }

        // 初始化
        renderCats();
        updateDisplay();
    </script>
</body>
</html>
