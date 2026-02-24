<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <title>PokéPhonics Pilot</title>
    <script src="https://aka.ms/csspeech/jsbrowserpackageraw"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        :root {
            --poke-red: #EE1515;
            --poke-blue: #3B4CCA;
            --poke-yellow: #FFDE00;
            --bg-soft: #f8fafc;
        }
        body { font-family: -apple-system, sans-serif; background: var(--bg-soft); overflow: hidden; }
        .app-container { display: grid; grid-template-columns: 280px 1fr; height: 100vh; }
        
        /* 側邊欄：iPad 專屬選單 */
        .sidebar { background: var(--poke-red); border-right: 6px solid #b91c1c; overflow-y: auto; }
        .unit-btn { width: 90%; margin: 12px auto; padding: 22px 15px; border-radius: 18px; background: white; color: #333; font-weight: 900; display: flex; align-items: center; gap: 12px; transition: 0.2s; box-shadow: 0 5px 0 #ddd; }
        .unit-btn.active { background: var(--poke-yellow); border: 4px solid var(--poke-blue); box-shadow: none; transform: translateY(4px); }

        /* 主顯示區 */
        .main-content { padding: 40px; overflow-y: auto; display: grid; grid-template-columns: repeat(auto-fill, minmax(320px, 1fr)); gap: 30px; align-content: start; }
        
        /* 巨型字卡 */
        .poke-card { background: white; border-radius: 35px; border: 12px solid var(--poke-yellow); padding: 40px; position: relative; box-shadow: 0 15px 30px rgba(0,0,0,0.08); }
        .poke-word { font-size: 6rem; font-weight: 900; color: var(--poke-blue); text-transform: uppercase; line-height: 1; }
        
        /* 寶貝球按鈕 */
        .pokeball-btn { width: 90px; height: 90px; border-radius: 50%; background: linear-gradient(to bottom, #ff0000 50%, #ffffff 50%); border: 5px solid #333; position: relative; cursor: pointer; }
        .pokeball-btn::after { content: ''; position: absolute; width: 28px; height: 28px; background: white; border: 5px solid #333; border-radius: 50%; top: 50%; left: 50%; transform: translate(-50%, -50%); }
        .recording .pokeball-btn { animation: poke-shake 0.4s infinite; }
        @keyframes poke-shake { 0% { transform: rotate(0); } 25% { transform: rotate(-10deg); } 75% { transform: rotate(10deg); } 100% { transform: rotate(0); } }

        .timer-badge { position: absolute; top: 15px; right: 15px; background: var(--poke-blue); color: white; padding: 5px 15px; border-radius: 20px; font-weight: bold; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
    </style>
</head>
<body>

    <div id="setup-overlay" class="fixed inset-0 bg-red-600 z-50 flex flex-col items-center justify-center text-white">
        <div class="text-[150px] mb-8">🔴</div>
        <h1 class="text-5xl font-black mb-10 tracking-widest">POKÉ PHONICS</h1>
        <button onclick="initApp()" class="bg-yellow-400 text-blue-900 px-20 py-6 rounded-full font-black text-3xl shadow-2xl active:scale-90 transition-transform">START TRAINING</button>
    </div>

    <div class="app-container">
        <aside class="sidebar p-4 no-scrollbar">
            <div class="text-center text-white font-black mb-8 pt-6">TRAINER MENU</div>
            <div id="unit-menu"></div>
        </aside>

        <main class="main-content no-scrollbar" id="main-display"></main>
    </div>

    <script>
        const azureKey = "76hZ0aIRPIj1XkOMXHRL8TAvTvurzg3JybPyFMJXvNKFaBI4h0i7JQQJ99CBAC3pKaRXJ3w3AAAYACOGP64k";
        const azureRegion = "eastasia";
        let isRecording = false;
        let activeUnit = 'shortA';

        // 在地資料庫預設教材
        const defaultData = {
            shortA: { label: 'Short A', icon: '🍎', data: [{word:'Cat'},{word:'Bat'},{word:'Ant'},{word:'Bag'},{word:'Cap'}] },
            longA: { label: 'Long A', icon: '🍰', data: [{word:'Cake'},{word:'Rain'},{word:'Lake'},{word:'Gate'}] },
            shortE: { label: 'Short E', icon: '🥚', data: [{word:'Bed'},{word:'Jet'},{word:'Red'}] }
        };

        let wordData = JSON.parse(localStorage.getItem('local_poke_phonics')) || defaultData;

        function initApp() {
            navigator.mediaDevices.getUserMedia({ audio: true }).then(() => {
                document.getElementById('setup-overlay').style.display = 'none';
                renderSidebar();
                renderMain();
            });
        }

        function renderSidebar() {
            const menu = document.getElementById('unit-menu');
            menu.innerHTML = Object.keys(wordData).map(key => `
                <button onclick="switchUnit('${key}')" class="unit-btn ${activeUnit === key ? 'active' : ''}">
                    <span class="text-2xl">${wordData[key].icon}</span> ${wordData[key].label}
                </button>
            `).join('');
        }

        function switchUnit(key) {
            activeUnit = key;
            renderSidebar();
            renderMain();
        }

        function playSound(text) {
            if(isRecording) return;
            const speechConfig = SpeechSDK.SpeechConfig.fromSubscription(azureKey, azureRegion);
            speechConfig.speechSynthesisVoiceName = "en-US-AvaNeural";
            const synthesizer = new SpeechSDK.SpeechSynthesizer(speechConfig);
            synthesizer.speakTextAsync(text, r => synthesizer.close());
        }

        async function startEval(word, index) {
            if(isRecording) return;
            isRecording = true;
            const card = document.getElementById(`card-${index}`);
            const fb = document.getElementById(`fb-${index}`);
            const timer = document.getElementById(`timer-${index}`);
            card.classList.add('recording');

            let timeLeft = 5;
            const countdown = setInterval(() => {
                timeLeft--;
                timer.innerText = timeLeft + 's';
                if(timeLeft <= 0) clearInterval(countdown);
            }, 1000);

            const speechConfig = SpeechSDK.SpeechConfig.fromSubscription(azureKey, azureRegion);
            const audioConfig = SpeechSDK.AudioConfig.fromDefaultMicrophoneInput();
            const pronConfig = new SpeechSDK.PronunciationAssessmentConfig(word, 
                SpeechSDK.PronunciationAssessmentGradingSystem.HundredMark, 
                SpeechSDK.PronunciationAssessmentGranularity.Phoneme);
            
            const recognizer = new SpeechSDK.SpeechRecognizer(speechConfig, audioConfig);
            pronConfig.applyTo(recognizer);

            recognizer.recognizeOnceAsync(result => {
                clearInterval(countdown);
                const assessment = SpeechSDK.PronunciationAssessmentResult.fromResult(result);
                card.classList.remove('recording');
                
                if (assessment.accuracyScore >= 80) {
                    fb.innerHTML = '<span class="text-green-600 font-black text-xl">EXCELLENT! 🌟</span>';
                } else {
                    fb.innerHTML = '<span class="text-red-500 font-black text-xl">TRY AGAIN! 💢</span>';
                }
                isRecording = false;
                recognizer.close();
            });
        }

        function renderMain() {
            const display = document.getElementById('main-display');
            display.innerHTML = wordData[activeUnit].data.map((item, i) => `
                <div class="poke-card" id="card-${i}">
                    <div class="timer-badge" id="timer-${i}">5s</div>
                    <div onclick="playSound('${item.word}')">
                        <h2 class="poke-word">${item.word}</h2>
                        <p class="text-gray-400 font-bold mt-2">TYPE: ${wordData[activeUnit].label}</p>
                    </div>
                    <div class="mt-12 flex justify-between items-center">
                        <div id="fb-${i}" class="font-black text-gray-300">GET READY!</div>
                        <div onclick="startEval('${item.word}', ${i})" class="pokeball-btn"></div>
                    </div>
                </div>
            `).join('');
        }
    </script>
</body>
</html>
