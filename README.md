# Tomato-Timer-I
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>学霸番茄钟 Pro</title>
    <!-- 引入图表库 -->
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root { --primary: #ff5e57; --bg: #f8f9fa; }
        body { font-family: sans-serif; background: var(--bg); margin: 0; padding: 15px; display: flex; flex-direction: column; align-items: center; }
        .card { background: white; width: 100%; max-width: 400px; border-radius: 20px; padding: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.05); margin-bottom: 15px; text-align: center; box-sizing: border-box; }
        #timer { font-size: 70px; font-weight: 800; color: var(--primary); margin: 10px 0; font-variant-numeric: tabular-nums; }
        .input-group { display: flex; flex-direction: column; gap: 10px; margin-bottom: 20px; }
        input, select { padding: 12px; border: 1px solid #ddd; border-radius: 10px; font-size: 16px; }
        .btns { display: flex; gap: 10px; justify-content: center; }
        button { flex: 1; padding: 15px; border: none; border-radius: 12px; font-weight: bold; font-size: 16px; transition: 0.2s; }
        .btn-start { background: var(--primary); color: white; }
        .btn-reset { background: #e0e0e0; color: #666; }
        
        /* 统计区域 */
        .stats-container { width: 100%; max-width: 400px; display: none; }
        .show-stats .stats-container { display: block; }
        .tabs { margin: 15px 0; display: flex; gap: 10px; }
        .tab-btn { background: white; border: 1px solid var(--primary); color: var(--primary); padding: 5px 15px; border-radius: 20px; font-size: 12px; }
        canvas { max-height: 250px; }
    </style>
</head>
<body>

    <div class="card">
        <div id="status">准备开始？</div>
        <div id="timer">25:00</div>
        
        <div class="input-group">
            <input type="text" id="taskName" placeholder="在做什么？(如: 函数练习)">
            <select id="category">
                <option value="数学">数学 📐</option>
                <option value="英语">英语 🔤</option>
                <option value="语文">语文 ✍️</option>
                <option value="其他">其他 ☕</option>
            </select>
        </div>

        <div class="btns">
            <button class="btn-start" id="startBtn" onclick="toggleTimer()">开始专注</button>
            <button class="btn-reset" onclick="resetTimer()">重置</button>
        </div>
    </div>

    <button class="tab-btn" onclick="document.body.classList.toggle('show-stats')">📊 查看今日统计数据</button>

    <div class="stats-container card">
        <h3>今日学科分布</h3>
        <canvas id="myChart"></canvas>
        <div id="logList" style="text-align: left; margin-top: 15px; font-size: 13px; color: #666;"></div>
    </div>

    <script>
        let timeLeft = 25 * 60;
        let timerId = null;
        let myChart = null;

        // 获取当前日期字符串 YYYY-MM-DD
        const getToday = () => new Date().toISOString().split('T')[0];

        // 从本地加载数据
        function getHistory() {
            return JSON.parse(localStorage.getItem('pomodoro_v2') || '{}');
        }

        function toggleTimer() {
            if (timerId) {
                clearInterval(timerId);
                timerId = null;
                document.getElementById('startBtn').innerText = "继续";
            } else {
                document.getElementById('startBtn').innerText = "暂停";
                timerId = setInterval(() => {
                    if (timeLeft > 0) {
                        timeLeft--;
                        updateDisplay();
                    } else {
                        saveRecord();
                    }
                }, 1000);
            }
        }

        function updateDisplay() {
            const m = Math.floor(timeLeft / 60);
            const s = timeLeft % 60;
            document.getElementById('timer').textContent = `${m.toString().padStart(2,'0')}:${s.toString().padStart(2,'0')}`;
        }

        function saveRecord() {
            clearInterval(timerId);
            timerId = null;
            
            const today = getToday();
            const history = getHistory();
            if (!history[today]) history[today] = [];

            const record = {
                cat: document.getElementById('category').value,
                task: document.getElementById('taskName').value || "未命名任务",
                time: new Date().toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'})
            };

            history[today].push(record);
            localStorage.setItem('pomodoro_v2', JSON.stringify(history));
            
            alert("完成一个番茄！");
            resetTimer();
            initChart();
        }

        function resetTimer() {
            clearInterval(timerId);
            timerId = null;
            timeLeft = 25 * 60;
            updateDisplay();
            document.getElementById('startBtn').innerText = "开始专注";
        }

        // 初始化图表
        function initChart() {
            const history = getHistory()[getToday()] || [];
            const summary = {};
            
            // 统计各科数量
            history.forEach(item => {
                summary[item.cat] = (summary[item.cat] || 0) + 1;
            });

            const ctx = document.getElementById('myChart').getContext('2d');
            
            if (myChart) myChart.destroy(); // 销毁旧图表重绘

            myChart = new Chart(ctx, {
                type: 'doughnut', // 饼图/环形图
                data: {
                    labels: Object.keys(summary),
                    datasets: [{
                        data: Object.values(summary),
                        backgroundColor: ['#ff5e57', '#1dd1a1', '#ff9f43', '#54a0ff']
                    }]
                },
                options: { responsive: true, maintainAspectRatio: false }
            });

            // 更新文字日志
            document.getElementById('logList').innerHTML = history.map(h => 
                `<div>• [${h.time}] <b>${h.cat}</b>: ${h.task}</div>`
            ).reverse().join('');
        }

        updateDisplay();
        initChart();
    </script>
</body>
</html>
