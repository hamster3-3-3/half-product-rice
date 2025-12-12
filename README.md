<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <!-- 關鍵：確保響應式設計正確啟用 -->
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>米飯感官評測系統 | 獨立部署版</title>
    
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <!-- QR Code Generation Library -->
    <script src="https://cdn.jsdelivr.net/npm/qrcodejs@1.0.0/qrcode.min.js"></script>
    
    <!-- 導入 Firebase Libraries (必須使用模組化方式) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, addDoc, onSnapshot, collection, query, serverTimestamp, setLogLevel, getDocs } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // 設置日誌級別，用於除錯
        setLogLevel('Debug');

        // =========================================================================
        // !!! 部署重要步驟 !!! 
        // 您必須將下方的 `firebaseConfig` 替換為您自己的 Firebase 專案配置物件。
        // 請登入您的 Firebase Console > 專案設定 > 網頁應用程式 > 複製配置。
        // =========================================================================
        const firebaseConfig = {
             apiKey: "YOUR_API_KEY_HERE",           // <-- 替換成您的 API Key
             authDomain: "YOUR_PROJECT_ID.firebaseapp.com", // <-- 替換成您的 Auth Domain
             projectId: "YOUR_PROJECT_ID",          // <-- 替換成您的 Project ID
             // ... [其他配置項目，如果有的話]
        };

        // 由於是外部部署，我們使用一個簡單的集合名稱。
        // 請確保您的 Firestore Security Rules 允許匿名用戶讀寫此集合。
        const collectionName = 'riceEvaluations'; 
        
        let app, db, auth;
        let userId = null;
        let isAuthReady = false;

        // 設置為 window 屬性，以便在 <script> 標籤中訪問
        window.firebaseData = {
            db: null,
            auth: null,
            userId: null,
            isAuthReady: false,
            firestore: {
                addDoc, collection, onSnapshot, query, serverTimestamp
            },
            collectionName: collectionName // 使用新的集合名稱變數
        };

        /**
         * 處理用戶認證 (在外部環境，只使用匿名登入)
         */
        async function authenticateUser() {
            if (!auth) return;

            try {
                // 在外部部署時，我們使用匿名登入，這要求您在 Firebase 啟用匿名認證
                await signInAnonymously(auth);
                console.log("Signed in anonymously successfully.");
            } catch (error) {
                console.warn("Anonymous Sign-in Failed:", error.message);
                isAuthReady = true;
                window.firebaseData.isAuthReady = isAuthReady;
            }
        }


        // 檢查配置是否為預設值
        if (firebaseConfig.apiKey === "YOUR_API_KEY_HERE") {
             console.error("Firebase 配置遺失。請用您自己的專案配置替換佔位符以啟用資料庫功能。");
             document.getElementById('displayUserId').textContent = "FIREBASE 配置錯誤";
        } else {
            app = initializeApp(firebaseConfig);
            db = getFirestore(app);
            auth = getAuth(app);
            window.firebaseData.db = db;
            window.firebaseData.auth = auth;

            // 1. 處理認證狀態 (Listener)
            onAuthStateChanged(auth, (user) => {
                isAuthReady = true; // 標記認證流程已結束
                window.firebaseData.isAuthReady = isAuthReady;

                if (user) {
                    userId = user.uid;
                    window.firebaseData.userId = userId;
                    console.log("Firebase Auth Ready. User ID:", userId);
                    
                    // 認證成功後，初始化實時數據監聽和 UI
                    window.setupRealtimeListener();
                    document.getElementById('displayUserId').textContent = userId;

                } else {
                    // 如果匿名登入失敗或沒有用戶，使用隨機 ID，但資料庫操作會失敗
                    userId = crypto.randomUUID();
                    window.firebaseData.userId = userId;
                    console.log("Authentication state changed: No user signed in. Using random UUID.", userId);
                    document.getElementById('displayUserId').textContent = userId + " (匿名失敗)";
                    window.setupRealtimeListener(); 
                }
            });

            // 2. 執行登入
            authenticateUser();
        }
    </script>
    
    <style>
        /* Custom Scrollbar for cleaner look */
        ::-webkit-scrollbar { width: 8px; }
        ::-webkit-scrollbar-track { background: #f5f5f4; }
        ::-webkit-scrollbar-thumb { background: #d6d3d1; border-radius: 4px; }
        ::-webkit-scrollbar-thumb:hover { background: #a8a29e; }

        .chart-container {
            position: relative;
            width: 100%;
            max-width: 600px; 
            margin-left: auto;
            margin-right: auto;
            height: 350px; 
            max-height: 400px;
        }
        @media (max-width: 640px) {
            .chart-container { height: 280px; } 
        }
        
        input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            appearance: none;
            width: 18px; 
            height: 18px;
            border-radius: 50%;
            background: #0d9488; 
            cursor: pointer;
            box-shadow: 0 0 4px rgba(0, 0, 0, 0.4);
            margin-top: -8px; 
        }
        input[type="range"] {
             height: 4px;
             margin-top: 4px;
        }
        .hidden { display: none !important; }
        .flex { display: flex !important; }
    </style>
</head>
<body class="bg-stone-50 text-stone-800 font-sans leading-relaxed selection:bg-teal-100 selection:text-teal-900">

    <!-- Header & Navigation -->
    <header class="bg-white border-b border-stone-200 shadow-sm sticky top-0 z-50">
        <div class="container mx-auto px-4 py-3 flex justify-between items-center">
            <div class="flex items-center space-x-3">
                <div class="w-10 h-10 bg-teal-600 rounded-full flex items-center justify-center text-white font-bold text-xl">米</div>
                <div>
                    <h1 class="text-lg md:text-xl font-bold text-stone-900 tracking-tight">米飯感官評測系統</h1>
                    <p class="text-xs text-stone-500 uppercase tracking-wider hidden sm:block">Rice Sensory Evaluation System</p>
                </div>
            </div>
            <div class="flex space-x-2 sm:space-x-3">
                
                <button id="showQrCodeBtn" class="text-sm text-stone-500 hover:text-teal-600 transition-colors flex items-center gap-1 p-2 rounded-lg hover:bg-stone-100">
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                        <path d="M3 3h2v2H3V3zM7 3h2v2H7V3zM3 7h2v2H3V7zM7 7h2v2H7V7zM11 3h2v2h-2V3zM15 3h2v2h-2V3zM11 7h2v2h-2V7zM15 7h2v2h-2V7zM3 11h2v2H3v-2zM7 11h2v2H7v-2zM3 15h2v2H3v-2zM7 15h2v2H7v-2zM11 15h2v2h-2v-2zM15 11h2v2h-2v-2zM15 15h2v2h-2v-2z"/>
                    </svg>
                    <span class="hidden sm:inline">分享連結/QR Code</span>
                </button>

                <button id="resetBtn" class="text-sm text-stone-500 hover:text-teal-600 transition-colors flex items-center gap-1 p-2 rounded-lg hover:bg-stone-100">
                    <span>&#10227;</span> <span class="hidden sm:inline">重置</span>
                </button>
                
                <!-- 提交數據按鈕 (目標: Firestore) -->
                <button id="submitBtn" class="bg-teal-600 hover:bg-teal-700 text-white text-sm font-medium py-2 px-3 sm:px-4 rounded-lg shadow-md transition-colors flex items-center gap-1">
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
                         <path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd" />
                    </svg>
                    <span id="submitBtnText">提交評測數據</span>
                </button>
            </div>
        </div>
    </header>

    <main class="container mx-auto px-4 py-6 md:py-8 space-y-8 md:space-y-12">

        <!-- User Info Bar -->
        <div class="max-w-4xl mx-auto flex flex-wrap justify-center sm:justify-between items-center bg-teal-50 border border-teal-200 rounded-lg p-3 text-sm font-medium text-teal-800">
            <span class="mb-2 sm:mb-0">當前用戶 ID (此 ID 用於資料庫分組):</span>
            <span id="displayUserId" class="font-mono text-xs sm:text-sm bg-teal-100 px-3 py-1 rounded-full shadow-inner break-all">
                等待認證...
            </span>
        </div>

        <!-- Intro Section -->
        <section class="max-w-4xl mx-auto text-center space-y-3">
            <h2 class="text-2xl md:text-3xl font-bold text-stone-800">米飯感官評測表</h2>
            <p class="text-base text-stone-600 max-w-2xl mx-auto">
                請透過六大感官指標（外觀、氣味、軟硬、粘度、彈力、味道）對樣品進行全方位評估。
            </p>
        </section>

        <!-- Dashboard Grid: Input & Chart -->
        <div class="grid grid-cols-1 lg:grid-cols-12 gap-8">
            
            <!-- Left Column: Input Form -->
            <div class="lg:col-span-5 order-first lg:order-none space-y-6">
                <div class="bg-white rounded-2xl shadow-xl border border-stone-200 p-5 sm:p-6">
                    <div class="flex items-center justify-between mb-6">
                        <h3 class="text-xl font-bold text-teal-800 flex items-center gap-2">
                            <span>&#9998;</span> 評測數據輸入
                        </h3>
                        <span class="text-xs font-mono bg-stone-100 text-stone-500 px-2 py-1 rounded" id="currentSampleIdDisplay">Sample: C-03 (測試)</span>
                    </div>

                    <!-- Meta Data -->
                    <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 mb-6">
                        <div>
                            <label for="input-assessor" class="block text-xs font-semibold text-stone-500 mb-1">評測者名稱</label>
                            <input type="text" id="input-assessor" value="評測員 A" class="w-full bg-stone-50 border border-stone-200 rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-teal-500 transition-colors" placeholder="請輸入姓名">
                        </div>
                        <div>
                            <label for="input-id" class="block text-xs font-semibold text-stone-500 mb-1">樣品編號 (ID)</label>
                            <input type="text" id="input-id" value="C-03" class="w-full bg-stone-50 border border-stone-200 rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-teal-500 transition-colors" placeholder="例如: A-01, B-02">
                        </div>
                    </div>

                    <div class="space-y-5">
                        <!-- Appearance (外觀) -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">外觀 (Appearance)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-appearance">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-appearance" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">完整度、色澤、光澤 (1:極差, 5:極佳)</p>
                        </div>

                        <!-- Aroma (氣味) -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">氣味 (Aroma)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-aroma">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-aroma" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米飯香味、新鮮度</p>
                        </div>

                        <!-- Texture (軟硬) -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">軟硬 (Texture)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-texture">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-texture" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米粒軟硬度</p>
                        </div>

                        <!-- Viscosity (粘度) -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">粘度 (Viscosity)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-viscosity">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-viscosity" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米飯黏性、抱團性</p>
                        </div>

                        <!-- Elasticity (彈力) -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">彈力 (Elasticity)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-elasticity">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-elasticity" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米飯彈性、咀嚼感</p>
                        </div>

                        <!-- Taste (味道) -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">味道 (Taste)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-taste">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-taste" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米飯甜度、鮮味</p>
                        </div>
                    </div>

                    <div class="mt-8 pt-6 border-t border-stone-200">
                        <label for="input-score" class="block text-sm font-bold text-stone-700 mb-2">整體分數 (Overall Score)</label>
                        <div class="flex items-center gap-4">
                            <input type="number" id="input-score" value="75" min="0" max="100" class="w-24 text-2xl font-bold text-center text-teal-700 border-2 border-teal-100 rounded-lg py-2 focus:outline-none focus:border-teal-500">
                            <span class="text-stone-400">/ 100</span>
                        </div>
                    </div>

                     <div class="mt-6">
                        <label for="input-comments" class="block text-sm font-bold text-stone-700 mb-2">樣品評價內容 (Comments)</label>
                        <textarea id="input-comments" class="w-full h-24 bg-stone-50 border border-stone-200 rounded-lg p-3 text-sm focus:outline-none focus:border-teal-500 resize-none" placeholder="描述優缺點、特殊風味..."></textarea>
                    </div>
                </div>
            </div>

            <!-- Right Column: Visualization -->
            <div class="lg:col-span-7 order-last lg:order-none space-y-6">
                
                <!-- Radar Chart Card -->
                <div class="bg-white rounded-2xl shadow-xl border border-stone-200 p-5 sm:p-6 flex flex-col h-full">
                    <div class="mb-4">
                        <h3 class="text-xl font-bold text-teal-800">感官輪廓分析 (Sensory Profile)</h3>
                    </div>
                    
                    <div class="flex-grow flex items-center justify-center bg-stone-50 rounded-xl p-4 border border-stone-100">
                        <div class="chart-container">
                            <canvas id="radarChart"></canvas>
                        </div>
                    </div>

                    <div class="mt-4 flex justify-center gap-2 text-center text-xs text-stone-500">
                         <div class="flex items-center justify-center gap-1">
                            <span class="w-3 h-3 rounded-full bg-teal-600 border border-teal-800"></span> 當前評測樣品
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <!-- Historical Data Table (即時更新) -->
        <section class="mt-12">
            <h3 class="text-2xl font-bold text-stone-800 mb-4 flex items-center gap-2">
                <span>&#128196;</span> 所有歷史評測數據
                <span id="loadingIndicator" class="text-sm font-normal text-stone-500 hidden">(載入中...)</span>
            </h3>
            
            <div class="bg-white rounded-xl shadow-lg border border-stone-200 overflow-x-auto">
                <table class="min-w-full divide-y divide-stone-200">
                    <thead class="bg-stone-50">
                        <tr>
                            <th class="px-6 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">樣品 ID</th>
                            <th class="px-6 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">評測者</th>
                            <th class="px-6 py-3 text-center text-xs font-medium text-stone-500 uppercase tracking-wider">外觀/氣味/軟硬/粘度/彈力/味道</th>
                            <th class="px-6 py-3 text-center text-xs font-medium text-stone-500 uppercase tracking-wider">總分</th>
                            <th class="px-6 py-3 text-left text-xs font-medium text-stone-500 uppercase tracking-wider">時間</th>
                        </tr>
                    </thead>
                    <tbody id="dataTableBody" class="bg-white divide-y divide-stone-200">
                        <!-- 數據將由 JS 填充 -->
                        <tr><td colspan="5" class="text-center py-4 text-stone-400">目前沒有數據紀錄。</td></tr>
                    </tbody>
                </table>
            </div>
        </section>


        <!-- QR Code Modal/Section (用於分享連結) -->
        <div id="qrCodeDisplaySection" class="fixed inset-0 bg-stone-900 bg-opacity-70 z-[99] flex items-center justify-center hidden" role="dialog" aria-modal="true">
            <div class="bg-white rounded-xl shadow-2xl p-6 w-11/12 max-w-sm mx-auto transform transition-all">
                <div class="flex justify-between items-center mb-4">
                    <h4 class="text-lg font-bold text-teal-800">手機掃碼填寫問卷 (分享連結)</h4>
                    <button id="closeQrCodeBtn" class="text-stone-400 hover:text-stone-600 transition-colors text-2xl leading-none">&times;</button>
                </div>

                <div id="qrcode" class="w-full max-w-[200px] h-[200px] mx-auto p-2 border border-stone-200 rounded-lg mb-4">
                    <!-- QR Code goes here -->
                </div>
                
                <p class="text-sm font-medium text-stone-700 mb-2">或直接複製以下連結：</p>
                <div class="flex">
                    <input type="text" id="shareUrlInput" readonly class="flex-grow bg-stone-100 border border-stone-300 rounded-l-lg px-3 py-2 text-sm text-stone-700 truncate" value="載入中...">
                    <button id="copyUrlBtn" class="bg-teal-600 hover:bg-teal-700 text-white text-sm font-medium py-2 px-3 rounded-r-lg transition-colors">複製</button>
                </div>
                
                <div id="copyMessage" class="mt-2 text-center text-xs text-green-600 font-semibold hidden">連結已複製到剪貼簿！</div>
                
                <div class="mt-4 bg-yellow-50 border-l-4 border-yellow-400 p-3 rounded-md">
                    <p class="text-sm font-semibold text-yellow-800">【分享提示】</p>
                    <p class="text-sm text-yellow-700 mt-1">請將此連結分享給您的同事。所有提交的數據將即時顯示在他們和您的儀表板中。</p>
                </div>
            </div>
        </div>
    </main>

    <!-- 主程式腳本 -->
    <script>
        // --- 輔助函數：將 Unix 時間戳轉換為可讀格式 ---
        function formatTimestamp(timestamp) {
            if (timestamp && timestamp.seconds) {
                const date = new Date(timestamp.seconds * 1000);
                // 為了讓時間在表格中顯示簡潔
                return date.toLocaleTimeString('zh-TW', { hour: '2-digit', minute: '2-digit', hour12: false });
            }
            return 'N/A';
        }

        // --- 感官評分項目 ---
        const SENSORY_CRITERIA = ['appearance', 'aroma', 'texture', 'viscosity', 'elasticity', 'taste'];

        // --- 雷達圖配置 ---
        const ctx = document.getElementById('radarChart').getContext('2d');
        let chartInstance = new Chart(ctx, {
            type: 'radar',
            data: {
                labels: ['外觀 (App.)', '氣味 (Aro.)', '軟硬 (Tex.)', '粘度 (Vis.)', '彈力 (Ela.)', '味道 (Tas.)'],
                datasets: [{
                    label: '當前評測分數',
                    data: SENSORY_CRITERIA.map(() => 3), // 初始值為 3
                    backgroundColor: 'rgba(13, 148, 136, 0.2)', // teal-600
                    borderColor: 'rgba(13, 148, 136, 1)',
                    pointBackgroundColor: 'rgba(13, 148, 136, 1)',
                    pointBorderColor: '#fff',
                    pointHoverBackgroundColor: '#fff',
                    pointHoverBorderColor: 'rgba(13, 148, 136, 1)',
                    borderWidth: 2,
                    tension: 0.1
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    r: {
                        angleLines: { color: 'rgba(0, 0, 0, 0.1)' },
                        grid: { color: 'rgba(0, 0, 0, 0.1)' },
                        pointLabels: {
                            font: { size: 12, family: 'sans-serif' },
                            color: '#444' 
                        },
                        suggestedMin: 1,
                        suggestedMax: 5,
                        ticks: {
                            stepSize: 1,
                            beginAtZero: true,
                            showLabelBackdrop: false,
                            backdropColor: 'rgba(255, 255, 255, 0.7)',
                            color: '#777',
                            callback: function(value) { return value; } // 僅顯示數值
                        }
                    }
                },
                plugins: {
                    legend: {
                        display: false
                    },
                    tooltip: {
                        callbacks: {
                            label: (context) => `${context.dataset.label}: ${context.formattedValue} 分`
                        }
                    }
                }
            }
        });

        // --- UI 事件監聽與數據更新 ---
        
        // 1. 綁定滑塊事件
        SENSORY_CRITERIA.forEach(criteria => {
            const rangeInput = document.getElementById(`input-${criteria}`);
            const valueSpan = document.getElementById(`val-${criteria}`);

            rangeInput.addEventListener('input', (e) => {
                const value = e.target.value;
                valueSpan.textContent = value;
                updateChart();
            });
        });

        // 2. 更新雷達圖數據
        function updateChart() {
            const newScores = SENSORY_CRITERIA.map(criteria => 
                parseInt(document.getElementById(`input-${criteria}`).value)
            );
            
            // 更新樣品ID顯示
            const sampleId = document.getElementById('input-id').value;
            document.getElementById('currentSampleIdDisplay').textContent = `Sample: ${sampleId}`;

            chartInstance.data.datasets[0].data = newScores;
            chartInstance.data.datasets[0].label = `樣品 ${sampleId} 分數`;
            chartInstance.update();
        }

        // 3. 重置按鈕
        document.getElementById('resetBtn').addEventListener('click', () => {
            // 重置所有滑塊到中間值 3
            SENSORY_CRITERIA.forEach(criteria => {
                const rangeInput = document.getElementById(`input-${criteria}`);
                const valueSpan = document.getElementById(`val-${criteria}`);
                rangeInput.value = 3;
                valueSpan.textContent = 3;
            });

            // 重置其他欄位
            document.getElementById('input-assessor').value = "評測員 A";
            document.getElementById('input-id').value = "C-03";
            document.getElementById('input-score').value = 75;
            document.getElementById('input-comments').value = "";

            updateChart();
        });

        // 4. 初始化時更新圖表
        updateChart(); 


        // --- Firestore 數據操作 ---

        /**
         * 處理提交評測數據到 Firestore
         */
        async function submitEvaluation() {
            const { db, userId, isAuthReady, firestore, collectionName } = window.firebaseData;
            if (!db || !isAuthReady) {
                alertModal("錯誤", "資料庫未準備好或未認證。請確認已配置您的 Firebase 專案。");
                return;
            }

            const assessor = document.getElementById('input-assessor').value.trim();
            const sampleId = document.getElementById('input-id').value.trim();
            const score = parseInt(document.getElementById('input-score').value);
            const comments = document.getElementById('input-comments').value.trim();

            if (!assessor || !sampleId) {
                alertModal("警告", "請填寫評測者名稱和樣品編號。");
                return;
            }
            if (isNaN(score) || score < 0 || score > 100) {
                 alertModal("警告", "整體分數必須在 0 到 100 之間。");
                return;
            }

            // 收集感官分數
            const sensoryScores = {};
            let isAllScoresValid = true;
            SENSORY_CRITERIA.forEach(criteria => {
                const value = parseInt(document.getElementById(`input-${criteria}`).value);
                sensoryScores[criteria] = value;
                if (isNaN(value) || value < 1 || value > 5) isAllScoresValid = false;
            });
            
            if (!isAllScoresValid) {
                 alertModal("警告", "所有感官評分必須在 1 到 5 之間。");
                return;
            }

            const evaluationData = {
                userId: userId, 
                assessor: assessor,
                sampleId: sampleId,
                scores: sensoryScores,
                overallScore: score,
                comments: comments,
                timestamp: firestore.serverTimestamp() // 使用伺服器時間戳
            };

            const submitBtn = document.getElementById('submitBtnText');
            submitBtn.textContent = '提交中...';
            document.getElementById('submitBtn').disabled = true;

            try {
                // 在外部部署，直接使用根集合名稱
                const docRef = await firestore.addDoc(firestore.collection(db, collectionName), evaluationData);
                
                alertModal("成功", `評測數據已提交！樣品 ID: ${sampleId}`);
                document.getElementById('resetBtn').click(); // 提交後重置表單

            } catch (e) {
                console.error("提交數據時發生錯誤: ", e);
                alertModal("錯誤", `數據提交失敗: ${e.message}。請檢查您的 Firebase 配置和 Security Rules。`);
            } finally {
                submitBtn.textContent = '提交評測數據';
                document.getElementById('submitBtn').disabled = false;
            }
        }
        
        document.getElementById('submitBtn').addEventListener('click', submitEvaluation);


        /**
         * 設置 Firestore 實時監聽器
         */
        window.setupRealtimeListener = function() {
            const { db, isAuthReady, firestore, collectionName } = window.firebaseData;
            if (!db || !isAuthReady) {
                console.warn("Realtime listener setup skipped: Firebase not ready.");
                return;
            }

            const loadingIndicator = document.getElementById('loadingIndicator');
            loadingIndicator.classList.remove('hidden');

            // 外部部署，直接使用根集合名稱
            const evaluationsColRef = firestore.collection(db, collectionName);
            const q = firestore.query(evaluationsColRef);

            // 設置 onSnapshot 實時監聽
            firestore.onSnapshot(q, (snapshot) => {
                loadingIndicator.classList.add('hidden');
                const dataTableBody = document.getElementById('dataTableBody');
                dataTableBody.innerHTML = ''; 

                let evaluations = [];
                snapshot.forEach((doc) => {
                    const data = doc.data();
                    data.id = doc.id;
                    evaluations.push(data);
                });

                // 在客戶端按時間戳降序排序 (最新提交的在最前面)
                evaluations.sort((a, b) => {
                    const timeA = a.timestamp?.seconds || 0;
                    const timeB = b.timestamp?.seconds || 0;
                    return timeB - timeA;
                });


                if (evaluations.length === 0) {
                    dataTableBody.innerHTML = '<tr><td colspan="5" class="text-center py-4 text-stone-400">目前沒有數據紀錄。</td></tr>';
                    return;
                }

                evaluations.forEach(eval => {
                    const sensoryScores = SENSORY_CRITERIA.map(c => eval.scores[c] || '-').join('/');
                    const formattedTime = formatTimestamp(eval.timestamp);
                    
                    const row = document.createElement('tr');
                    row.className = 'hover:bg-teal-50 transition-colors';
                    row.innerHTML = `
                        <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-stone-900">${eval.sampleId || 'N/A'}</td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-stone-600">${eval.assessor || 'N/A'}</td>
                        <td class="px-6 py-4 whitespace-nowrap text-center text-sm font-mono text-teal-700">${sensoryScores}</td>
                        <td class="px-6 py-4 whitespace-nowrap text-center text-sm font-bold text-teal-800">${eval.overallScore || 'N/A'}</td>
                        <td class="px-6 py-4 whitespace-nowrap text-sm text-stone-500">${formattedTime}</td>
                    `;
                    dataTableBody.appendChild(row);
                });
            }, (error) => {
                console.error("Realtime data fetch failed:", error);
                loadingIndicator.classList.add('hidden');
                document.getElementById('dataTableBody').innerHTML = `<tr><td colspan="5" class="text-center py-4 text-red-500">數據載入失敗: ${error.message}</td></tr>`;
            });
        };

        // --- 彈出視窗功能 (取代 alert/confirm) ---
        function alertModal(title, message) {
            const modalHtml = `
                <div id="customAlertModal" class="fixed inset-0 bg-stone-900 bg-opacity-70 z-[100] flex items-center justify-center">
                    <div class="bg-white rounded-xl shadow-2xl p-6 w-11/12 max-w-sm mx-auto">
                        <h4 class="text-xl font-bold ${title === '錯誤' ? 'text-red-600' : 'text-teal-600'} mb-3">${title}</h4>
                        <p class="text-stone-700 mb-6">${message}</p>
                        <div class="flex justify-end">
                            <button id="closeAlertModal" class="bg-teal-600 hover:bg-teal-700 text-white font-medium py-2 px-4 rounded-lg transition-colors">確認</button>
                        </div>
                    </div>
                </div>
            `;
            document.body.insertAdjacentHTML('beforeend', modalHtml);
            
            const modal = document.getElementById('customAlertModal');
            document.getElementById('closeAlertModal').addEventListener('click', () => {
                modal.remove();
            });
        }
        window.alertModal = alertModal; // 供內部使用

        // --- 連結分享/QR Code 功能 (與之前相同，使用 window.location.href) ---
        let qrCodeInstance = null;
        const qrCodeSection = document.getElementById('qrCodeDisplaySection');
        const showQrCodeBtn = document.getElementById('showQrCodeBtn');
        const closeQrCodeBtn = document.getElementById('closeQrCodeBtn');
        const shareUrlInput = document.getElementById('shareUrlInput');
        const copyUrlBtn = document.getElementById('copyUrlBtn');
        const copyMessage = document.getElementById('copyMessage');

        // 顯示 QR Code Modal
        showQrCodeBtn.addEventListener('click', () => {
            const currentUrl = window.location.href;
            shareUrlInput.value = currentUrl;
            qrCodeSection.classList.remove('hidden');
            qrCodeSection.classList.add('flex');
            
            // 首次顯示或網址變更時生成 QR Code
            if (!qrCodeInstance) {
                const qrCodeDiv = document.getElementById('qrcode');
                qrCodeInstance = new QRCode(qrCodeDiv, {
                    text: currentUrl,
                    width: 200,
                    height: 200,
                    colorDark : "#0d9488", // teal-600
                    colorLight : "#ffffff",
                    correctLevel : QRCode.CorrectLevel.H
                });
            } else {
                qrCodeInstance.makeCode(currentUrl);
            }
            
            // 隱藏複製成功訊息
            copyMessage.classList.add('hidden');
        });

        // 關閉 QR Code Modal
        closeQrCodeBtn.addEventListener('click', () => {
            qrCodeSection.classList.add('hidden');
            qrCodeSection.classList.remove('flex');
        });

        // 複製連結到剪貼簿
        copyUrlBtn.addEventListener('click', () => {
            shareUrlInput.select();
            shareUrlInput.setSelectionRange(0, 99999); // For mobile devices

            // 使用 document.execCommand('copy') 進行複製，以確保在 iFrame 環境中兼容
            try {
                const successful = document.execCommand('copy');
                if (successful) {
                    copyMessage.textContent = '連結已複製到剪貼簿！';
                    copyMessage.classList.remove('hidden');
                    setTimeout(() => copyMessage.classList.add('hidden'), 2000);
                } else {
                    copyMessage.textContent = '複製失敗，請手動複製。';
                    copyMessage.classList.remove('hidden');
                }
            } catch (err) {
                console.error('複製失敗:', err);
                copyMessage.textContent = '複製失敗，請手動複製。';
                copyMessage.classList.remove('hidden');
            }
        });
        
    </script>
</body>
</html>
