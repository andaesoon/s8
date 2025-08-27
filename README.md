# s8
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>실시간 번역기</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
        }
        .container {
            width: 95%;
            max-width: 768px;
        }
        .textarea-container {
            height: 150px;
        }
    </style>
</head>
<body class="bg-gray-100 p-4">

    <div class="container bg-white p-6 rounded-3xl shadow-xl flex flex-col gap-6">
        <h1 class="text-3xl font-bold text-center text-gray-800">실시간 번역기</h1>
        <p class="text-center text-gray-500">마이크에 대고 말하면 자동으로 번역해 드립니다.</p>

        <div class="flex flex-col md:flex-row gap-4">
            <!-- Source Language Selection -->
            <div class="flex-1">
                <label for="source-lang" class="block text-sm font-medium text-gray-700 mb-2">원래 언어</label>
                <select id="source-lang" class="w-full p-3 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500">
                    <option value="ko-KR">한국어</option>
                    <option value="zh-CN">중국어</option>
                    <option value="ru-RU">러시아어</option>
                </select>
            </div>
            
            <!-- Target Language Selection -->
            <div class="flex-1">
                <label for="target-lang" class="block text-sm font-medium text-gray-700 mb-2">번역할 언어</label>
                <select id="target-lang" class="w-full p-3 border border-gray-300 rounded-lg shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500">
                    <option value="zh-CN">중국어</option>
                    <option value="ko-KR">한국어</option>
                    <option value="ru-RU">러시아어</option>
                </select>
            </div>
        </div>

        <div class="flex flex-col gap-4">
            <div class="bg-gray-50 p-4 rounded-lg border border-gray-200 textarea-container overflow-y-auto">
                <p id="source-text" class="text-gray-800"></p>
            </div>
            <div class="bg-blue-50 p-4 rounded-lg border border-blue-200 textarea-container overflow-y-auto">
                <p id="target-text" class="text-gray-800"></p>
            </div>
        </div>

        <div class="flex justify-center items-center gap-4">
            <button id="start-btn" class="bg-blue-600 text-white font-bold py-3 px-6 rounded-full shadow-lg hover:bg-blue-700 transition duration-300">
                <i class="fas fa-play mr-2"></i> 시작
            </button>
            <button id="stop-btn" class="bg-red-600 text-white font-bold py-3 px-6 rounded-full shadow-lg hover:bg-red-700 transition duration-300 hidden">
                <i class="fas fa-stop mr-2"></i> 중지
            </button>
            <div id="loading" class="hidden">
                <i class="fas fa-spinner fa-spin text-2xl text-blue-500"></i>
            </div>
        </div>

        <p id="status-message" class="text-center text-sm text-red-500"></p>
    </div>

    <script>
        // API key will be provided by the Canvas environment.
        const apiKey = "AIzaSyCWpt1uxNSdV2B_QKV5B1G7Vd6osT3xCIc";
        
        // --- UI Element references ---
        const startBtn = document.getElementById('start-btn');
        const stopBtn = document.getElementById('stop-btn');
        const sourceLangSelect = document.getElementById('source-lang');
        const targetLangSelect = document.getElementById('target-lang');
        const sourceTextEl = document.getElementById('source-text');
        const targetTextEl = document.getElementById('target-text');
        const loadingSpinner = document.getElementById('loading');
        const statusMessageEl = document.getElementById('status-message');

        // --- Global variables for speech recognition ---
        let recognition;
        let isRecognizing = false;

        // --- Core logic on window load ---
        window.onload = function() {
            // Check for Web Speech API support
            if (!('SpeechRecognition' in window) && !('webkitSpeechRecognition' in window)) {
                statusMessageEl.textContent = "죄송합니다. 이 브라우저에서는 음성 인식이 지원되지 않습니다.";
                startBtn.disabled = true;
                return;
            }

            // Create a new SpeechRecognition instance
            const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
            recognition = new SpeechRecognition();
            
            // Set properties
            recognition.interimResults = true;
            recognition.continuous = true;
            
            // --- Event Listeners ---
            startBtn.addEventListener('click', startRecognition);
            stopBtn.addEventListener('click', stopRecognition);
            
            // Handle speech recognition results
            recognition.onresult = (event) => {
                let interimTranscript = '';
                let finalTranscript = '';

                // Loop through all results to get interim and final transcripts
                for (let i = event.resultIndex; i < event.results.length; ++i) {
                    const transcript = event.results[i][0].transcript;
                    if (event.results[i].isFinal) {
                        finalTranscript += transcript;
                    } else {
                        interimTranscript += transcript;
                    }
                }

                // Display recognized text
                sourceTextEl.textContent = finalTranscript || interimTranscript;

                // When a final result is available, send it for translation
                if (finalTranscript) {
                    handleTranslation(finalTranscript, targetLangSelect.value);
                }
            };

            // Handle recognition errors
            recognition.onerror = (event) => {
                console.error('Speech recognition error:', event.error);
                statusMessageEl.textContent = `오류 발생: ${event.error}`;
                stopRecognition();
            };

            // Restart recognition to handle continuous speech
            recognition.onend = () => {
                if (isRecognizing) {
                    recognition.start();
                }
            };
        };

        // --- Start/Stop functions ---
        function startRecognition() {
            isRecognizing = true;
            sourceTextEl.textContent = '';
            targetTextEl.textContent = '';
            statusMessageEl.textContent = '';
            
            recognition.lang = sourceLangSelect.value;
            recognition.start();

            startBtn.classList.add('hidden');
            stopBtn.classList.remove('hidden');
        }

        function stopRecognition() {
            isRecognizing = false;
            recognition.stop();
            startBtn.classList.remove('hidden');
            stopBtn.classList.add('hidden');
            hideLoading();
        }

        function showLoading() {
            loadingSpinner.classList.remove('hidden');
        }

        function hideLoading() {
            loadingSpinner.classList.add('hidden');
        }

        // --- API Call functions ---

        // Handle text translation via Gemini API with exponential backoff
        async function handleTranslation(text, targetLangCode) {
            showLoading();
            const prompt = `Translate the following text into ${targetLangCode} and provide only the translated text. Do not add any extra information, comments, or punctuation. Text: "${text}"`;
            const payload = {
                contents: [{ role: "user", parts: [{ text: prompt }] }]
            };
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=${apiKey}`;

            for (let i = 0; i < 5; i++) { // Retry up to 5 times
                try {
                    const response = await fetch(apiUrl, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });

                    if (response.ok) {
                        const result = await response.json();
                        const translatedText = result?.candidates?.[0]?.content?.parts?.[0]?.text;
                        
                        if (translatedText) {
                            targetTextEl.textContent = translatedText;
                            handleTTS(translatedText, targetLangCode);
                            return; // Success, exit the loop
                        } else {
                            statusMessageEl.textContent = "번역 결과가 없습니다.";
                            hideLoading();
                            return; // No result, exit the loop
                        }
                    } else if (response.status === 429 || response.status >= 500) {
                        // Handle rate limit or server errors with backoff
                        const delay = Math.pow(2, i) * 1000;
                        console.warn(`Translation API request failed with status ${response.status}. Retrying in ${delay}ms...`);
                        await new Promise(resolve => setTimeout(resolve, delay));
                    } else if (response.status === 403) {
                        // Handle specific API key error
                        statusMessageEl.textContent = "API 인증 오류: API 키가 올바르지 않거나 유효하지 않습니다. 잠시 후 다시 시도해 주세요.";
                        hideLoading();
                        return;
                    } else {
                        // Other client-side errors, no retry
                        const errorDetails = await response.json();
                        console.error('Translation API error details:', errorDetails);
                        statusMessageEl.textContent = `번역 API 오류: ${errorDetails.error.message}`;
                        hideLoading();
                        return;
                    }
                } catch (error) {
                    console.error('Translation error:', error);
                    statusMessageEl.textContent = "번역 중 오류가 발생했습니다. 잠시 후 다시 시도해 주세요.";
                    hideLoading();
                    return; // Fail, exit the loop
                }
            }
            statusMessageEl.textContent = "번역에 여러 번 실패했습니다. 나중에 다시 시도해 주세요.";
            hideLoading();
        }

        // Handle Text-to-Speech via Gemini API with exponential backoff
        async function handleTTS(text, langCode) {
            const voiceNames = {
                'ko-KR': 'Kore',
                'zh-CN': 'Zubenelgenubi',
                'ru-RU': 'Schedar'
            };
            const voiceName = voiceNames[langCode] || 'Kore';

            const payload = {
                contents: [{
                    parts: [{ text: text }]
                }],
                generationConfig: {
                    responseModalities: ["AUDIO"],
                    speechConfig: {
                        voiceConfig: {
                            prebuiltVoiceConfig: { voiceName: voiceName }
                        }
                    }
                },
                model: "gemini-2.5-flash-preview-tts"
            };

            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key=${apiKey}`;

            for (let i = 0; i < 5; i++) {
                try {
                    const response = await fetch(apiUrl, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });

                    if (response.ok) {
                        const result = await response.json();
                        const part = result?.candidates?.[0]?.content?.parts?.[0];
                        const audioData = part?.inlineData?.data;
                        const mimeType = part?.inlineData?.mimeType;

                        if (audioData && mimeType && mimeType.startsWith("audio/")) {
                            const audio = base64ToAudio(audioData, mimeType);
                            audio.play();
                            hideLoading();
                            return;
                        } else {
                            console.error('No audio data received from TTS API.');
                            statusMessageEl.textContent = "오디오 재생 중 오류가 발생했습니다.";
                            hideLoading();
                            return;
                        }
                    } else if (response.status === 429 || response.status >= 500) {
                        const delay = Math.pow(2, i) * 1000;
                        console.warn(`TTS API request failed with status ${response.status}. Retrying in ${delay}ms...`);
                        await new Promise(resolve => setTimeout(resolve, delay));
                    } else if (response.status === 403) {
                        statusMessageEl.textContent = "음성 변환 API 인증 오류: API 키가 올바르지 않거나 유효하지 않습니다. 잠시 후 다시 시도해 주세요.";
                        hideLoading();
                        return;
                    } else {
                        const errorDetails = await response.json();
                        console.error('TTS API error details:', errorDetails);
                        statusMessageEl.textContent = `음성 변환 API 오류: ${errorDetails.error.message}`;
                        hideLoading();
                        return;
                    }
                } catch (error) {
                    console.error('TTS error:', error);
                    statusMessageEl.textContent = "음성 변환 중 오류가 발생했습니다.";
                    hideLoading();
                    return;
                }
            }
            statusMessageEl.textContent = "음성 변환에 여러 번 실패했습니다. 나중에 다시 시도해 주세요.";
            hideLoading();
        }
        
        // --- Helper functions for audio conversion ---

        function base64ToAudio(base64Data, mimeType) {
            // Extract sample rate from mime type (e.g., audio/L16;rate=16000)
            const match = mimeType.match(/rate=(\d+)/);
            const sampleRate = match ? parseInt(match[1], 10) : 16000;
            
            const pcmData = base64ToArrayBuffer(base64Data);
            const pcm16 = new Int16Array(pcmData);
            const wavBlob = pcmToWav(pcm16, sampleRate);
            const audioUrl = URL.createObjectURL(wavBlob);
            return new Audio(audioUrl);
        }

        function base64ToArrayBuffer(base64) {
            const binaryString = atob(base64);
            const len = binaryString.length;
            const bytes = new Uint8Array(len);
            for (let i = 0; i < len; i++) {
                bytes[i] = binaryString.charCodeAt(i);
            }
            return bytes.buffer;
        }

        // Converts raw PCM data to a playable WAV blob
        function pcmToWav(pcm16, sampleRate) {
            const dataLength = pcm16.length * 2; // 16-bit PCM, so 2 bytes per sample
            const buffer = new ArrayBuffer(44 + dataLength);
            const view = new DataView(buffer);

            // RIFF chunk descriptor
            writeString(view, 0, 'RIFF');
            view.setUint32(4, 36 + dataLength, true);
            writeString(view, 8, 'WAVE');
            
            // fmt sub-chunk
            writeString(view, 12, 'fmt ');
            view.setUint32(16, 16, true);
            view.setUint16(20, 1, true); // PCM format
            view.setUint16(22, 1, true); // Mono
            view.setUint32(24, sampleRate, true);
            view.setUint32(28, sampleRate * 2, true); // Byte rate
            view.setUint16(32, 2, true); // Block align
            view.setUint16(34, 16, true); // Bits per sample
            
            // data sub-chunk
            writeString(view, 36, 'data');
            view.setUint32(40, dataLength, true);
            
            // Write PCM data
            for (let i = 0; i < pcm16.length; i++) {
                view.setInt16(44 + i * 2, pcm16[i], true);
            }

            return new Blob([view], { type: 'audio/wav' });
        }

        function writeString(view, offset, string) {
            for (let i = 0; i < string.length; i++) {
                view.setUint8(offset + i, string.charCodeAt(i));
            }
        }

    </script>
</body>
</html>
