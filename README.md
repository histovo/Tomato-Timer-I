<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>学霸全能看板</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root { --primary: #ff5e57; --secondary: #54a0ff; --bg: #f8f9fa; }
        body { font-family: -apple-system, sans-serif; background: var(--bg); margin: 0; padding: 15px; display: flex; flex-direction: column; align-items: center; }
        .card { background: white; width: 100%; max-width: 400px; border-radius: 20px; padding: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.05); margin-bottom: 15px; box-sizing: border-box; }
        #timer { font-size: 70px; font-weight: 800; color: var(--primary); text-align: center; margin: 10px 0; }
        
        .input-group { display: flex; flex-direction: column; gap: 10px; margin-bottom: 20px; }
        input, select { padding: 12px; border: 1px solid #ddd; border-radius: 10px; font-size: 16px; outline: none; }
        .category-row { display: flex; gap: 5px; }
        .category-row input { flex: 1; }
        .btn-add { background: var(--secondary); color: white; border: none; padding: 0 15px; border-radius: 10px; font-weight: bold; }

        .btns { display: flex; gap: 10px; }
        button { flex: 1; padding: 15px; border: none; border-radius: 12px; font-weight: bold; cursor: pointer; transition: 0.2s; }
        .btn-start { background: var(--primary); color: white; }
        .btn-cal { background: var(--secondary); color: white; }
        button:active { transform: scale(0.98); opacity: 0.9; }

        /* 日历弹窗 */
        #calendarModal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: white; z-index: 100; overflow-y: auto; padding: 20px; box-sizing: border-box; }
        .calendar-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
        .calendar-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 5px; text-align: center; }
        .calendar-day { padding: 12px 0; border-radius: 10px; position: relative; font-size: 14px; background: #f9f9f9; }
        .has-data::after { content: ''; position: absolute; bottom: 4px; left: 50%; transform: translateX(-50%); width: 4px; height: 4px; background: var(--primary); border-radius: 50%; }
        .today { border: 2px solid var(--primary); font-weight: bold; }
        .selected-day { background: var(--primary) !important; color: white; }
        
        /* 统计区块 */
        .stat-summary { background: #f0f7ff; padding: 15px; border-radius: 15px; margin: 15px 0; text-align: center; }
        .stat-number { font-size: 24px; font-weight: 800; color: var(--secondary); }
        canvas { max-width: 100%; margin: 15px 0; }
        .close-btn { background: #333; color: white; width: 100%; margin-top: 20px; padding: 15px; }
    </style>
</head>
<body>

    <div class="card">
        <div id="timer">25:00</div>
        <div class="input-group">
            <input type="text" id="taskName" placeholder="任务名称 (如: 函数练习)">
            <select id="categorySelect"></select>
            <div class="category-row">
                <input type="text" id="newCatInput" placeholder="新增学科">
                <button class="btn-add" onclick="addNewCategory()">添加</button>
            </div>
        </div>
                <div class="btns" style="display: flex; flex-direction: column; gap: 10px;">
            <button class="btn-start" id="startBtn" onclick="toggleTimer()">开始专注</button>
            <div style="display: flex; gap: 10px;">
                <button onclick="saveRecord()" style="background:#2ecc71; color:white;">结束并保存</button>
                <button onclick="resetTimer()" style="background:#eee; color:#666; flex:0.5;">重置</button>
            </div>
            <button class="btn-cal" onclick="openCalendar()">查看日历历史</button>
        </div>

    <!-- 日历弹窗内容 -->
    <div id="calendarModal">
        <div class="calendar-header">
            <button onclick="changeMonth(-1)" style="width:auto; padding:5px 15px;">上个月</button>
            <h3 id="currentMonthDisplay"></h3>
            <button onclick="changeMonth(1)" style="width:auto; padding:5px 15px;">下个月</button>
        </div>
        
        <div class="calendar-grid" id="calendarGrid"></div>

        <div id="dayDetailArea" style="display:none; margin-top:20px;">
            <div class="stat-summary">
                <div id="selectedDateText" style="font-weight:bold; margin-bottom:5px;"></div>
                今日总专注：<span class="stat-number" id="totalMinutes">0</span> 分钟
            </div>
            <canvas id="dayChart"></canvas>
            <div id="logList" style="font-size:14px; border-top:1px dashed #ddd; padding-top:10px;"></div>
        </div>
        
        <button class="close-btn" onclick="closeCalendar()">返回计时器</button>
    </div>

    <script>
                // 1. 定义状态变量
        let timeLeft = 25 * 60; 
        let timerId = null;
        let isOvertime = false;      
        let totalSeconds = 0;        
        let categories = JSON.parse(localStorage.getItem('my_categories') || '["数学", "英语"]');
        let currentViewDate = new Date();
        let dayChart = null;

        // 2. 计时与暂停逻辑
        function toggleTimer() {
            const startBtn = document.getElementById('startBtn');
            if (timerId) {
                clearInterval(timerId);
                timerId = null;
                startBtn.innerText = "继续专注";
                startBtn.style.background = "#2ecc71"; 
            } else {
                startBtn.innerText = "暂停计时";
                startBtn.style.background = "#f1c40f"; 
                timerId = setInterval(() => {
                    totalSeconds++; 
                    if (!isOvertime) {
                        if (timeLeft > 0) { timeLeft--; } 
                        else {
                            isOvertime = true;
                            document.getElementById('timer').style.color = "#2ecc71"; 
                            if (navigator.vibrate) navigator.vibrate(200);
                        }
                    } else {
                        timeLeft++; 
                    }
                    updateDisplay();
                }, 1000);
            }
        }

        // 3. 显示刷新
        function updateDisplay() {
            const m = Math.floor(Math.abs(timeLeft) / 60);
            const s = Math.floor(Math.abs(timeLeft) % 60);
            const prefix = isOvertime ? "+" : ""; 
            document.getElementById('timer').textContent = `${prefix}${m.toString().padStart(2,'0')}:${s.toString().padStart(2,'0')}`;
        }

        // 4. 手动保存结算（关键！）
        function saveRecord() {
            if (totalSeconds < 5) {
                alert("时间太短，不记录哦");
                return;
            }
            if (!confirm(`本次已专注 ${Math.floor(totalSeconds/60)} 分钟，确定结算并保存吗？`)) return;

            clearInterval(timerId);
            timerId = null;
            const now = new Date();
            const today = `${now.getFullYear()}-${now.getMonth()+1}-${now.getDate()}`;
            const history = JSON.parse(localStorage.getItem('pomodoro_v2') || '{}');
            if (!history[today]) history[today] = [];

            history[today].push({
                cat: document.getElementById('categorySelect').value,
                task: document.getElementById('taskName').value || "专注任务",
                duration: Math.round(totalSeconds / 60 * 10) / 10, 
                time: now.toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'})
            });

            localStorage.setItem('pomodoro_v2', JSON.stringify(history));
            
            // 重置所有状态回到初始
            totalSeconds = 0;
            timeLeft = 25 * 60;
            isOvertime = false;
            document.getElementById('timer').style.color = "#ff5e57";
            document.getElementById('startBtn').innerText = "开始专注";
            document.getElementById('startBtn').style.background = "#ff5e57";
            updateDisplay();
            alert("保存成功！");
        }

        // 5. 放弃当前进度
        function resetTimer() {
            if (confirm("确定要放弃本次进度吗？")) {
                clearInterval(timerId);
                timerId = null;
                totalSeconds = 0;
                timeLeft = 25 * 60;
                isOvertime = false;
                document.getElementById('timer').style.color = "#ff5e57";
                updateDisplay();
                document.getElementById('startBtn').innerText = "开始专注";
                document.getElementById('startBtn').style.background = "#ff5e57";
            }
        }

        // 6. 分类管理
        function updateCategoryDropdown() {
            document.getElementById('categorySelect').innerHTML = categories.map(c => `<option value="${c}">${c}</option>`).join('');
        }

        function addNewCategory() {
            const val = document.getElementById('newCatInput').value.trim();
            if (val && !categories.includes(val)) {
                categories.push(val);
                localStorage.setItem('my_categories', JSON.stringify(categories));
                updateCategoryDropdown();
                document.getElementById('newCatInput').value = "";
            }
        }

        // --- 日历统计系统 ---
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
            document.getElementById('currentMonthDisplay').innerText = `${y}年 ${m + 1}月`;

            const firstDay = new Date(y, m, 1).getDay();
            const daysInMonth = new Date(y, m + 1, 0).getDate();

            for (let i = 0; i < firstDay; i++) grid.innerHTML += '<div></div>';
            for (let d = 1; d <= daysInMonth; d++) {
                const dateStr = `${y}-${m+1}-${d}`;
                const hasData = history[dateStr] ? 'has-data' : '';
                const isToday = `${new Date().getFullYear()}-${new Date().getMonth()+1}-${new Date().getDate()}` === dateStr ? 'today' : '';
                grid.innerHTML += `<div class="calendar-day ${hasData} ${isToday}" onclick="showDayDetail('${dateStr}')">${d}</div>`;
            }
        }

        function showDayDetail(dateStr) {
            const history = JSON.parse(localStorage.getItem('pomodoro_v2') || '{}');
            const records = history[dateStr] || [];
            document.getElementById('dayDetailArea').style.display = 'block';
            document.getElementById('selectedDateText').innerText = dateStr;
            
            // 计算时长和分布
            let total = 0;
            const summary = {};
            records.forEach(r => {
                total += (r.duration || 25);
                summary[r.cat] = (summary[r.cat] || 0) + (r.duration || 25);
            });

            document.getElementById('totalMinutes').innerText = total;

            // 渲染列表
            document.getElementById('logList').innerHTML = records.map(r => 
                `<div style="margin-bottom:5px;">🕒 [${r.time}] <b>${r.cat}</b>: ${r.task}</div>`
            ).reverse().join('');

            // 更新饼图
            const ctx = document.getElementById('dayChart').getContext('2d');
            if (dayChart) dayChart.destroy();
            dayChart = new Chart(ctx, {
                type: 'doughnut',
                data: {
                    labels: Object.keys(summary),
                    datasets: [{
                        data: Object.values(summary),
                        backgroundColor: ['#ff5e57', '#54a0ff', '#1dd1a1', '#ffbe76', '#a29bfe']
                    }]
                },
                options: { plugins: { legend: { position: 'bottom' } } }
            });
        }

        function changeMonth(dir) { currentViewDate.setMonth(currentViewDate.getMonth() + dir); renderCalendar(); }

        updateCategoryDropdown(); updateDisplay();
    </script>
</body>
</html>
