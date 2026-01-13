# examen-seguro
exámenes cbt, tecno, etc


<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Examen Seguro - Monitorizado</title>
    
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs"></script>
    <script src="https://cdn.jsdelivr.net/npm/@tensorflow-models/blazeface"></script>
    <script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>

    <style>
        body { margin: 0; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; overflow: hidden; background: #f0f2f5; }
        
        /* PANTALLA DE INICIO */
        #welcome-screen {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: linear-gradient(135deg, #1a2a6c, #b21f1f, #fdbb2d);
            display: flex; flex-direction: column; justify-content: center; align-items: center;
            z-index: 9999; color: white; text-align: center;
        }
        .btn-start {
            padding: 15px 40px; font-size: 1.5rem; background: white; color: #333;
            border: none; border-radius: 50px; cursor: pointer; transition: transform 0.2s;
            font-weight: bold; margin-top: 20px; box-shadow: 0 4px 15px rgba(0,0,0,0.3);
        }
        .btn-start:hover { transform: scale(1.05); }

        /* CONTENEDOR PRINCIPAL */
        #app-container { display: flex; height: 100vh; opacity: 0; transition: opacity 1s; }

        /* BARRA LATERAL (MONITOREO) */
        #sidebar {
            width: 280px; background: #1e1e1e; color: white;
            display: flex; flex-direction: column; padding: 15px;
            box-shadow: 2px 0 10px rgba(0,0,0,0.2);
        }
        #video-container {
            width: 100%; height: 200px; background: black; border-radius: 8px;
            overflow: hidden; margin-bottom: 15px; border: 2px solid #333; position: relative;
        }
        video { width: 100%; height: 100%; object-fit: cover; transform: scaleX(-1); }
        
        .metric-box { background: #2c2c2c; padding: 10px; border-radius: 8px; margin-bottom: 15px; }
        .metric-title { font-size: 0.8rem; color: #aaa; margin-bottom: 5px; }
        
        /* BARRA DE CONFIANZA */
        #trust-bar-container { height: 10px; background: #444; border-radius: 5px; overflow: hidden; }
        #trust-bar { height: 100%; width: 100%; background: #4caf50; transition: width 0.5s, background 0.5s; }
        #trust-score { font-size: 2rem; font-weight: bold; text-align: center; margin: 5px 0; }

        #log-console {
            flex-grow: 1; background: #000; border: 1px solid #333; 
            font-family: monospace; font-size: 0.75rem; color: #0f0; 
            padding: 10px; overflow-y: auto; border-radius: 5px;
        }
        .log-item { margin-bottom: 4px; border-bottom: 1px solid #222; padding-bottom: 2px; }
        .log-bad { color: #ff4444; }

        /* ÁREA DEL EXAMEN */
        #exam-area { flex-grow: 1; position: relative; }
        iframe { width: 100%; height: 100%; border: none; }

        /* BLOQUEO DE PANTALLA */
        #lock-overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.95); color: red;
            display: none; flex-direction: column; justify-content: center; align-items: center;
            z-index: 1000;
        }
    </style>
</head>
<body>

    <div id="welcome-screen">
        <h1>Sistema de Examen Seguro</h1>
        <p>Se requieren permisos de cámara para verificar tu identidad.</p>
        <p>⚠️ No cambies de pestaña o tu nivel de confianza bajará.</p>
        <button class="btn-start" onclick="iniciarExamen()">COMENZAR EXAMEN</button>
    </div>

    <div id="app-container">
        <div id="sidebar">
            <div id="video-container">
                <video id="webcam" autoplay playsinline muted></video>
            </div>
            
            <div class="metric-box">
                <div class="metric-title">NIVEL DE CONFIANZA</div>
                <div id="trust-score">100%</div>
                <div id="trust-bar-container">
                    <div id="trust-bar"></div>
                </div>
            </div>

            <div class="metric-title">REGISTRO DE EVENTOS</div>
            <div id="log-console">
                <div class="log-item">Esperando inicio...</div>
            </div>
        </div>

        <div id="exam-area">
            <div id="lock-overlay">
                <h1>🚫 EXAMEN BLOQUEADO</h1>
                <p>Nivel de confianza en 0%. Contacta al profesor.</p>
            </div>

            <iframe 
                src="https://docs.google.com/forms/d/e/1FAIpQLScjy0T7oC8P0Wu01iIjx9dC48IKOj8yYH1Qaw7TjqCNVv0P5g/viewform?embedded=true" 
            ></iframe>
        </div>
    </div>

    <script>
        // CONFIGURACIÓN
        let trustScore = 100;
        let model = null;
        let isExamActive = false;
        const video = document.getElementById('webcam');
        const logConsole = document.getElementById('log-console');

        // 1. FUNCIÓN DE INICIO
        async function iniciarExamen() {
            try {
                // Solicitar cámara
                const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
                video.srcObject = stream;
                
                // Cargar modelo de IA
                Swal.fire({ title: 'Cargando IA...', text: 'Por favor espera', allowOutsideClick: false, didOpen: () => { Swal.showLoading() } });
                model = await blazeface.load();
                Swal.close();

                // Mostrar interfaz
                document.getElementById('welcome-screen').style.display = 'none';
                document.getElementById('app-container').style.opacity = '1';
                isExamActive = true;
                
                logEvent("Examen iniciado. Cámara activa.");
                monitorizarRostro(); // Iniciar loop de IA

            } catch (err) {
                Swal.fire('Error', 'Necesitamos acceso a la cámara para continuar.', 'error');
            }
        }

        // 2. SISTEMA DE PUNTUACIÓN Y LOGS
        function updateTrust(amount, reason) {
            if (!isExamActive) return;
            
            trustScore += amount;
            if (trustScore > 100) trustScore = 100;
            if (trustScore <= 0) {
                trustScore = 0;
                bloquearExamen();
            }

            // Actualizar UI
            const scoreEl = document.getElementById('trust-score');
            const barEl = document.getElementById('trust-bar');
            scoreEl.innerText = trustScore + "%";
            barEl.style.width = trustScore + "%";

            // Colores del semáforo
            if (trustScore > 80) { barEl.style.background = "#4caf50"; scoreEl.style.color = "#4caf50"; }
            else if (trustScore > 50) { barEl.style.background = "#ff9800"; scoreEl.style.color = "#ff9800"; }
            else { barEl.style.background = "#f44336"; scoreEl.style.color = "#f44336"; }

            if (amount < 0) logEvent(`ALERTA: ${reason} (${amount} pts)`, true);
        }

        function logEvent(msg, isBad = false) {
            const div = document.createElement('div');
            div.className = isBad ? 'log-item log-bad' : 'log-item';
            const time = new Date().toLocaleTimeString();
            div.innerText = `[${time}] ${msg}`;
            logConsole.prepend(div);
        }

        function bloquearExamen() {
            isExamActive = false;
            document.getElementById('lock-overlay').style.display = 'flex';
            logEvent("EXAMEN BLOQUEADO POR SEGURIDAD", true);
        }

        // 3. VIGILANCIA DE PESTAÑAS (VISIBILITY API)
        document.addEventListener("visibilitychange", () => {
            if (document.hidden) {
                updateTrust(-15, "Cambio de pestaña detectado");
                Swal.fire({
                    title: '¡Ojo!',
                    text: 'No salgas de la pantalla del examen.',
                    icon: 'warning',
                    timer: 2000,
                    showConfirmButton: false
                });
            }
        });

        window.addEventListener("blur", () => {
            if (isExamActive) updateTrust(-5, "Pérdida de foco (Clic fuera)");
        });

        // 4. VIGILANCIA DE ROSTRO (IA)
        async function monitorizarRostro() {
            if (!isExamActive) return;

            const predictions = await model.estimateFaces(video, false);

            if (predictions.length === 0) {
                updateTrust(-1, "Rostro no detectado"); // Penalización leve pero constante
            } else if (predictions.length > 1) {
                updateTrust(-20, "Múltiples rostros detectados"); // Penalización grave
            } else {
                // Si todo está bien, recuperamos confianza lentamente
                if (trustScore < 100) updateTrust(0.5, "Recuperando confianza"); 
            }

            // Repetir cada 500ms
            setTimeout(monitorizarRostro, 500);
        }
    </script>
</body>
</html>
