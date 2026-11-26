//pianta-3D
//An HTML grafic simulation: A tree that change appearance with wind
<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Pianta Frattale 3D - Oscillazione Elastica al Vento</title>
    <style>
        body, html {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #0d1117;
            font-family: sans-serif;
            touch-action: manipulation;
        }
        canvas {
            display: block;
            width: 100vw;
            height: 100vh;
        }
        #info {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            color: #8b949e;
            background: rgba(22, 27, 34, 0.85);
            padding: 10px 20px;
            border-radius: 8px;
            font-size: 14px;
            pointer-events: none;
            border: 1px solid #30363d;
            box-shadow: 0 4px 12px rgba(0,0,0,0.5);
            text-align: center;
            white-space: nowrap;
        }
    </style>
</head>
<body>

    <canvas id="canvas"></canvas>
    <div id="info">Orizzontale = dall'alto • Inclina il telefono per variare vista • Tocca per cambiare pianta</div>

    <script>
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');

        // Parametri di proiezione prospettica dinamica
        const FOV = 680;
        const CAMERA_HEIGHT = 260;
        const SOLVER_PASSES = 3; // Numero di iterazioni dal ramo inferiore verso i rami esterni per frame

        // Stato angolare della vista (in radianti): PI/2 = vista dall'alto (telefono orizzontale)
        let currentViewAngle = Math.PI / 2;
        let targetViewAngle = Math.PI / 2;
        let currentTiltXOffset = 0;
        let targetTiltXOffset = 0;
        let hasSensorData = false;

        function resizeCanvas() {
            const dpr = window.devicePixelRatio || 1;
            canvas.width = window.innerWidth * dpr;
            canvas.height = window.innerHeight * dpr;
            ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
        }
        window.addEventListener('resize', resizeCanvas);

        // --- FUNZIONI MATEMATICHE VETTORIALI 3D ---
        function vecSub(a, b) { return { x: a.x - b.x, y: a.y - b.y, z: a.z - b.z }; }
        function vecAdd(a, b) { return { x: a.x + b.x, y: a.y + b.y, z: a.z + b.z }; }
        function vecScale(v, s) { return { x: v.x * s, y: v.y * s, z: v.z * s }; }
        function vecDot(a, b) { return a.x * b.x + a.y * b.y + a.z * b.z; }
        function vecCross(a, b) {
            return {
                x: a.y * b.z - a.z * b.y,
                y: a.z * b.x - a.x * b.z,
                z: a.x * b.y - a.y * b.x
            };
        }
        function vecLen(v) { return Math.hypot(v.x, v.y, v.z); }
        function vecNorm(v) {
            const l = vecLen(v);
            return l > 0.00001 ? { x: v.x / l, y: v.y / l, z: v.z / l } : { x: 0, y: 0, z: 0 };
        }
        function vecLerp(a, b, t) {
            return {
                x: a.x + (b.x - a.x) * t,
                y: a.y + (b.y - a.y) * t,
                z: a.z + (b.z - a.z) * t
            };
        }

        function rotateAroundAxis(v, k, theta) {
            if (Math.abs(theta) < 1e-6 || vecLen(k) < 1e-5) return { ...v };
            const cos = Math.cos(theta);
            const sin = Math.sin(theta);
            const cross = vecCross(k, v);
            const dot = vecDot(k, v);
            return {
                x: v.x * cos + cross.x * sin + k.x * dot * (1 - cos),
                y: v.y * cos + cross.y * sin + k.y * dot * (1 - cos),
                z: v.z * cos + cross.z * sin + k.z * dot * (1 - cos)
            };
        }

        function rotateVector(vx, vy, vz, angleX, angleY, angleZ) {
            let cos = Math.cos(angleX), sin = Math.sin(angleX);
            let y1 = vy * cos - vz * sin;
            let z1 = vy * sin + vz * cos;
            let x1 = vx;

            cos = Math.cos(angleY); sin = Math.sin(angleY);
            let x2 = x1 * cos + z1 * sin;
            let z2 = -x1 * sin + z1 * cos;
            let y2 = y1;

            cos = Math.cos(angleZ); sin = Math.sin(angleZ);
            let x3 = x2 * cos - y2 * sin;
            let y3 = x2 * sin + y2 * cos;
            let z3 = z2;

            return { x: x3, y: y3, z: z3 };
        }

        function randomRange(min, max) {
            return min + Math.random() * (max - min);
        }

        function clamp(val, min, max) {
            return Math.max(min, Math.min(max, val));
        }

        // Aggiorna l'angolo di vista in base all'inclinazione del telefono (beta e gamma)
        function handleOrientation(event) {
            if (event.beta === null && event.gamma === null) return;
            hasSensorData = true;

            // beta: 0° = orizzontale (vista dall'alto = 90°), inclinazione fino a ~70° abbassa la vista verso ~18°
            const beta = clamp(Math.abs(event.beta || 0), 0, 75);
            const topDownRatio = 1 - (beta / 75); // 1.0 orizzontale -> 0.0 inclinato
            const minAngle = 18 * (Math.PI / 180);
            const maxAngle = 90 * (Math.PI / 180);
            targetViewAngle = minAngle + topDownRatio * (maxAngle - minAngle);

            // gamma: inclinazione laterale sinistra/destra (-45° .. +45°)
            const gamma = clamp(event.gamma || 0, -45, 45);
            targetTiltXOffset = (gamma / 45) * 140;
        }

        async function requestSensorAccessIfNeeded() {
            if (typeof DeviceOrientationEvent !== 'undefined' &&
                typeof DeviceOrientationEvent.requestPermission === 'function') {
                try {
                    const response = await DeviceOrientationEvent.requestPermission();
                    if (response === 'granted') {
                        window.addEventListener('deviceorientation', handleOrientation, true);
                    }
                } catch (err) {
                    // Ignora se negato o non supportato
                }
            }
        }

        window.addEventListener('deviceorientation', handleOrientation, true);

        // Proiezione prospettica dinamica legata all'inclinazione del telefono
        function projectTopPerspective(x, y, z, width, height) {
            const cx = width / 2;
            const cy = height * 0.56;

            const relX = x + currentTiltXOffset;
            const relY = y + CAMERA_HEIGHT;
            const relZ = z;

            const cosA = Math.cos(currentViewAngle);
            const sinA = Math.sin(currentViewAngle);

            const yt = relY * cosA - relZ * sinA;
            const zt = relY * sinA + relZ * cosA;

            const depthDist = zt + FOV;
            if (depthDist <= 20) return null;

            const scale = FOV / depthDist;

            return {
                x: cx + relX * scale,
                y: cy + yt * scale,
                scale: scale,
                depthDist: depthDist
            };
        }

        // Mappa un punto sullo schermo sul piano di base (y = 0)
        function screenToWorldGround(screenX, screenY, width, height) {
            const cx = width / 2;
            const cy = height * 0.56;

            const cosA = Math.cos(currentViewAngle);
            const sinA = Math.sin(currentViewAngle);

            const syOffset = screenY - cy;
            const ytNum = CAMERA_HEIGHT * cosA;
            const ztConst = CAMERA_HEIGHT * sinA + FOV;

            const denom = syOffset * cosA + FOV * sinA;
            const num = FOV * ytNum - syOffset * ztConst;

            const z = Math.abs(denom) > 0.0001 ? num / denom : 0;
            const zt = CAMERA_HEIGHT * sinA + z * cosA;
            const depthDist = zt + FOV;
            const scale = depthDist > 20 ? FOV / depthDist : 1;

            const x = (screenX - cx) / scale - currentTiltXOffset;
            return { x: x, y: 0, z: z };
        }

        // --- SISTEMA DI SIMULAZIONE DEL VENTO (VELOCITÀ DIMEZZATA, AMPIEZZA INVARIATA) ---
        class SistemaVento {
            constructor() {
                this.maxForce = 18.0;
                this.speedMultiplier = 1.0;
                this.state = 'IDLE';
                this.timer = 0.3;
                this.duration = 0.9;

                this.baseAngle = randomRange(0, Math.PI * 2);
                this.currentAngle = this.baseAngle;
                this.targetAngleOffset = 0;
                this.startAngleOffset = 0;

                this.dirTimer = 0;
                this.dirDuration = 0.3;

                this.magnitude = 0;
                this.vector = { x: 0, y: 0, z: 0 };
            }

            update(dt) {
                const windDt = dt * this.speedMultiplier;

                if (this.state === 'IDLE') {
                    this.magnitude = 0;
                    this.vector = { x: 0, y: 0, z: 0 };
                    this.timer -= windDt;

                    if (this.timer <= 0) {
                        this.state = 'GROWING';
                        this.baseAngle = randomRange(0, Math.PI * 2);
                        this.startAngleOffset = 0;
                        this.targetAngleOffset = randomRange(-56, 56) * (Math.PI / 180);
                        this.dirTimer = 0;
                        this.dirDuration = randomRange(0.175, 0.5);

                        this.duration = randomRange(0.6, 1.3);
                        this.timer = this.duration;
                    }
                } else {
                    this.dirTimer += windDt;
                    if (this.dirTimer >= this.dirDuration) {
                        this.dirTimer = 0;
                        this.dirDuration = randomRange(0.175, 0.5);
                        this.startAngleOffset = this.targetAngleOffset;
                        this.targetAngleOffset = randomRange(-56, 56) * (Math.PI / 180);
                    }
                    const dirProgress = Math.min(1, this.dirTimer / this.dirDuration);
                    const smoothDir = dirProgress * dirProgress * (3 - 2 * dirProgress);
                    this.currentAngle = this.baseAngle + this.startAngleOffset + (this.targetAngleOffset - this.startAngleOffset) * smoothDir;

                    this.timer -= windDt;
                    const progress = 1 - Math.max(0, this.timer / this.duration);

                    if (this.state === 'GROWING') {
                        this.magnitude = this.maxForce * Math.sin(progress * (Math.PI / 2));
                        if (this.timer <= 0) {
                            this.state = 'DECREASING';
                            this.duration = randomRange(0.75, 1.5);
                            this.timer = this.duration;
                        }
                    } else if (this.state === 'DECREASING') {
                        this.magnitude = this.maxForce * Math.cos(progress * (Math.PI / 2));
                        if (this.timer <= 0) {
                            this.state = 'IDLE';
                            this.magnitude = 0;
                            this.timer = randomRange(0.9, 2.0);
                        }
                    }

                    this.vector = {
                        x: Math.cos(this.currentAngle) * this.magnitude,
                        y: 0,
                        z: Math.sin(this.currentAngle) * this.magnitude
                    };
                }
            }
        }

        // --- CLASSE PUNTO 3D ---
        class Punto {
            constructor(x, y, z, vx, vy, vz, length, color, parentSegment = null) {
                this.x = x;
                this.y = y;
                this.z = z;
                this.vx = vx;
                this.vy = vy;
                this.vz = vz;
                this.length = length;
                this.color = color;
                this.parentSegment = parentSegment;
            }
        }

        // --- CLASSE SEGMENTO CON CALCOLO ITERATIVO DAL RAMO INFERIORE AI PIÙ ESTERNI ---
        class Segmento {
            constructor(x1, y1, z1, x2, y2, z2, depth, color, parentSegment = null) {
                this.baseStart = { x: x1, y: y1, z: z1 };
                this.baseEnd = { x: x2, y: y2, z: z2 };

                this.x1 = x1; this.y1 = y1; this.z1 = z1;
                this.x2 = x2; this.y2 = y2; this.z2 = z2;

                this.depth = depth;
                this.color = color;
                this.parent = parentSegment;
                this.children = [];

                if (this.parent) {
                    this.relVec = vecSub(this.baseEnd, this.parent.baseEnd);
                    this.parent.children.push(this);
                } else {
                    this.relVec = vecSub(this.baseEnd, this.baseStart);
                }
                this.length = vecLen(this.relVec);

                this.angle = 0;
                this.vel = 0;
                this.axis = { x: 0, y: 0, z: 1 };

                // Range di flessione: 1° al livello 0, +10° per ogni livello successivo (1°, 11°, 21°, 31°, ...)
                this.maxDeflection = (1 + 10 * depth) * (Math.PI / 180);
                this.stiffness = Math.max(24, 58 - depth * 4.5 + randomRange(-2, 2));
                this.damping = Math.max(3.4, 5.6 - depth * 0.35);
            }

            // Calcola la posizione del segmento partendo dal ramo inferiore (genitore già aggiornato) verso i rami esterni
            updateFromLowerToOuter(windVec, stepDt) {
                if (this.parent) {
                    this.x1 = this.parent.x2;
                    this.y1 = this.parent.y2;
                    this.z1 = this.parent.z2;
                } else {
                    this.x1 = this.baseStart.x;
                    this.y1 = this.baseStart.y;
                    this.z1 = this.baseStart.z;
                }

                // Direzione di riferimento corrente: include l'eventuale rotazione del ramo genitore
                const referenceVec = this.parent
                    ? rotateAroundAxis(this.relVec, this.parent.axis, this.parent.angle)
                    : this.relVec;

                const windMag = vecLen(windVec);
                let targetAngle = 0;

                if (windMag > 0.001 && this.length > 0.001) {
                    const segDir = vecNorm(referenceVec);
                    const windDir = vecNorm(windVec);

                    // Angolo theta tra direzione del vento e direzione attuale del segmento
                    const cosTheta = clamp(vecDot(windDir, segDir), -1, 1);
                    const theta = Math.acos(cosTheta);
                    const sinTheta = Math.sin(theta);

                    const windFactor = clamp(windMag / 18.0, 0, 1);
                    const angleFactor = sinTheta * (0.45 + 0.55 * (theta / Math.PI));
                    const forceMag = windFactor * angleFactor;

                    if (forceMag > 0.0001) {
                        const perpWind = vecSub(windVec, vecScale(segDir, vecDot(windVec, segDir)));
                        const newAxis = vecNorm(vecCross(segDir, perpWind));
                        if (vecLen(newAxis) > 0.001) {
                            const alignedAxis = vecDot(this.axis, newAxis) < 0
                                ? vecScale(newAxis, -1)
                                : newAxis;
                            this.axis = vecNorm(
                                vecLerp(this.axis, alignedAxis, Math.min(1, stepDt * 12))
                            );
                        }
                        targetAngle = forceMag * this.maxDeflection;
                    }
                }

                const accel = this.stiffness * (targetAngle - this.angle) - this.damping * this.vel;
                this.vel += accel * stepDt;
                this.angle += this.vel * stepDt;

                const finalVec = rotateAroundAxis(referenceVec, this.axis, this.angle);
                this.x2 = this.x1 + finalVec.x;
                this.y2 = this.y1 + finalVec.y;
                this.z2 = this.z1 + finalVec.z;
            }

            getDepthScore(width, height) {
                const p1 = projectTopPerspective(this.x1, this.y1, this.z1, width, height);
                const p2 = projectTopPerspective(this.x2, this.y2, this.z2, width, height);
                if (!p1 && !p2) return 1e9;
                if (!p1) return p2.depthDist;
                if (!p2) return p1.depthDist;
                return (p1.depthDist + p2.depthDist) * 0.5;
            }

            draw(ctx, width, height) {
                const p1 = projectTopPerspective(this.x1, this.y1, this.z1, width, height);
                const p2 = projectTopPerspective(this.x2, this.y2, this.z2, width, height);

                if (!p1 || !p2) return;

                const avgScale = (p1.scale + p2.scale) * 0.5;

                ctx.beginPath();
                ctx.moveTo(p1.x, p1.y);
                ctx.lineTo(p2.x, p2.y);

                ctx.lineWidth = Math.max(0.8, (6.5 - this.depth) * avgScale * 2.0);

                const depthShading = (this.z2 / 10);
                const finalL = Math.min(82, Math.max(15, this.color.l + depthShading));

                ctx.strokeStyle = `hsl(${this.color.h}, ${this.color.s}%, ${finalL}%)`;
                ctx.lineCap = 'round';
                ctx.stroke();
            }
        }

        // --- STATO GLOBALE E PARAMETRI ---
        let tuttiISegmenti = [];
        const vento = new SistemaVento();

        let lastParams = {
            m: 5,
            nMax: 6,
            alfa: 35 * (Math.PI / 180),
            k: 0.8,
            initialLen: window.innerWidth / 15,
            color: { h: 60, s: 50, l: 30 }
        };

        // --- GENERAZIONE DI UNA SOLA PIANTA ---
        function mostraNuovaPianta(width, height) {
            const isFirst = tuttiISegmenti.length === 0;
            tuttiISegmenti = [];

            if (!isFirst) {
                lastParams.m = Math.round(clamp(lastParams.m * randomRange(0.5, 3.0), 3, 6));
                lastParams.nMax = Math.round(clamp(lastParams.nMax * randomRange(0.5, 3.0), 1, 6));
                lastParams.alfa = clamp(lastParams.alfa * randomRange(0.5, 3.0), 10 * Math.PI/180, 55 * Math.PI/180);
                lastParams.k = clamp(lastParams.k * randomRange(0.5, 3.0), 0.5, 0.92);
                lastParams.initialLen = clamp(lastParams.initialLen * randomRange(0.5, 3.0), width / 40, width / 6);

                lastParams.color.h = (lastParams.color.h * randomRange(0.5, 3.0)) % 360;
                lastParams.color.s = clamp(lastParams.color.s * randomRange(0.5, 3.0), 20, 80);
                lastParams.color.l = clamp(lastParams.color.l * randomRange(0.5, 3.0), 20, 50);
            }

            const basePos = screenToWorldGround(width / 2, height * 0.62, width, height);

            const puntoIniziale = new Punto(
                basePos.x, basePos.y, basePos.z,
                0, -1, 0,
                lastParams.initialLen,
                { ...lastParams.color },
                null
            );

            function ricorsioneRami(punto, livelloCorrente) {
                if (livelloCorrente >= lastParams.m) return;

                const numRami = Math.floor(randomRange(1, 7));

                for (let i = 0; i < numRami; i++) {
                    const devX = randomRange(-lastParams.alfa, lastParams.alfa);
                    const devY = randomRange(-lastParams.alfa, lastParams.alfa);
                    const devZ = randomRange(-lastParams.alfa, lastParams.alfa);

                    const rot = rotateVector(punto.vx, punto.vy, punto.vz, devX, devY, devZ);
                    const lenSeg = randomRange(0, punto.length);

                    const xEnd = punto.x + rot.x * lenSeg;
                    const yEnd = punto.y + rot.y * lenSeg;
                    const zEnd = punto.z + rot.z * lenSeg;

                    const varFactorH = 1 + randomRange(-0.10, 0.10);
                    const varFactorS = 1 + randomRange(-0.10, 0.10);
                    const levelBrightnessBoost = (livelloCorrente * 2);

                    const segColor = {
                        h: Math.max(0, Math.min(360, punto.color.h * varFactorH)),
                        s: Math.max(0, Math.min(100, punto.color.s * varFactorS)),
                        l: Math.max(10, Math.min(85, punto.color.l + levelBrightnessBoost + randomRange(-10, 10)))
                    };

                    const nuovoSegmento = new Segmento(
                        punto.x, punto.y, punto.z,
                        xEnd, yEnd, zEnd,
                        livelloCorrente,
                        segColor,
                        punto.parentSegment
                    );
                    tuttiISegmenti.push(nuovoSegmento);

                    const nuovoPunto = new Punto(
                        xEnd, yEnd, zEnd,
                        rot.x, rot.y, rot.z,
                        punto.length * lastParams.k,
                        segColor,
                        nuovoSegmento
                    );

                    ricorsioneRami(nuovoPunto, livelloCorrente + 1);
                }
            }

            ricorsioneRami(puntoIniziale, 0);
        }

        // --- CICLO DI ANIMAZIONE E DISEGNO ---
        let lastTime = performance.now();

        function loop(now) {
            const dt = Math.min(0.05, (now - lastTime) / 1000);
            lastTime = now;

            // Interpolazione fluida dell'angolo e dell'inclinazione del sensore
            const smoothFactor = Math.min(1, dt * 10);
            currentViewAngle += (targetViewAngle - currentViewAngle) * smoothFactor;
            currentTiltXOffset += (targetTiltXOffset - currentTiltXOffset) * smoothFactor;

            vento.update(dt);

            // Calcolo iterativo più volte per frame dal ramo inferiore (depth minore) verso i più esterni (depth maggiore)
            const dalBassoVersoEsterno = [...tuttiISegmenti].sort((a, b) => a.depth - b.depth);
            const stepDt = dt / SOLVER_PASSES;

            for (let pass = 0; pass < SOLVER_PASSES; pass++) {
                for (let i = 0; i < dalBassoVersoEsterno.length; i++) {
                    dalBassoVersoEsterno[i].updateFromLowerToOuter(vento.vector, stepDt);
                }
            }

            // Disegno dalla distanza maggiore alla minore (painter's algorithm)
            ctx.fillStyle = '#0d1117';
            ctx.fillRect(0, 0, window.innerWidth, window.innerHeight);

            const segmentiOrdinati = [...tuttiISegmenti].sort(
                (a, b) => b.getDepthScore(window.innerWidth, window.innerHeight) - a.getDepthScore(window.innerWidth, window.innerHeight)
            );

            for (let i = 0; i < segmentiOrdinati.length; i++) {
                segmentiOrdinati[i].draw(ctx, window.innerWidth, window.innerHeight);
            }

            requestAnimationFrame(loop);
        }

        // Inizializzazione
        resizeCanvas();
        mostraNuovaPianta(window.innerWidth, window.innerHeight);
        requestAnimationFrame(loop);

        // Eventi interattivi
        window.addEventListener('click', async () => {
            await requestSensorAccessIfNeeded();
            mostraNuovaPianta(window.innerWidth, window.innerHeight);
        });

        window.addEventListener('touchstart', async (e) => {
            e.preventDefault();
            await requestSensorAccessIfNeeded();
            mostraNuovaPianta(window.innerWidth, window.innerHeight);
        }, { passive: false });
    </script>
</body>
</html>
