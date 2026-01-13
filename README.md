<head>
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>飞书年会3D导航系统</title>
    <style>
        /* 飞书适配样式：去除边距、适配全屏、兼容触摸 */
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            background-color: #f5f7fa;
            font-family: "PingFang SC", "Hiragino Sans GB", sans-serif; /* 飞书默认字体 */
        }
        .nav-container {
            width: 100vw;
            height: 100vh;
            position: relative;
        }
        /* 姓名输入面板：飞书风格弹窗 */
        .name-panel {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            width: 85%;
            max-width: 400px;
            background: #ffffff;
            border-radius: 16px;
            padding: 32px;
            box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
            z-index: 100;
        }
        .name-panel h3 {
            font-size: 18px;
            color: #1d2129;
            margin-bottom: 24px;
            text-align: center;
        }
        .name-input {
            width: 100%;
            height: 48px;
            padding: 0 16px;
            border: 1px solid #c9cdD4;
            border-radius: 8px;
            font-size: 16px;
            margin-bottom: 24px;
            outline: none;
        }
        .name-input:focus {
            border-color: #007bff;
            box-shadow: 0 0 0 2px rgba(0, 123, 255, 0.1);
        }
        .submit-btn {
            width: 100%;
            height: 48px;
            background: #007bff;
            color: #ffffff;
            border: none;
            border-radius: 8px;
            font-size: 16px;
            cursor: pointer;
        }
        .submit-btn:active {
            background: #0069d9;
        }
        /* 3D画布：全屏渲染 */
        #glCanvas {
            width: 100%;
            height: 100%;
            display: block;
        }
        /* 导航提示文本 */
        .nav-tip {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            color: #ffffff;
            background: rgba(0, 0, 0, 0.6);
            padding: 8px 16px;
            border-radius: 20px;
            font-size: 14px;
            z-index: 99;
            display: none;
        }
    </style>
</head>
<body>
    <div class="nav-container">
        <!-- 姓名输入面板 -->
        <div class="name-panel" id="namePanel">
            <h3>请输入您的姓名</h3>
            <input type="text" class="name-input" id="userName" placeholder="例如：张三" maxlength="10">
            <button class="submit-btn" id="submitBtn">确认进入3D导航</button>
        </div>

        <!-- 3D渲染画布 -->
        <canvas id="glCanvas"></canvas>

        <!-- 导航提示 -->
        <div class="nav-tip" id="navTip">已生成导航路线，可缩放/旋转查看详情</div>
    </div>

    <!-- 引入Three.js 及 GLTFLoader（CDN方式，无需本地下载） -->
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/loaders/GLTFLoader.js"></script>

    <script>
        // 一、全局变量定义（飞书适配+3D核心对象）
        let scene, camera, renderer, clock;
        let会场3D模型 = null;
        let导航路线 = null;
        const 屏幕宽高比 = window.innerWidth / window.innerHeight;

        // 二、座位数据模板（替换为你的智能体输出数据，格式与之前JSON模板一致）
        const 座位数据库 = [
            {
                seatId: "T01-S01",
                tableId: "T01",
                position: { x: 5, y: 0, z: 8 }, // 座位3D坐标
                name: null
            },
            {
                seatId: "T01-S02",
                tableId: "T01",
                position: { x: 6, y: 0, z: 8 },
                name: null
            },
            {
                seatId: "T02-S01",
                tableId: "T02",
                position: { x: 5, y: 0, z: 12 },
                name: null
            }
        ];
        // 入口坐标（会场入口，固定）
        const 入口坐标 = { x: 0, y: 0, z: 0 };

        // 三、页面加载完成后初始化（飞书内优先执行）
        window.onload = function() {
            // 绑定姓名提交按钮事件
            document.getElementById("submitBtn").addEventListener("click", 提交姓名并匹配座位);
            // 初始化3D场景（隐藏，待姓名提交后显示）
            初始化3D场景();
            // 监听窗口大小变化，适配飞书窗口缩放
            window.addEventListener("resize", 适配飞书窗口大小);
        };

        // 四、核心功能1：姓名提交与座位匹配（飞书用户交互）
        function 提交姓名并匹配座位() {
            const 输入姓名 = document.getElementById("userName").value.trim();
            if (!输入姓名) {
                alert("请输入您的真实姓名~"); // 飞书兼容弹窗
                return;
            }

            // 1. 随机匹配一个座位（简化版，可改为按seatId匹配）
            const 可选座位 = 座位数据库.filter(item => item.name === null);
            if (可选座位.length === 0) {
                alert("所有座位已绑定，暂无空余座位~");
                return;
            }
            const 选中座位 = 可选座位[0];
            选中座位.name = 输入姓名;

            // 2. 隐藏姓名输入面板，显示3D导航
            document.getElementById("namePanel").style.display = "none";
            document.getElementById("navTip").style.display = "block";

            // 3. 生成导航路线并渲染
            生成3D导航路线(选中座位.position);
            // 4. 开始3D场景动画循环
            requestAnimationFrame(动画循环);
        }

        // 五、核心功能2：初始化3D场景（飞书WebGL适配）
        function 初始化3D场景() {
            // 1. 初始化时钟（控制动画流畅度）
            clock = new THREE.Clock();

            // 2. 创建场景
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0xf5f7fa); // 匹配飞书背景色
            scene.fog = new THREE.Fog(0xf5f7fa, 50, 100); // 抗锯齿，适配飞书低性能设备

            // 3. 创建相机（适配移动端/飞书PC端）
            camera = new THREE.PerspectiveCamera(45, 屏幕宽高比, 0.1, 200);
            camera.position.set(10, 8, 15); // 初始视角，可调整
            camera.lookAt(0, 0, 0); // 朝向入口

            // 4. 创建渲染器（飞书WebGL兼容，关闭抗锯齿提升性能）
            renderer = new THREE.WebGLRenderer({
                canvas: document.getElementById("glCanvas"),
                antialias: false, // 飞书低配设备适配，如需高清可改为true
                alpha: true
            });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio || 1); // 适配飞书Retina屏

            // 5. 添加光源（让3D模型更清晰，飞书内视觉友好）
            const 环境光 = new THREE.AmbientLight(0xffffff, 0.8);
            scene.add(环境光);
            const 方向光 = new THREE.DirectionalLight(0xffffff, 0.6);
            方向光.position.set(10, 20, 15);
            scene.add(方向光);

            // 6. 加载3D会场模型（替换为你的MediaAgent+ReconAgent生成的glb/gltf文件）
            const gltfLoader = new THREE.GLTFLoader();
            // 替换此处：将 "your_meeting_scene.glb" 改为你的3D模型路径（飞书云存储/公网地址）
            gltfLoader.load(
                "your_meeting_scene.glb",
                (gltf) => {
                    会场3D模型 = gltf.scene;
                    会场3D模型.scale.set(1, 1, 1); // 调整模型大小，适配坐标
                    scene.add(会场3D模型);
                },
                (progress) => {
                    console.log(`3D模型加载中：${Math.round(progress.loaded / progress.total * 100)}%`);
                },
                (error) => {
                    console.error("飞书内3D模型加载失败：", error);
                    alert("3D会场模型加载失败，请刷新重试~");
                }
            );
        }

        // 六、核心功能3：生成3D导航路线（红色高亮，飞书内醒目）
        function 生成3D导航路线(目标座位坐标) {
            // 1. 移除已有导航路线，避免重复
            if (导航路线) {
                scene.remove(导航路线);
                导航路线.geometry.dispose();
                导航路线.material.dispose();
            }

            // 2. 构建导航路径点（入口 → 目标座位）
            const 路径点数组 = [];
            路径点数组.push(new THREE.Vector3(入口坐标.x, 入口坐标.y, 入口坐标.z));
            // 可选：添加中间节点（通道），让路线更贴合会场
            路径点数组.push(new THREE.Vector3(2, 0, 4));
            路径点数组.push(new THREE.Vector3(目标座位坐标.x, 目标座位坐标.y, 目标座位坐标.z));

            // 3. 创建高亮导航线（飞书内醒目红色）
            const 路线几何 = new THREE.BufferGeometry().setFromPoints(路径点数组);
            const 路线材质 = new THREE.LineBasicMaterial({
                color: 0xff0000, // 红色高亮
                linewidth: 3, // 线宽（飞书WebGL兼容）
                transparent: true,
                opacity: 0.8
            });
            导航路线 = new THREE.Line(路线几何, 路线材质);
            scene.add(导航路线);

            // 4. 在座位位置添加标记点（蓝色小球，标注目标座位）
            const 座位标记几何 = new THREE.SphereGeometry(0.2, 16, 16);
            const 座位标记材质 = new THREE.MeshBasicMaterial({ color: 0x007bff });
            const 座位标记 = new THREE.Mesh(座位标记几何, 座位标记材质);
            座位标记.position.set(目标座位坐标.x, 目标座位坐标.y, 目标座位坐标.z);
            scene.add(座位标记);
        }

        // 七、核心功能4：3D场景动画循环（支持触摸/鼠标操作，飞书交互适配）
        function 动画循环() {
            // 1. 更新时钟
            const deltaTime = clock.getDelta();

            // 2. 让场景缓慢旋转（方便飞书用户查看，无需手动操作）
            if (会场3D模型) {
                会场3D模型.rotation.y += deltaTime * 0.1;
            }

            // 3. 渲染场景
            renderer.render(scene, camera);

            // 4. 循环执行（飞书内流畅运行）
            requestAnimationFrame(动画循环);
        }

        // 八、辅助功能：适配飞书窗口大小变化
        function 适配飞书窗口大小() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }
    </script>
</body>
</html>
  <script src='https://lf-package-cn.feishucdn.com/obj/feishu-static/op/fe/devtools_frontend/remote-debug-0.0.1-alpha.6.js'></script>
</head>
