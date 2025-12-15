#<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8" />
    <title>Système Particules & Hand Tracking - Temps Réel</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; }
        canvas { display: block; }
        #loading {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            color: white; font-family: monospace; font-size: 24px; pointer-events: none;
            text-align: center;
        }
        video { display: none; } /* Cache la vidéo brute */
    </style>
</head>
<body>

    <div id="loading">
        Initialisation Webcam &amp; IA...<br>
        <span style="font-size:14px; opacity:0.7">Veuillez autoriser la caméra</span>
    </div>
    <video id="input_video" playsinline></video>

    <script type="module">
        import * as THREE from 'https://cdn.skypack.dev/three@0.136.0';
        import { Hands } from 'https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js';
        import { Camera } from 'https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js';

        // --- CONFIGURATION ---
        const CONFIG = {
            particleCount: 15000,
            particleSize: 0.8,
            baseSpeed: 0.05,
            attractionStrength: 0.08,
            friction: 0.96,
            pinchMinSize: 0.3,   // taille min des particules
            pinchMaxSize: 2.0    // taille max des particules
        };

        // --- VARIABLES GLOBALES ---
        let scene, camera, renderer, particles, positions, velocities;
        let particleMaterial;
        let handPosition = new THREE.Vector3(0, 0, 0);
        let isHandDetected = false;
        let isPinched = false;
        let pinchFactor = 1.0;
        let time = 0;

        // --- 1. SETUP GRAPHIQUE (Three.js) ---
        function initThree() {
            scene = new THREE.Scene();
            scene.fog = new THREE.FogExp2(0x000000, 0.002);

            camera = new THREE.PerspectiveCamera(
                75,
                window.innerWidth / window.innerHeight,
                1,
                1000
            );
            camera.position.z = 100;

            renderer = new THREE.WebGLRenderer({
                antialias: true,
                alpha: true,
                powerPreference: 'high-performance'
            });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.body.appendChild(renderer.domElement);

            createParticles();
            window.addEventListener('resize', onWindowResize, false);
        }

        // --- 2. CRÉATION DES PARTICULES ---
        function createParticles() {
            const geometry = new THREE.BufferGeometry();
            positions = [];
            velocities = [];

            // Texture générée dynamiquement (Glow soft)
            const canvas = document.createElement('canvas');
            canvas.width = 32;
            canvas.height = 32;
            const context = canvas.getContext('2d');
            const gradient = context.createRadialGradient(16, 16, 0, 16, 16, 16);
            gradient.addColorStop(0, 'rgba(255, 255, 255, 1)');
            gradient.addColorStop(0.2, 'rgba(0, 255, 255, 0.6)');
            gradient.addColorStop(0.5, 'rgba(0, 0, 64, 0.2)');
            gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');
            context.fillStyle = gradient;
            context.fillRect(0, 0, 32, 32);
            const texture = new THREE.CanvasTexture(canvas);

            for (let i = 0; i < CONFIG.particleCount; i++) {
                positions.push((Math.random() * 200) - 100); // X
                positions.push((Math.random() * 120) - 60);  // Y
                positions.push((Math.random() * 100) - 50);  // Z
                velocities.push(0, 0, 0);
            }

            geometry.setAttribute(
                'position',
                new THREE.Float32BufferAttribute(positions, 3)
            );

            particleMaterial = new THREE.PointsMaterial({
                color: 0xffffff,
                size: CONFIG.particleSize,
                map: texture,
                transparent: true,
                blending: THREE.AdditiveBlending,
                depthWrite: false,
                sizeAttenuation: true
            });

            particles = new THREE.Points(geometry, particleMaterial);
            scene.add(particles);
        }

        // --- 3. MOTEUR PHYSIQUE (Mise à jour) ---
        function updateParticles() {
            if (!particles) return;

            const posAttribute = particles.geometry.attributes.position;
            const p = posAttribute.array;

            for (let i = 0; i < CONFIG.particleCount; i++) {
                let ix = i * 3;
                let iy = i * 3 + 1;
                let iz = i * 3 + 2;

                // 1. Interaction
                if (isHandDetected) {
                    const dx = handPosition.x - p[ix];
                    const dy = handPosition.y - p[iy];
                    const dz = handPosition.z - p[iz];

                    const dist = Math.sqrt(dx * dx + dy * dy + dz * dz);

                    if (dist > 1) {
                        velocities[ix] += (dx / dist) * CONFIG.attractionStrength;
                        velocities[iy] += (dy / dist) * CONFIG.attractionStrength;
                        velocities[iz] += (dz / dist) * CONFIG.attractionStrength;
                    }
                } else {
                    velocities[ix] += Math.sin(iy * 0.05 + time) * 0.02;
                    velocities[iy] += Math.cos(ix * 0.05 + time) * 0.02;
                }

                // 2. Physique
                velocities[ix] *= CONFIG.friction;
                velocities[iy] *= CONFIG.friction;
                velocities[iz] *= CONFIG.friction;

                p[ix] += velocities[ix];
                p[iy] += velocities[iy];
                p[iz] += velocities[iz];

                // 3. Respawn si hors zone
                if (Math.abs(p[ix]) > 150 || Math.abs(p[iy]) > 100) {
                    p[ix] = (Math.random() * 200) - 100;
                    p[iy] = (Math.random() * 120) - 60;
                    velocities[ix] = 0;
                    velocities[iy] = 0;
                }
            }

            posAttribute.needsUpdate = true;
            particles.rotation.y += 0.001;
        }

        // --- 4. CONFIGURATION MEDIAPIPE (Tracking + pincement) ---
        function onResults(results) {
            const loading = document.getElementById('loading');
            if (loading) loading.style.display = 'none';

            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                isHandDetected = true;
                const hand = results.multiHandLandmarks[0];

                const indexFinger = hand[8]; // INDEX_FINGER_TIP
                const thumbTip   = hand[4]; // THUMB_TIP

                // Coordonnées de la main vers l'espace 3D
                const targetX = (0.5 - indexFinger.x) * 250;
                const targetY = (0.5 - indexFinger.y) * 150;

                handPosition.x += (targetX - handPosition.x) * 0.1;
                handPosition.y += (targetY - handPosition.y) * 0.1;

                // Détection du pincement (distance pouce-index)
                const dx = thumbTip.x - indexFinger.x;
                const dy = thumbTip.y - indexFinger.y;
                const dz = (thumbTip.z || 0) - (indexFinger.z || 0);
                const pinchDistance = Math.sqrt(dx * dx + dy * dy + dz * dz);

                isPinched = pinchDistance < 0.07;

                const minDist = 0.03;
                const maxDist = 0.20;
                const t = Math.max(
                    0,
                    Math.min(1, (pinchDistance - minDist) / (maxDist - minDist))
                );
                const targetFactor =
                    CONFIG.pinchMinSize +
                    (CONFIG.pinchMaxSize - CONFIG.pinchMinSize) * t;

                pinchFactor += (targetFactor - pinchFactor) * 0.15;
            } else {
                isHandDetected = false;
            }
        }

        const videoElement = document.getElementById('input_video');

        const hands = new Hands({
            locateFile: (file) =>
                `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
        });

        hands.setOptions({
            maxNumHands: 2,
            modelComplexity: 1,
            minDetectionConfidence: 0.5,
            minTrackingConfidence: 0.5
        });

        hands.onResults(onResults);

        // --- Accès caméra via Camera Utils (utilise getUserMedia) ---
        const cameraFeed = new Camera(videoElement, {
            onFrame: async () => {
                await hands.send({ image: videoElement });
            },
            width: 640,
            height: 480
        });

        // Gestion des erreurs d'accès caméra (optionnel mais recommandé)
        async function startCamera() {
            try {
                // CameraUtils se charge d'appeler getUserMedia;
                // démarrer la capture déclenche la demande d'autorisation.
                cameraFeed.start();
            } catch (err) {
                console.error('Erreur d’accès à la caméra : ', err);
                const loading = document.getElementById('loading');
                if (loading) {
                    loading.innerHTML =
                        'Impossible d’accéder à la caméra.<br><span style="font-size:14px; opacity:0.7">Vérifiez les permissions du navigateur.</span>';
                }
            }
        }

        // --- 5. BOUCLE PRINCIPALE ---
        function animate() {
            requestAnimationFrame(animate);
            time += 0.05;
            updateParticles();

            if (particleMaterial) {
                particleMaterial.size = CONFIG.particleSize * pinchFactor;
            }

            renderer.render(scene, camera);
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        // Démarrage de la scène et de la caméra
        initThree();
        startCamera();
        animate();
    </script>
</body>
</html>
 didactic-octo-couscous
Pas grave chose
