<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>年会桌号查询</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: "微软雅黑", sans-serif;
        }
        body {
            position: relative;
            min-height: 100vh;
            /* 背景图铺满整个页面，适配不同屏幕 */
            background-image: url('meeting_room.jpg');
            background-size: cover;
            background-position: center;
            background-repeat: no-repeat;
            cursor: pointer; /* 背景添加点击光标提示 */
        }
        /* 搜索框样式：居中显示 */
        .search-box {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(255,255,255,0.95);
            padding: 20px 30px;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.15);
            z-index: 100;
            display: flex;
            flex-direction: column;
            gap: 15px;
        }
        /* 输入框提示文字样式 */
        .search-box label {
            font-size: 18px;
            color: #333;
            text-align: center;
            font-weight: 500;
        }
        #nameInput {
            padding: 12px 15px;
            border: 1px solid #ddd;
            border-radius: 8px;
            width: 300px;
            font-size: 16px;
            outline: none;
            text-align: center;
        }
        #nameInput::placeholder {
            color: #999;
        }
        #searchBtn {
            padding: 12px 0;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-size: 16px;
            font-weight: 500;
            transition: background 0.3s;
        }
        #searchBtn:hover {
            background: #0056b3;
        }
        /* 结果框样式：默认居中，点击背景后固定到左上角 */
        .result {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -60%); /* 在搜索框下方 */
            background: rgba(255,255,255,0.95);
            padding: 15px 25px;
            border-radius: 8px;
            font-size: 20px;
            display: none;
            z-index: 99;
            box-shadow: 0 2px 15px rgba(0,0,0,0.1);
            min-width: 300px;
            text-align: center;
        }
        /* 结果框固定到左上角的样式 */
        .result.fixed {
            top: 20px;
            left: 20px;
            transform: none;
            display: block !important; /* 强制显示 */
        }
        /* 吉祥物样式 */
        #mascot {
            position: absolute;
            left: 100px;   
            top: 800px;    
            width: 60px;
            height: 60px;
            transition: all 1.5s ease;
            z-index: 50;
            pointer-events: none; /* 不影响背景点击 */
        }
        #mascot img {
            width: 100%;
            height: 100%;
            object-fit: contain;
        }
    </style>
</head>
<body onclick="fixResultBox()">
    <div class="container">
        <!-- 搜索框：居中显示，提示语调整 -->
        <div class="search-box">
            <label>请输入您的名字</label>
            <input type="text" id="nameInput" placeholder="例如：张三" autocomplete="off">
            <button id="searchBtn">查询桌号</button>
        </div>
        
        <!-- 结果框：自定义提示语 -->
        <div class="result" id="resultBox"></div>
        
        <!-- 吉祥物 -->
        <div id="mascot">
            <img src="mascot.png" alt="吉祥物">
        </div>
    </div>

    <script>
        // 初始化元素
        const nameInput = document.getElementById('nameInput');
        const searchBtn = document.getElementById('searchBtn');
        const resultBox = document.getElementById('resultBox');
        const mascot = document.getElementById('mascot');

        let staffTable = {};
        let tableCoords = {};

        // 加载人员名单和坐标数据
        async function loadData() {
            try {
                const [staffRes, coordsRes] = await Promise.all([
                    fetch('staff_list.json').then(res => res.json()),
                    fetch('table_coords.json').then(res => res.json())
                ]);
                staffTable = staffRes;
                tableCoords = coordsRes;
                // 隐藏加载提示，不干扰用户
            } catch (e) {
                showResult('数据加载失败，请稍后重试', 'error');
            }
        }

        // 重置吉祥物位置
        function resetMascot() {
            mascot.style.left = '100px';
            mascot.style.top = '800px';
            // 移除结果框固定样式
            resultBox.classList.remove('fixed');
            resultBox.style.display = 'none';
        }

        // 查询桌号核心逻辑
        async function searchTable() {
            const name = nameInput.value.trim();
            if (!name) {
                showResult('请输入您的名字', 'error');
                return;
            }

            // 查询桌号
            const table = staffTable[name];
            if (!table) {
                showResult(`未找到「${name}」小伙伴的桌号信息`, 'error');
                resetMascot();
                return;
            }

            // 查询坐标
            const coords = tableCoords[table];
            if (!coords) {
                showResult(`「${name}」小伙伴的桌号${table}暂无坐标配置`, 'error');
                resetMascot();
                return;
            }

            // 显示自定义提示语
            showResult(`亲爱的${name}小伙伴，您座位的桌号为${table}`, 'success');
            // 移动吉祥物到对应桌号
            mascot.style.left = `${coords.x}px`;
            mascot.style.top = `${coords.y}px`;
        }

        // 显示结果提示
        function showResult(text, type) {
            resultBox.textContent = text;
            resultBox.style.display = 'block';
            resultBox.style.color = type === 'success' ? '#28a745' : '#dc3545';
            // 移除固定样式（每次查询重新居中）
            resultBox.classList.remove('fixed');
        }

        // 点击背景图，将结果框固定到左上角
        function fixResultBox() {
            if (resultBox.style.display === 'block') {
                resultBox.classList.add('fixed');
            }
        }

        // 绑定事件
        searchBtn.addEventListener('click', searchTable);
        // 回车查询
        nameInput.addEventListener('keydown', (e) => {
            if (e.key === 'Enter') searchTable();
        });
        // 聚焦输入框时重置
        nameInput.addEventListener('focus', resetMascot);

        // 页面加载完成后自动加载数据
        window.onload = loadData;
    </script>
</body>
</html>
