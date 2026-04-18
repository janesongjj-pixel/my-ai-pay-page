<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Stargate: Gesture Experience</title>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@200;300&display=swap" rel="stylesheet">
    <style>
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #000;
            font-family: 'Inter', -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
        }

        #canvas-container {
            width: 100vw;
            height: 100vh;
        }

        /* 顶部标题样式：玻璃质感并置顶 */
        #main-title {
            position: absolute;
            top: 8%;
            left: 50%;
            transform: translateX(-50%);
            color: rgba(255, 255, 255, 0.75);
            font-family: 'Inter', sans-serif;
            font-weight: 200;
            font-size: 26px;
            letter-spacing: 0.5em;
            text-transform: uppercase;
            white-space: nowrap;
            z-index: 10;
            pointer-events: none;
            
            padding: 12px 35px;
            background: rgba(255, 255, 255, 0.03);
            backdrop-filter: blur(10px);
            -webkit-backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.1);
            border-radius: 50px;
            box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.5);
            text-shadow: 0 0 15px rgba(255, 255, 255, 0.3);
            
            transition: opacity 0.8s cubic-bezier(0.4, 0, 0.2, 1), transform 0.8s cubic-bezier(0.4, 0, 0.2, 1);
        }

        #ui-overlay {
            position: absolute;
            top: 20px;
            left: 20px;
            color: rgba(255, 255, 255, 0.6);
            z-index: 10;
            pointer-events: none;
        }

        .status-badge {
            display: inline-block;
            padding: 5px 12px;
            background: rgba(138, 43, 226, 0.15);
            border: 1px solid rgba(138, 43, 226, 0.3);
            border-radius: 20px;
            font-size: 11px;
            backdrop-filter: blur(5px);
            margin-bottom: 10px;
        }

        #camera-feed {
            position: absolute;
            bottom: 20px;
            right: 20px;
            width: 160px;
            height: 120px;
            border-radius: 12px;
            border: 1px solid rgba(0, 210, 255, 0.2);
            transform: scaleX(-1);
            object-fit: cover;
            background: #111;
            z-index: 5;
            opacity: 0.7;
        }

        #gesture-hint {
            position: absolute;
            top: 55%;
            left: 50%;
            transform: translate(-50%, -50%);
            color: white;
            text-align: center;
            font-size: 24px;
            font-weight: 200;
            letter-spacing: 4px;
            text-transform: uppercase;
            pointer-events: none;
            opacity: 0;
            /* 稍微拉长淡入时间 */
            transition: opacity 1.2s ease; 
            text-shadow: 0 0 20px rgba(0, 210, 255, 0.8);
        }

        .loading-screen {
            position: fixed;
            inset: 0;
            background: #000;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 100;
            color: #8A2BE2;
        }

        .loader {
            width: 40px;
            height: 40px;
            border: 2px solid rgba(138, 43, 226, 0.2);
            border-radius: 50%;
            border-top-color: #00D2FF;
            animation: spin 1s linear infinite;
            margin-bottom: 20px;
        }

        @keyframes spin {
            to { transform: rotate(360deg); }
        }
    </style>
</head>
<body>

    <div class="loading-screen" id="loading-screen">
        <div class="loader"></div>
        <p id="loading-text">正在校准时空参数...</p>
    </div>

    <div id="main-title">打开AI时代支付大门</div>

    <div id="ui-overlay">
        <div class="status-badge" id="status">系统初始化...</div>
        <div style="font-size: 10px; opacity: 0.5;">
            [敲击] 涟漪 | [推掌] 跃迁 | [食指] 重置
        </div>
    </div>

    <div id="gesture-hint">Future Is Now</div>

    <video id="camera-feed" autoplay playsinline></video>
    <div id="canvas-container"></div>

    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>

    <script type="importmap">
        {
            "imports": {
                "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
                "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
        import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
        import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';

        const state = {
            gateOpen: false,
            warping: false,
            handDetected: false,
            lastKnockTime: 0,
            knockCooldown: 400,
            isResetting: false,
            cameraReady: false
        };

        let scene, camera, renderer, composer, particles, starfield, gateGeometry;
        const PARTICLE_COUNT = 15000;
        const GATE_WIDTH = 8;
        const GATE_HEIGHT = 12;

        const mainTitle = document.getElementById('main-title');
        const gestureHint = document.getElementById('gesture-hint');

        function initThree() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x000000);
            scene.fog = new THREE.FogExp2(0x000000, 0.02);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.z = 15;

            renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
            renderer.toneMapping = THREE.ReinhardToneMapping;
            document.getElementById('canvas-container').appendChild(renderer.domElement);

            const renderScene = new RenderPass(scene, camera);
            const bloomPass = new UnrealBloomPass(
                new THREE.Vector2(window.innerWidth, window.innerHeight),
                1.5, 0.4, 0.85
            );
            
            composer = new EffectComposer(renderer);
            composer.addPass(renderScene);
            composer.addPass(bloomPass);

            createBackground();
            createGate();
            
            window.addEventListener('resize', onWindowResize);
        }

        function createBackground() {
            const starsGeometry = new THREE.BufferGeometry();
            const pos = [];
            for (let i = 0; i < 2000; i++) {
                pos.push((Math.random() - 0.5) * 100, (Math.random() - 0.5) * 100, -Math.random() * 50);
            }
            starsGeometry.setAttribute('position', new THREE.Float32BufferAttribute(pos, 3));
            const starsMaterial = new THREE.PointsMaterial({ size: 0.1, color: 0x4444ff, transparent: true, opacity: 0.5 });
            starfield = new THREE.Points(starsGeometry, starsMaterial);
            scene.add(starfield);
        }

        function createGate() {
            const positions = new Float32Array(PARTICLE_COUNT * 3);
            const colors = new Float32Array(PARTICLE_COUNT * 3);
            const sizes = new Float32Array(PARTICLE_COUNT);
            const randoms = new Float32Array(PARTICLE_COUNT);

            const colorA = new THREE.Color("#8A2BE2");
            const colorB = new THREE.Color("#00D2FF");

            for (let i = 0; i < PARTICLE_COUNT; i++) {
                const u = Math.random();
                const v = Math.random();
                let x, y;
                if (Math.random() > 0.5) {
                    x = (u - 0.5) * GATE_WIDTH;
                    y = (v > 0.5 ? 0.5 : -0.5) * GATE_HEIGHT + (Math.random() - 0.5) * 2;
                } else {
                    x = (u > 0.5 ? 0.5 : -0.5) * GATE_WIDTH + (Math.random() - 0.5) * 2;
                    y = (v - 0.5) * GATE_HEIGHT;
                }

                positions[i * 3] = x;
                positions[i * 3 + 1] = y;
                positions[i * 3 + 2] = (Math.random() - 0.5) * 1;

                const mix = Math.random();
                colors[i * 3] = THREE.MathUtils.lerp(colorA.r, colorB.r, mix);
                colors[i * 3 + 1] = THREE.MathUtils.lerp(colorA.g, colorB.g, mix);
                colors[i * 3 + 2] = THREE.MathUtils.lerp(colorA.b, colorB.b, mix);

                sizes[i] = Math.random() * 2 + 1;
                randoms[i] = Math.random();
            }

            gateGeometry = new THREE.BufferGeometry();
            gateGeometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
            gateGeometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
            gateGeometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));
            gateGeometry.setAttribute('random', new THREE.BufferAttribute(randoms, 1));

            const material = new THREE.ShaderMaterial({
                uniforms: {
                    time: { value: 0 },
                    uRipple: { value: 0 },
                    uOpenProgress: { value: 0 },
                    uWarp: { value: 0 }
                },
                vertexShader: `
                    uniform float time;
                    uniform float uRipple;
                    uniform float uOpenProgress;
                    uniform float uWarp;
                    attribute float size;
                    attribute float random;
                    varying vec3 vColor;
                    
                    void main() {
                        vColor = color;
                        vec3 pos = position;
                        pos.x += sin(time * 0.5 + random * 10.0) * 0.1;
                        pos.y += cos(time * 0.3 + random * 10.0) * 0.1;

                        float dist = length(pos.xy);
                        if(uRipple > 0.0) {
                            float r = mod(dist - uRipple * 10.0, 5.0);
                            float rippleEff = smoothstep(0.5, 0.0, abs(r)) * (1.0 - uRipple);
                            pos.z += rippleEff * 2.0;
                        }

                        if(uOpenProgress > 0.0) {
                            vec2 dir = normalize(pos.xy + 0.001);
                            pos.xy += dir * pow(uOpenProgress, 1.8) * 22.0;
                            pos.z += uOpenProgress * 12.0;
                        }

                        if(uWarp > 0.0) {
                            pos.z += mod(random * 100.0 + time * 60.0, 120.0) - 60.0;
                        }

                        vec4 mvPosition = modelViewMatrix * vec4(pos, 1.0);
                        gl_PointSize = size * (35.0 / -mvPosition.z);
                        gl_Position = projectionMatrix * mvPosition;
                    }
                `,
                fragmentShader: `
                    varying vec3 vColor;
                    void main() {
                        float d = length(gl_PointCoord - vec2(0.5));
                        if(d > 0.5) discard;
                        gl_FragColor = vec4(vColor, 1.0 - (d * 2.0));
                    }
                `,
                transparent: true,
                blending: THREE.AdditiveBlending,
                vertexColors: true,
                depthWrite: false
            });

            particles = new THREE.Points(gateGeometry, material);
            scene.add(particles);
        }

        async function initMediaPipe() {
            const video = document.getElementById('camera-feed');
            const statusBadge = document.getElementById('status');
            const loadingText = document.getElementById('loading-text');
            
            try {
                if (!window.Hands || !window.Camera) {
                    throw new Error("Core libraries missing");
                }

                const hands = new window.Hands({
                    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
                });

                hands.setOptions({
                    maxNumHands: 1,
                    modelComplexity: 1,
                    minDetectionConfidence: 0.55,
                    minTrackingConfidence: 0.55
                });

                hands.onResults(onResults);

                const cameraPipe = new window.Camera(video, {
                    onFrame: async () => {
                        await hands.send({image: video});
                    },
                    width: 640,
                    height: 480
                });

                await cameraPipe.start();
                state.cameraReady = true;
                statusBadge.innerText = "系统就绪：请面向摄像头";
                document.getElementById('loading-screen').style.opacity = '0';
                setTimeout(() => document.getElementById('loading-screen').style.display = 'none', 500);
            } catch (err) {
                console.error("MediaPipe Error:", err);
                statusBadge.innerText = "加载失败";
                loadingText.innerText = "ERROR: " + err.message;
            }
        }

        let lastZ = 0;

        function onResults(results) {
            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                state.handDetected = true;
                const landmarks = results.multiHandLandmarks[0];
                const indexTip = landmarks[8];
                const currentZ = indexTip.z;
                
                const knockDelta = (currentZ - lastZ);
                if (knockDelta < -0.05 && Date.now() - state.lastKnockTime > state.knockCooldown) {
                    triggerKnock();
                }
                lastZ = currentZ;

                const isPalm = isHandOpen(landmarks);
                if (isPalm && currentZ < -0.12 && !state.gateOpen && !state.isResetting) {
                    triggerOpen();
                }

                if (isIndexOnly(landmarks) && state.gateOpen && !state.isResetting) {
                    triggerReset();
                }
            } else {
                state.handDetected = false;
            }
        }

        function isHandOpen(landmarks) {
            const fingers = [8, 12, 16, 20]; 
            return fingers.every(f => landmarks[f].y < landmarks[f - 1].y);
        }

        function isIndexOnly(landmarks) {
            const indexExtended = landmarks[8].y < landmarks[6].y;
            const middleFolded = landmarks[12].y > landmarks[10].y;
            const ringFolded = landmarks[16].y > landmarks[14].y;
            const pinkyFolded = landmarks[20].y > landmarks[18].y;
            return indexExtended && middleFolded && ringFolded && pinkyFolded;
        }

        function triggerKnock() {
            if (state.gateOpen || state.isResetting) return;
            state.lastKnockTime = Date.now();
            particles.material.uniforms.uRipple.value = 0.1;
            const originalZ = camera.position.z;
            camera.position.z -= 0.4;
            setTimeout(() => camera.position.z = originalZ, 80);
        }

        function triggerOpen() {
            state.gateOpen = true;
            document.getElementById('status').innerText = "穿越中...";
            
            // 隐藏玻璃标题
            mainTitle.style.opacity = '0';
            mainTitle.style.transform = 'translateX(-50%) translateY(-30px)';
            
            // 延迟出现 "Future Is Now" 提示词 (约800ms后)
            setTimeout(() => {
                if(state.gateOpen) gestureHint.style.opacity = 1;
            }, 800);
            
            let progress = particles.material.uniforms.uOpenProgress.value;
            const animateOpen = () => {
                if (!state.gateOpen) return;
                progress += 0.022;
                particles.material.uniforms.uOpenProgress.value = progress;
                
                if (progress > 0.35) {
                    state.warping = true;
                    particles.material.uniforms.uWarp.value = 1.0;
                    camera.fov = THREE.MathUtils.lerp(camera.fov, 108, 0.06);
                    camera.updateProjectionMatrix();
                }

                if (progress < 1.8) requestAnimationFrame(animateOpen);
            };
            animateOpen();
        }

        function triggerReset() {
            state.isResetting = true;
            state.gateOpen = false;
            state.warping = false;
            document.getElementById('status').innerText = "重构中...";
            
            // 立即隐藏提示词
            gestureHint.style.opacity = 0;

            // 重新显示玻璃标题
            mainTitle.style.opacity = '1';
            mainTitle.style.transform = 'translateX(-50%) translateY(0)';

            let progress = particles.material.uniforms.uOpenProgress.value;
            let currentFov = camera.fov;

            const animateReset = () => {
                let active = false;

                if (progress > 0) {
                    progress -= 0.07;
                    if (progress < 0) progress = 0;
                    particles.material.uniforms.uOpenProgress.value = progress;
                    active = true;
                }

                if (currentFov > 75) {
                    currentFov -= 2.0;
                    if (currentFov < 75) currentFov = 75;
                    camera.fov = currentFov;
                    camera.updateProjectionMatrix();
                    active = true;
                }

                if (progress < 0.25) {
                    particles.material.uniforms.uWarp.value = 0;
                }

                if (active) {
                    requestAnimationFrame(animateReset);
                } else {
                    state.isResetting = false;
                    document.getElementById('status').innerText = "重置完成";
                }
            };
            animateReset();
        }

        function animate() {
            requestAnimationFrame(animate);
            const time = performance.now() * 0.001;
            
            if (particles) {
                particles.material.uniforms.time.value = time;
                if (particles.material.uniforms.uRipple.value > 0) {
                    particles.material.uniforms.uRipple.value += 0.035;
                    if (particles.material.uniforms.uRipple.value > 1.3) {
                        particles.material.uniforms.uRipple.value = 0;
                    }
                }
            }

            if (starfield) {
                starfield.rotation.y = time * 0.02;
                if (state.warping) {
                    starfield.position.z += 2.0;
                    if (starfield.position.z > 50) starfield.position.z = 0;
                } else if (starfield.position.z > 0) {
                    starfield.position.z = THREE.MathUtils.lerp(starfield.position.z, 0, 0.1);
                }
            }

            composer.render();
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
            composer.setSize(window.innerWidth, window.innerHeight);
        }

        window.onload = () => {
            initThree();
            initMediaPipe();
            animate();
        };
    </script>
</body>
</html>
