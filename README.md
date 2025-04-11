// ==UserScript==
// @name         Virtual Webcam Mix
// @namespace    http://tampermonkey.net/
// @version      1.7
// @description  Exibe imagem ou vídeo como webcam virtual
// @author       Seu Nome
// @match        https://idnvui.vfsglobal.com/*
// @grant        GM_addStyle
// @grant        window.onurlchange
// @require      https://cdnjs.cloudflare.com/ajax/libs/howler/2.2.3/howler.min.js
// ==/UserScript==

(function() {
    'use strict';

    let virtualStream = null;
    let mediaSource = null;
    let canvasContext = null;
    let mediaElement = null;

    // Interface do usuário
    GM_addStyle(`
        #virtualCamUI {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(0,0,0,0.95);
            padding: 15px;
            border-radius: 10px;
            z-index: 999999;
            color: white;
            font-family: Arial, sans-serif;
            width: 300px;
            box-shadow: 0 0 15px rgba(0,0,0,0.7);
        }

        .vcam-btn {
            background: #2196F3;
            border: none;
            color: white;
            padding: 10px;
            margin: 5px 0;
            border-radius: 5px;
            cursor: pointer;
            width: 100%;
            transition: transform 0.2s;
        }

        .vcam-btn:hover {
            transform: scale(1.05);
        }

        #mediaPreview {
            max-width: 100%;
            margin: 10px 0;
            border-radius: 5px;
            display: none;
        }

        #status {
            font-size: 12px;
            text-align: center;
            margin-top: 10px;
            color: #00ff00;
        }
    `);

    const ui = document.createElement('div');
    ui.id = 'virtualCamUI';
    ui.innerHTML = `
        <input type="file" id="mediaInput" accept="video/*,image/*" hidden>
        <button class="vcam-btn" id="loadMedia">📁 Carregar Mídia</button>
        <button class="vcam-btn" id="toggleCam" disabled>🎥 Iniciar Webcam Virtual</button>
        <div id="previewContainer"></div>
        <div id="status">Status: Pronto</div>
        <canvas id="hiddenCanvas" hidden></canvas>
    `;

    document.body.appendChild(ui);

    // Elementos DOM
    const previewContainer = document.getElementById('previewContainer');
    const loadButton = document.getElementById('loadMedia');
    const toggleButton = document.getElementById('toggleCam');
    const hiddenCanvas = document.getElementById('hiddenCanvas');
    
    // Configurar canvas
    hiddenCanvas.width = 640;
    hiddenCanvas.height = 480;
    canvasContext = hiddenCanvas.getContext('2d');

    // Interceptar webcam
    const originalGetUserMedia = navigator.mediaDevices.getUserMedia.bind(navigator.mediaDevices);

    navigator.mediaDevices.getUserMedia = async (constraints) => {
        if (constraints.video && virtualStream) {
            return virtualStream.clone();
        }
        return originalGetUserMedia(constraints);
    };

    // Eventos
    loadButton.addEventListener('click', () => document.getElementById('mediaInput').click());

    document.getElementById('mediaInput').addEventListener('change', async (e) => {
        const file = e.target.files[0];
        if (!file) return;

        if (mediaSource) URL.revokeObjectURL(mediaSource);

        mediaSource = URL.createObjectURL(file);
        toggleButton.disabled = false;

        // Remover mídia anterior
        if (mediaElement) previewContainer.removeChild(mediaElement);

        if (file.type.startsWith('video/')) {
            // Criar vídeo
            mediaElement = document.createElement('video');
            mediaElement.src = mediaSource;
            mediaElement.controls = true;
            mediaElement.loop = true;
            mediaElement.muted = true;
            mediaElement.playsInline = true;
            mediaElement.style.maxWidth = '100%';
        } else {
            // Criar imagem
            mediaElement = document.createElement('img');
            mediaElement.src = mediaSource;
            mediaElement.style.maxWidth = '100%';
        }

        previewContainer.appendChild(mediaElement);
        updateStatus('Mídia carregada com sucesso!');
    });

    toggleButton.addEventListener('click', () => {
        if (virtualStream) {
            stopVirtualCam();
            toggleButton.textContent = '🎥 Iniciar Webcam Virtual';
        } else {
            startVirtualCam();
            toggleButton.textContent = '⏹ Parar Webcam Virtual';
        }
    });

    // Funções principais
    function startVirtualCam() {
        if (!mediaSource || !mediaElement) {
            updateStatus('Carregue uma mídia primeiro!', true);
            return;
        }

        if (mediaElement.tagName === 'IMG') {
            handleImageSource();
        } else if (mediaElement.tagName === 'VIDEO') {
            handleVideoSource();
        }
    }

    function handleImageSource() {
        const img = new Image();
        img.onload = () => {
            hiddenCanvas.width = img.naturalWidth;
            hiddenCanvas.height = img.naturalHeight;

            const drawFrame = () => {
                canvasContext.drawImage(img, 0, 0, hiddenCanvas.width, hiddenCanvas.height);
                requestAnimationFrame(drawFrame);
            };

            drawFrame();

            const stream = hiddenCanvas.captureStream(30);
            virtualStream = new MediaStream([stream.getVideoTracks()[0]]);
            updateStatus('Imagem sendo transmitida como vídeo!');
        };
        img.src = mediaSource;
    }

    function handleVideoSource() {
        mediaElement.play().then(() => {
            const stream = mediaElement.captureStream(30);
            virtualStream = new MediaStream([stream.getVideoTracks()[0]]);
            updateStatus('Vídeo sendo transmitido!');
        }).catch(error => {
            updateStatus(`Erro: ${error.message}`, true);
        });
    }

    function stopVirtualCam() {
        if (virtualStream) {
            virtualStream.getTracks().forEach(track => track.stop());
            virtualStream = null;
        }
        updateStatus('Webcam real restaurada');
    }

    function updateStatus(message, isError = false) {
        const status = document.getElementById('status');
        status.textContent = `Status: ${message}`;
        status.style.color = isError ? '#ff0000' : '#00ff00';
    }

    // Limpeza
    window.addEventListener('beforeunload', () => {
        if (virtualStream) virtualStream.getTracks().forEach(track => track.stop());
        if (mediaSource) URL.revokeObjectURL(mediaSource);
    });
})();
