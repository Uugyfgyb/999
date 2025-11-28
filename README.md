# 999
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>老虎机：Loopy 的挑战</title>
    <!-- 引入 Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* 保持酒馆主题的样式 */
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;800&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #0d471a; /* Dark Green Felt */
            background-image: radial-gradient(#2e7d32, #0d471a);
            background-size: cover;
            background-attachment: fixed;
        }
        .slot-machine {
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.9);
            border: 12px solid #5d4037; /* Wood border */
            border-radius: 20px;
            background-color: #4e342e; /* Darker inner wood color */
        }
        .reel-container {
            background-color: #1a202c; /* Black inner reel area */
            border: 4px solid #795548;
            border-radius: 8px;
            overflow: hidden;
        }
        .reel {
            height: 100px; /* Reel display height */
            transition: transform 0.15s ease-out; /* For stopping animation */
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            font-size: 4rem; /* Symbol size */
            line-height: 100px;
            text-align: center;
        }
        .reel-item {
            height: 100px;
            width: 100%;
        }
        /* Custom Keyframes for a simple spinning effect */
        @keyframes spin-up {
            from { transform: translateY(0); }
            to { transform: translateY(-1000%); }
        }
        .spinning {
            animation: spin-up 0.1s linear infinite;
        }

        /* Responsive reel size */
        @media (min-width: 640px) {
            .reel-container {
                height: 150px;
            }
            .reel {
                font-size: 6rem;
                line-height: 150px;
                height: 150px;
            }
        }

    </style>
</head>
<body class="min-h-screen flex items-center justify-center p-4">

    <div id="game-container" class="slot-machine w-full max-w-lg p-6 md:p-10 text-amber-100 space-y-6">
        <!-- 标题区域 -->
        <header class="text-center space-y-2">
            <h1 class="text-3xl md:text-4xl font-extrabold text-yellow-300 tracking-wider uppercase">
                酒馆老虎机
            </h1>
            <p class="text-xl font-semibold text-amber-200">
                Loopy 的财富挑战
            </p>
        </header>

        <!-- 余额和投注信息 -->
        <div class="flex justify-around items-center text-center font-bold text-lg md:text-2xl p-4 rounded-lg bg-black/40 border border-yellow-700">
            <div>
                <p class="text-yellow-300">余额 (金币)</p>
                <p id="balance" class="text-3xl md:text-4xl mt-1 text-green-400">1000</p>
            </div>
            <div>
                <p class="text-yellow-300">当前投注</p>
                <p id="bet-amount" class="text-3xl md:text-4xl mt-1 text-red-400">100</p>
            </div>
        </div>

        <!-- 卷轴区域 -->
        <div id="reels-display" class="reel-container flex justify-around items-center p-2 space-x-2 md:space-x-4">
            <div id="reel-0" class="reel w-1/3">🤪</div>
            <div id="reel-1" class="reel w-1/3">🤪</div>
            <div id="reel-2" class="reel w-1/3">🤪</div>
        </div>

        <!-- 结果和消息区域 -->
        <div id="results" class="text-center space-y-4">
            <p id="message" class="text-xl md:text-2xl font-semibold text-yellow-100 p-2 rounded bg-black/50 min-h-[50px] flex items-center justify-center">
                点击“旋转”开始游戏！
            </p>
        </div>

        <!-- 控制按钮区域 -->
        <div class="flex flex-wrap justify-center pt-2">
            <button onclick="spin()" id="btn-spin" class="w-full px-8 py-4 bg-red-700 hover:bg-red-600 text-white font-bold text-2xl rounded-xl transition" style="box-shadow: 0 6px #991b1b;">
                旋转 (Spin)
            </button>
        </div>

    </div>

    <script>
        // --- 游戏状态变量 ---
        let balance = 1000;
        const betAmount = 100;
        let isSpinning = false;

        // --- 符号和赔率配置 ---
        // 符号列表 (用于随机选择)
        const symbols = ['🤪', '⭐', '💰', '7️⃣', '🍒', '🍋']; // Loopy, Star, Coin, Seven, Cherry, Lemon

        // 赔率表：[Symbol]: { 3x: multiplier, 2x: multiplier }
        const Payouts = {
            '🤪': { '3x': 100, '2x': 5 },   // Loopy - 大奖
            '⭐': { '3x': 50, '2x': 3 },    // Star - 高额
            '💰': { '3x': 20, '2x': 2 },    // Coin - 中等
            '7️⃣': { '3x': 15, '2x': 0 },    // Seven - 中等
            '🍒': { '3x': 10, '2x': 2 },    // Cherry - 低额
            '🍋': { '3x': 5, '2x': 0 },     // Lemon - 低额
        };
        
        // --- DOM 元素引用 ---
        const balanceEl = document.getElementById('balance');
        const messageEl = document.getElementById('message');
        const spinButton = document.getElementById('btn-spin');
        const reels = [
            document.getElementById('reel-0'),
            document.getElementById('reel-1'),
            document.getElementById('reel-2')
        ];

        // --- 核心游戏逻辑 ---

        /**
         * 随机选择一个符号
         * @returns {string} 符号 emoji
         */
        function getRandomSymbol() {
            const index = Math.floor(Math.random() * symbols.length);
            return symbols[index];
        }

        /**
         * 检查结果并计算奖金
         * @param {Array<string>} results - 3个卷轴的结果
         * @returns {number} 奖金金额
         */
        function checkWin(results) {
            const [r1, r2, r3] = results;
            let winnings = 0;
            let winMultiplier = 0;
            let winningSymbol = '';

            // 检查 3 个相同
            if (r1 === r2 && r2 === r3) {
                winMultiplier = Payouts[r1]['3x'];
                winningSymbol = r1;
            } 
            // 检查 2 个相同 (左对齐)
            else if (r1 === r2 && Payouts[r1]['2x'] > 0) {
                winMultiplier = Payouts[r1]['2x'];
                winningSymbol = r1;
            } 
            // 检查 2 个相同 (中对齐)
            else if (r2 === r3 && Payouts[r2]['2x'] > 0) {
                 // 确保只在 r1 != r2 时计算，防止 3x 重复计算
                if (r1 !== r2) {
                    winMultiplier = Payouts[r2]['2x'];
                    winningSymbol = r2;
                }
            } 
            
            // 计算总奖金
            if (winMultiplier > 0) {
                winnings = betAmount * winMultiplier;
                // 确保 Loopy 符号的奖金消息被高亮
                if (winningSymbol === '🤪') {
                    return { winnings, message: `🎉 恭喜！Loopy 大奖！赢得了 ${winnings} 金币！` };
                }
                return { winnings, message: `🥳 恭喜！${winningSymbol} 获胜！赢得了 ${winnings} 金币！` };
            }

            return { winnings: 0, message: '😞 很遗憾，本轮没有中奖。' };
        }

        /**
         * 更新 UI 显示
         */
        function updateUI() {
            balanceEl.textContent = balance;
            if (balance < betAmount) {
                spinButton.disabled = true;
                spinButton.classList.add('opacity-50', 'cursor-not-allowed');
                messageEl.textContent = '❌ 余额不足！请重新开始游戏。';
            } else {
                spinButton.disabled = false;
                spinButton.classList.remove('opacity-50', 'cursor-not-allowed');
            }
        }

        /**
         * 启动旋转
         */
        function spin() {
            if (isSpinning || balance < betAmount) return;

            isSpinning = true;
            balance -= betAmount;
            updateUI();
            
            spinButton.disabled = true;
            messageEl.textContent = '🎰 正在旋转中...';

            const finalResults = [];
            const stopDelay = [1500, 2500, 3500]; // 卷轴停止的延迟时间 (ms)

            // 1. 开始所有卷轴的动画
            reels.forEach(reel => {
                reel.classList.add('spinning');
                // 填充卷轴，模拟滚动效果 (非必需但增加视觉效果)
                reel.innerHTML = '';
                for (let i = 0; i < 15; i++) { // 15 个滚动符号
                    const symbol = getRandomSymbol();
                    const item = document.createElement('div');
                    item.className = 'reel-item';
                    item.textContent = symbol;
                    reel.appendChild(item);
                }
            });

            // 2. 依次停止卷轴
            for (let i = 0; i < reels.length; i++) {
                setTimeout(() => {
                    const finalSymbol = getRandomSymbol();
                    finalResults.push(finalSymbol);
                    
                    const reel = reels[i];
                    
                    // 停止动画
                    reel.classList.remove('spinning');
                    
                    // 清空并显示最终符号
                    reel.innerHTML = `<div class="reel-item">${finalSymbol}</div>`; 

                    // 最后一个卷轴停止后进行检查
                    if (i === reels.length - 1) {
                        handleSpinEnd(finalResults);
                    }
                }, stopDelay[i]);
            }
        }

        /**
         * 处理旋转结束后的逻辑
         * @param {Array<string>} results - 最终结果
         */
        function handleSpinEnd(results) {
            isSpinning = false;
            spinButton.disabled = false;
            
            const { winnings, message } = checkWin(results);

            if (winnings > 0) {
                balance += winnings;
                messageEl.className = 'text-xl md:text-2xl font-extrabold p-2 rounded bg-green-700 text-white';
            } else {
                messageEl.className = 'text-xl md:text-2xl font-semibold text-yellow-100 p-2 rounded bg-black/50';
            }
            
            messageEl.textContent = message;
            updateUI();
        }

        // 初始设置
        window.onload = updateUI;
    </script>
</body>
</html>
