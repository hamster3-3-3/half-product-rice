<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <!-- 關鍵：確保響應式設計正確啟用 -->
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>米飯感官評測儀表板 | Rice Sensory Evaluation Dashboard</title>
    
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <!-- QR Code Generation Library -->
    <script src="https://cdn.jsdelivr.net/npm/qrcodejs@1.0.0/qrcode.min.js"></script>
    
    <style>
        /* Custom Scrollbar for cleaner look */
        ::-webkit-scrollbar { width: 8px; }
        ::-webkit-scrollbar-track { background: #f5f5f4; }
        ::-webkit-scrollbar-thumb { background: #d6d3d1; border-radius: 4px; }
        ::-webkit-scrollbar-thumb:hover { background: #a8a29e; }

        /* Chart Container Utilities for responsiveness */
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
            .chart-container { height: 280px; } /* Smaller height for mobile */
        }
        
        /* Style for range input (slider) */
        input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            appearance: none;
            width: 18px; /* Slightly larger touch target on mobile */
            height: 18px;
            border-radius: 50%;
            background: #0d9488; /* Teal-600 */
            cursor: pointer;
            box-shadow: 0 0 4px rgba(0, 0, 0, 0.4);
            margin-top: -8px; /* Adjust based on track height */
        }
        /* Track optimization for mobile */
        input[type="range"] {
             height: 4px;
             margin-top: 4px;
        }
    </style>
</head>
<body class="bg-stone-50 text-stone-800 font-sans leading-relaxed selection:bg-teal-100 selection:text-teal-900">

    <!-- Header & Navigation -->
    <header class="bg-white border-b border-stone-200 shadow-sm sticky top-0 z-50">
        <div class="container mx-auto px-4 py-3 flex justify-between items-center">
            <div class="flex items-center space-x-3">
                <div class="w-10 h-10 bg-teal-600 rounded-full flex items-center justify-center text-white font-bold text-xl">米</div>
                <div>
                    <h1 class="text-lg md:text-xl font-bold text-stone-900 tracking-tight">米飯感官評測</h1>
                    <p class="text-xs text-stone-500 uppercase tracking-wider hidden sm:block">Rice Blind Tasting Assessment</p>
                </div>
            </div>
            <div class="flex space-x-2 sm:space-x-3">
                
                <!-- QR Code Button -->
                <button id="showQrCodeBtn" class="text-sm text-stone-500 hover:text-teal-600 transition-colors flex items-center gap-1 p-2 rounded-lg hover:bg-stone-100">
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                        <path fill-rule="evenodd" d="M3 5a2 2 0 012-2h10a2 2 0 012 2v2H3V5zm0 4h14v7a2 2 0 01-2 2H5a2 2 0 01-2-2v-7zm7 3a1 1 0 100 2 1 1 0 000-2z" clip-rule="evenodd" />
                    </svg>
                    <span class="hidden sm:inline">顯示 QR Code</span>
                </button>

                <button id="resetBtn" class="text-sm text-stone-500 hover:text-teal-600 transition-colors flex items-center gap-1 p-2 rounded-lg hover:bg-stone-100">
                    <span>&#10227;</span> <span class="hidden sm:inline">重置</span>
                </button>
                
                <!-- CSV 匯出按鈕 -->
                <button id="exportCsvBtn" class="bg-teal-600 hover:bg-teal-700 text-white text-sm font-medium py-2 px-3 sm:px-4 rounded-lg shadow-md transition-colors flex items-center gap-1">
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
                        <path fill-rule="evenodd" d="M3 17a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm3.293-7.707a1 1 0 011.414 0L10 11.586l1.293-1.293a1 1 0 111.414 1.414l-2 2a1 1 0 01-1.414 0l-2-2a1 1 0 010-1.414z" clip-rule="evenodd" />
                        <path fill-rule="evenodd" d="M10 4a1 1 0 011 1v7a1 1 0 11-2 0V5a1 1 0 011-1z" clip-rule="evenodd" />
                    </svg>
                    <span class="hidden sm:inline">匯出 CSV</span>
                </button>
            </div>
        </div>
    </header>

    <main class="container mx-auto px-4 py-6 md:py-8 space-y-8 md:space-y-12">

        <!-- Intro Section -->
        <section class="max-w-4xl mx-auto text-center space-y-3">
            <h2 class="text-2xl md:text-3xl font-bold text-stone-800">半成品(米飯)盲測評分標準</h2>
            <p class="text-base text-stone-600 max-w-2xl mx-auto">
                請透過六大感官指標（外觀、氣味、軟硬、粘度、彈力、味道）對樣品進行全方位評估。
            </p>
        </section>

        <!-- Dashboard Grid: Mobile-first stacking, then 2 columns on desktop -->
        <div class="grid grid-cols-1 lg:grid-cols-12 gap-8">
            
            <!-- Left Column: Input Form (Priority on mobile) -->
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
                            <label for="input-assessor" class="block text-xs font-semibold text-stone-500 mb-1">評測者 (Assessor)</label>
                            <input type="text" id="input-assessor" value="評測員 C" class="w-full bg-stone-50 border border-stone-200 rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-teal-500 transition-colors">
                        </div>
                        <div>
                            <label for="input-id" class="block text-xs font-semibold text-stone-500 mb-1">樣品編號 (ID)</label>
                            <input type="text" id="input-id" value="C-03 (測試)" class="w-full bg-stone-50 border border-stone-200 rounded-lg px-3 py-2 text-sm focus:outline-none focus:border-teal-500 transition-colors">
                        </div>
                    </div>

                    <div class="space-y-5">
                        <!-- Appearance -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">外觀 (Appearance)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-appearance">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-appearance" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">完整度、色澤、光澤</p>
                        </div>

                        <!-- Aroma -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">氣味 (Aroma)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-aroma">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-aroma" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米飯香味、新鮮度</p>
                        </div>

                        <!-- Texture -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">軟硬 (Texture)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-texture">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-texture" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米粒軟硬度</p>
                        </div>

                        <!-- Viscosity -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">粘度 (Viscosity)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-viscosity">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-viscosity" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米飯黏性、抱團性</p>
                        </div>

                        <!-- Elasticity -->
                        <div>
                            <div class="flex justify-between mb-1">
                                <label class="text-sm font-medium text-stone-700">彈力 (Elasticity)</label>
                                <span class="text-sm font-bold text-teal-600" id="val-elasticity">3</span>
                            </div>
                            <input type="range" min="1" max="5" step="1" value="3" id="input-elasticity" class="w-full h-2 bg-stone-200 rounded-lg appearance-none cursor-pointer accent-teal-600">
                            <p class="text-xs text-stone-400 mt-1">米飯彈性、咀嚼感</p>
                        </div>

                        <!-- Taste -->
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

            <!-- Right Column: Visualization (Second on mobile, next to form on desktop) -->
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
        
        <!-- QR Code Modal/Section (Now with copy-to-clipboard functionality) -->
        <div id="qrCodeDisplaySection" class="fixed inset-0 bg-stone-900 bg-opacity-70 z-[99] items-center justify-center hidden" role="dialog" aria-modal="true">
            <div class="bg-white rounded-xl shadow-2xl p-6 w-11/12 max-w-sm mx-auto transform transition-all">
                <div class="flex justify-between items-center mb-4">
                    <h4 class="text-lg font-bold text-teal-800">手機掃碼填寫問卷</h4>
                    <button id="closeQrCodeBtn" class="text-stone-400 hover:text-stone-600 transition-colors text-2xl leading-none">&times;</button>
                </div>

                <!-- IMPORTANT Guidance Block -->
                <div class="bg-red-50 border-l-4 border-red-400 p-3 mb-4 rounded-md">
                    <p class="text-sm font-semibold text-red-700">【重要提示】</p>
                    <p class="text-xs text-red-600 mt-1">
                        由於此處網址可能為測試環境地址，若手機掃描後「沒有反應」或「無法開啟」，請**手動複製下方網址**，並透過電子郵件或通訊軟體傳送至您的手機上開啟。
                    </p>
                </div>

                <div class="text-center space-y-3">
                    <div id="qrcode" class="p-3 bg-white border border-stone-200 inline-block rounded-lg shadow-inner">
                        <!-- QR Code 會在此處生成 -->
                    </div>
                    
                    <p class="text-xs text-stone-500 break-words pt-2 font-mono" id="qrCodeUrlDisplay"></p>
                    
                    <button id="copyUrlBtn" class="w-full bg-teal-500 hover:bg-teal-600 text-white text-sm font-medium py-2 px-4 rounded-lg shadow-md transition-colors mt-4">
                        點擊複製網址
                    </button>
                    <p id="copyMessage" class="text-xs text-green-600 mt-2 h-4"></p>
                </div>
            </div>
        </div>


        <!-- Reference Guide (Scoring Notes) -->
        <section class="grid grid-cols-1 md:grid-cols-3 gap-6">
            <div class="bg-red-50 border border-red-100 rounded-xl p-5 hover:shadow-md transition-shadow">
                <div class="flex items-center gap-3 mb-2">
                    <span class="flex items-center justify-center w-8 h-8 rounded-full bg-red-100 text-red-600 font-bold">1</span>
                    <h4 class="font-bold text-red-800">極差 (Poor)</h4>
                </div>
                <p class="text-sm text-red-700 leading-snug">無法接受的品質或特性。例如：有異味、夾生、色澤灰暗。</p>
            </div>

            <div class="bg-yellow-50 border border-yellow-100 rounded-xl p-5 hover:shadow-md transition-shadow">
                 <div class="flex items-center gap-3 mb-2">
                    <span class="flex items-center justify-center w-8 h-8 rounded-full bg-yellow-100 text-yellow-700 font-bold">3</span>
                    <h4 class="font-bold text-yellow-800">普通 (Average)</h4>
                </div>
                <p class="text-sm text-yellow-800 leading-snug">品質尚可，達到基本食用標準，但沒有突出的優點或明顯的記憶點。</p>
            </div>

             <div class="bg-teal-50 border border-teal-100 rounded-xl p-5 hover:shadow-md transition-shadow">
                 <div class="flex items-center gap-3 mb-2">
                    <span class="flex items-center justify-center w-8 h-8 rounded-full bg-teal-100 text-teal-700 font-bold">5</span>
                    <h4 class="font-bold text-teal-800">極佳 (Excellent)</h4>
                </div>
                <p class="text-sm text-teal-800 leading-snug">頂級品質，特性完美。例如：光澤誘人、香氣濃郁、口感絕佳。</p>
            </div>
        </section>

    </main>

    <footer class="bg-stone-800 text-stone-400 py-6 mt-12">
        <div class="container mx-auto px-4 text-center text-sm">
            <p>&copy; 2025 Rice Blind Tasting Assessment System.</p>
        </div>
    </footer>

    <!-- Javascript Logic -->
    <script>
        // --- Data & State ---
        let currentSample = {
            assessor: "評測員 C",
            id: "C-03 (測試)",
            name: "當前評測",
            scores: { appearance: 3, aroma: 3, texture: 3, viscosity: 3, elasticity: 3, taste: 3 },
            total: 75,
            comments: ""
        };

        // --- Chart Initialization ---
        let radarChartInstance = null;

        function getCurrentSampleData() {
            // 從輸入欄位抓取最新的數據
            currentSample.assessor = document.getElementById('input-assessor').value.trim();
            currentSample.id = document.getElementById('input-id').value.trim();
            currentSample.total = parseInt(document.getElementById('input-score').value) || 0;
            currentSample.comments = document.getElementById('input-comments').value.trim().replace(/"/g, '""');
            
            const inputs = ['appearance', 'aroma', 'texture', 'viscosity', 'elasticity', 'taste'];
            inputs.forEach(key => {
                currentSample.scores[key] = Math.round(parseFloat(document.getElementById(`input-${key}`).value));
            });

            return currentSample;
        }

        function initCharts() {
            // Radar Chart (只顯示當前評測數據)
            const ctxRadar = document.getElementById('radarChart').getContext('2d');
            radarChartInstance = new Chart(ctxRadar, {
                type: 'radar',
                data: {
                    labels: ['外觀', '氣味', '軟硬', '粘度', '彈力', '味道'],
                    datasets: [
                        {
                            label: '當前評測',
                            data: Object.values(currentSample.scores),
                            borderColor: 'rgba(13, 148, 136, 1)', // Teal-600
                            backgroundColor: 'rgba(13, 148, 136, 0.4)',
                            borderWidth: 2,
                            pointBackgroundColor: '#fff',
                            pointBorderColor: 'rgba(13, 148, 136, 1)',
                            pointHoverBackgroundColor: '#fff',
                            pointHoverBorderColor: 'rgba(13, 148, 136, 1)'
                        }
                    ]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    scales: {
                        r: {
                            min: 0,
                            max: 5,
                            ticks: { stepSize: 1, display: false },
                            grid: { color: 'rgba(0, 0, 0, 0.05)' },
                            angleLines: { color: 'rgba(0, 0, 0, 0.05)' },
                            pointLabels: {
                                font: { size: 12, family: "'Noto Sans TC', sans-serif" },
                                color: '#44403c' // Stone-700
                            }
                        }
                    },
                    plugins: {
                        legend: { display: false },
                        tooltip: {
                            backgroundColor: 'rgba(255, 255, 255, 0.9)',
                            titleColor: '#111',
                            bodyColor: '#333',
                            borderColor: '#e5e7eb',
                            borderWidth: 1,
                            padding: 10
                        }
                    }
                }
            });
        }

        /**
         * 核心 CSV 匯出函數 - 只匯出當前輸入的一筆數據
         */
        function downloadCSV() {
            const currentData = getCurrentSampleData();
            const allSamples = [currentData]; 

            const header = [
                "評測者", "樣品編號", "外觀", "氣味", "軟硬", "粘度", "彈力", "味道", "整體分數", "樣品評價內容"
            ].join(',');
            
            const rows = allSamples.map(sample => {
                const comments = `"${sample.comments.replace(/"/g, '""')}"`;

                return [
                    sample.assessor,
                    sample.id,
                    sample.scores.appearance,
                    sample.scores.aroma,
                    sample.scores.texture,
                    sample.scores.viscosity,
                    sample.scores.elasticity,
                    sample.scores.taste,
                    sample.total,
                    comments
                ].join(',');
            });

            const csvContent = "\ufeff" + [header, ...rows].join('\n');
            
            const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
            const url = URL.createObjectURL(blob);
            
            const link = document.createElement('a');
            link.setAttribute('href', url);
            
            const date = new Date().toLocaleDateString('zh-TW', { year: 'numeric', month: '2-digit', day: '2-digit' }).replace(/\//g, '');
            link.setAttribute('download', `米飯盲測評分_${currentData.id}_${date}.csv`);
            
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }
        
        /**
         * 生成 QR Code 並在 Modal 中顯示
         */
        function generateQRCode() {
            const qrCodeContainer = document.getElementById('qrcode');
            qrCodeContainer.innerHTML = '';
            
            // 獲取當前頁面 URL
            const url = window.location.href; 
            
            document.getElementById('qrCodeUrlDisplay').innerText = url;
            const copyMessage = document.getElementById('copyMessage');
            copyMessage.innerText = ''; // 清除複製提示

            if (typeof QRCode !== 'undefined') {
                try {
                    // 檢查 QRCode 是否已生成，避免重複生成導致的錯誤
                    if (qrCodeContainer.querySelector('canvas')) {
                        qrCodeContainer.innerHTML = ''; 
                    }
                    
                    new QRCode(qrCodeContainer, {
                        text: url,
                        width: 150,
                        height: 150,
                        colorDark : "#0d9488", // Teal-600
                        colorLight : "#ffffff",
                        correctLevel : QRCode.CorrectLevel.H
                    });
                    
                } catch (error) {
                    qrCodeContainer.innerHTML = '<p class="text-sm text-red-500">QR Code 生成失敗，請手動複製下方網址。</p>';
                    console.error("QR Code generation error:", error);
                }
            } else {
                 qrCodeContainer.innerHTML = '<p class="text-sm text-red-500">QR Code 函式庫未載入，請手動複製下方網址。</p>';
            }
        }
        
        /**
         * 將網址複製到剪貼簿
         */
        function copyUrlToClipboard() {
            const url = window.location.href;
            const copyMessage = document.getElementById('copyMessage');
            
            // 由於瀏覽器安全限制，某些環境下 navigator.clipboard 可能不可用
            try {
                // 嘗試使用現代 Clipboard API
                navigator.clipboard.writeText(url).then(() => {
                    copyMessage.innerText = '網址已複製到剪貼簿！';
                    setTimeout(() => copyMessage.innerText = '', 2000);
                }).catch(() => {
                    // 如果失敗，使用 execCommand 作為後備方案
                    fallbackCopyTextToClipboard(url, copyMessage);
                });
            } catch (err) {
                // 如果 navigator.clipboard 不存在，使用後備方案
                fallbackCopyTextToClipboard(url, copyMessage);
            }
        }
        
        // 後備複製方案 (適用於舊版瀏覽器或特定 iFrame 環境)
        function fallbackCopyTextToClipboard(text, copyMessage) {
            const textArea = document.createElement("textarea");
            textArea.value = text;
            // 避免在頁面滾動或選中
            textArea.style.position = "fixed";  
            textArea.style.opacity = "0";
            document.body.appendChild(textArea);
            textArea.focus();
            textArea.select();

            try {
                document.execCommand('copy');
                copyMessage.innerText = '網址已複製到剪貼簿！ (後備)';
            } catch (err) {
                copyMessage.innerText = '複製失敗，請手動選取網址。';
            }

            document.body.removeChild(textArea);
            setTimeout(() => copyMessage.innerText = '', 2000);
        }


        function setupListeners() {
            const inputs = ['appearance', 'aroma', 'texture', 'viscosity', 'elasticity', 'taste'];
            
            inputs.forEach(key => {
                const slider = document.getElementById(`input-${key}`);
                const display = document.getElementById(`val-${key}`);
                
                slider.addEventListener('input', (e) => {
                    const val = Math.round(parseFloat(e.target.value)); 
                    display.innerText = val;
                    currentSample.scores[key] = val;
                    updateDashboard();
                });
            });

            ['input-score', 'input-assessor', 'input-id', 'input-comments'].forEach(id => {
                 document.getElementById(id).addEventListener('input', updateDashboard);
            });


            document.getElementById('resetBtn').addEventListener('click', () => {
                inputs.forEach(key => {
                    document.getElementById(`input-${key}`).value = 3;
                    document.getElementById(`val-${key}`).innerText = 3;
                });
                document.getElementById('input-score').value = 75;
                document.getElementById('input-assessor').value = "評測員 C";
                document.getElementById('input-id').value = "C-03 (測試)";
                document.getElementById('input-comments').value = "";
                updateDashboard();
            });

            document.getElementById('exportCsvBtn').addEventListener('click', downloadCSV);
            
            // QR Code Modal Handlers
            const qrCodeModal = document.getElementById('qrCodeDisplaySection');
            document.getElementById('showQrCodeBtn').addEventListener('click', () => {
                generateQRCode(); // 每次點擊都重新生成，確保 URL 最準確
                qrCodeModal.classList.remove('hidden');
                qrCodeModal.classList.add('flex');
            });
            
            document.getElementById('copyUrlBtn').addEventListener('click', copyUrlToClipboard); // 新增複製按鈕監聽器

            document.getElementById('closeQrCodeBtn').addEventListener('click', () => {
                qrCodeModal.classList.add('hidden');
                qrCodeModal.classList.remove('flex');
            });
            
            qrCodeModal.addEventListener('click', (e) => {
                if (e.target === qrCodeModal) {
                    qrCodeModal.classList.add('hidden');
                    qrCodeModal.classList.remove('flex');
                }
            });
        }

        function updateDashboard() {
            const currentData = getCurrentSampleData();
            document.getElementById('currentSampleIdDisplay').innerText = `Sample: ${currentData.id}`;
            radarChartInstance.data.datasets[0].data = Object.values(currentData.scores);
            radarChartInstance.update();
        }


        // --- Init ---
        window.addEventListener('DOMContentLoaded', () => {
            // 初始化輸入欄位
            document.getElementById('input-assessor').value = currentSample.assessor;
            document.getElementById('input-id').value = currentSample.id;
            document.getElementById('input-score').value = currentSample.total;
            
             const inputs = ['appearance', 'aroma', 'texture', 'viscosity', 'elasticity', 'taste'];
             inputs.forEach(key => {
                 document.getElementById(`input-${key}`).value = currentSample.scores[key];
                 document.getElementById(`val-${key}`).innerText = currentSample.scores[key];
             });


            initCharts();
            setupListeners();
        });

    </script>
</body>
</html>
