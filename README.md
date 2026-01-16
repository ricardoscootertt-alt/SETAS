<!DOCTYPE html>
<html lang="pt-pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Explorador Estelar 3D</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        #ui-container {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #0ff;
            text-shadow: 0 0 10px #0ff;
            pointer-events: none;
            z-index: 10;
        }
        .stat-label { font-size: 14px; text-transform: uppercase; letter-spacing: 2px; }
        .stat-value { font-size: 32px; font-weight: bold; }
        
        #game-over {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.85);
            padding: 40px;
            border: 2px solid #0ff;
            border-radius: 15px;
            text-align: center;
            color: white;
            display: none;
            z-index: 20;
            box-shadow: 0 0 30px rgba(0, 255, 255, 0.5);
        }
        button {
            background: transparent;
            color: #0ff;
            border: 1px solid #0ff;
            padding: 10px 25px;
            font-size: 18px;
            cursor: pointer;
            margin-top: 20px;
            transition: 0.3s;
            text-transform: uppercase;
        }
        button:hover { background: #0ff; color: #000; }
        #controls-hint {
            position: absolute;
            bottom: 20px;
            width: 100%;
            text-align: center;
            color: rgba(0, 255, 255, 0.6);
            font-size: 12px;
            pointer-events: none;
        }
    </style>
</head>
<body>

    <div id="ui-container">
        <div class="stat-label">Pontuação</div>
        <div id="score" class="stat-value">0</div>
        <div class="stat-label" style="margin-top: 10px;">Distância</div>
        <div id="distance" class="stat-value">0m</div>
    </div>

    <div id="game-over">
        <h1 style="color: #f0f; text-shadow: 0 0 10px #f0f;">MISSÃO ABORTADA</h1>
        <p>A tua nave colidiu com um asteroide.</p>
        <p>Pontuação Final: <span id="final-score">0</span></p>
        <button onclick="restartGame()">Reiniciar Sistema</button>
    </div>

    <div id="controls-hint">Usa as SETAS ou A/D para mover | Toque nos lados do ecrã no telemóvel</div>

    <!-- Carregar Three.js da CDN -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

    <script>
        let scene, camera, renderer, player, clock;
        let obstacles = [];
        let particles = [];
        let gameActive = true;
        let score = 0;
        let speed = 0.5;
        let laneWidth = 4;
        let targetX = 0;
        
        // Configuração Inicial
        function init() {
            scene = new THREE.Scene();
            scene.fog = new THREE.FogExp2(0x000011, 0.02);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 3, 7);
            camera.lookAt(0, 0, -5);

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.body.appendChild(renderer.domElement);

            clock = new THREE.Clock();

            // Iluminação
            const ambientLight = new THREE.AmbientLight(0x404040, 2);
            scene.add(ambientLight);

            const moonLight = new THREE.DirectionalLight(0x00ffff, 1);
            moonLight.position.set(1, 1, 1);
            scene.add(moonLight);

            // Criar Nave (Jogador)
            createPlayer();

            // Criar Chão (Grid Neon)
            createGrid();

            // Criar Estrelas de Fundo
            createStars();

            // Eventos
            window.addEventListener('keydown', handleInput);
            window.addEventListener('resize', onWindowResize);
            window.addEventListener('touchstart', handleTouch);

            animate();
        }

        function createPlayer() {
            const group = new THREE.Group();

            // Corpo principal
            const bodyGeo = new THREE.ConeGeometry(0.5, 2, 4);
            const bodyMat = new THREE.MeshPhongMaterial({ color: 0xcccccc, flatShading: true });
            const body = new THREE.Mesh(bodyGeo, bodyMat);
            body.rotation.x = Math.PI / 2;
            group.add(body);

            // Asas
            const wingGeo = new THREE.BoxGeometry(2, 0.1, 0.8);
            const wingMat = new THREE.MeshPhongMaterial({ color: 0x00ffff });
            const wings = new THREE.Mesh(wingGeo, wingMat);
            group.add(wings);

            // Propulsor (Luz)
            const engineGeo = new THREE.SphereGeometry(0.2, 8, 8);
            const engineMat = new THREE.MeshBasicMaterial({ color: 0xff00ff });
            const engine = new THREE.Mesh(engineGeo, engineMat);
            engine.position.z = 1;
            group.add(engine);

            const light = new THREE.PointLight(0xff00ff, 2, 5);
            light.position.z = 1.2;
            group.add(light);

            player = group;
            scene.add(player);
        }

        function createGrid() {
            const size = 100;
            const divisions = 20;
            const grid = new THREE.GridHelper(size, divisions, 0xff00ff, 0x004444);
            grid.position.y = -1;
            grid.position.z = -size/4;
            scene.add(grid);
            
            // Segunda grid para efeito infinito
            const grid2 = grid.clone();
            grid2.position.z = -size/4 - size;
            scene.add(grid2);
            
            scene.userData.grids = [grid, grid2];
        }

        function createStars() {
            const geo = new THREE.BufferGeometry();
            const vertices = [];
            for(let i=0; i<2000; i++) {
                vertices.push(
                    Math.random() * 400 - 200,
                    Math.random() * 400 - 200,
                    Math.random() * 400 - 400
                );
            }
            geo.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
            const mat = new THREE.PointsMaterial({ color: 0xffffff, size: 0.5 });
            const stars = new THREE.Points(geo, mat);
            scene.add(stars);
        }

        function spawnObstacle() {
            const types = ['box', 'sphere', 'tetra'];
            const type = types[Math.floor(Math.random() * types.length)];
            let geo;
            
            if(type === 'box') geo = new THREE.BoxGeometry(1.5, 1.5, 1.5);
            else if(type === 'sphere') geo = new THREE.SphereGeometry(1, 12, 12);
            else geo = new THREE.TetrahedronGeometry(1.2);

            const mat = new THREE.MeshPhongMaterial({ 
                color: 0x333333, 
                emissive: 0x220022,
                flatShading: true 
            });
            const mesh = new THREE.Mesh(geo, mat);
            
            mesh.position.x = (Math.random() - 0.5) * 15;
            mesh.position.y = 0;
            mesh.position.z = -60;
            
            mesh.userData.rotationSpeed = {
                x: Math.random() * 0.05,
                y: Math.random() * 0.05
            };

            scene.add(mesh);
            obstacles.push(mesh);
        }

        function handleInput(e) {
            if(!gameActive) return;
            if(e.key === 'ArrowLeft' || e.key === 'a') targetX -= laneWidth;
            if(e.key === 'ArrowRight' || e.key === 'd') targetX += laneWidth;
            targetX = Math.max(-10, Math.min(10, targetX));
        }

        function handleTouch(e) {
            if(!gameActive) return;
            const touchX = e.touches[0].clientX;
            if(touchX < window.innerWidth / 2) targetX -= laneWidth;
            else targetX += laneWidth;
            targetX = Math.max(-10, Math.min(10, targetX));
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        function updateGame() {
            // Movimento suave do jogador
            player.position.x += (targetX - player.position.x) * 0.1;
            player.rotation.z = (player.position.x - targetX) * 0.1;
            player.rotation.y = (targetX - player.position.x) * 0.05;

            // Movimento das Grids (Efeito infinito)
            scene.userData.grids.forEach(grid => {
                grid.position.z += speed;
                if(grid.position.z > 25) {
                    grid.position.z -= 200;
                }
            });

            // Gerir Obstáculos
            if(Math.random() < 0.03 + (speed * 0.02)) spawnObstacle();

            for(let i = obstacles.length - 1; i >= 0; i--) {
                const obs = obstacles[i];
                obs.position.z += speed;
                obs.rotation.x += obs.userData.rotationSpeed.x;
                obs.rotation.y += obs.userData.rotationSpeed.y;

                // Colisão (Cálculo de distância simples)
                const dist = player.position.distanceTo(obs.position);
                if(dist < 1.5) {
                    gameOver();
                }

                // Remover se passar do jogador
                if(obs.position.z > 10) {
                    scene.remove(obs);
                    obstacles.splice(i, 1);
                    score += 10;
                    document.getElementById('score').innerText = score;
                }
            }

            // Aumentar velocidade
            speed += 0.0001;
            document.getElementById('distance').innerText = Math.floor(speed * 1000) + 'm';
        }

        function gameOver() {
            gameActive = false;
            document.getElementById('game-over').style.display = 'block';
            document.getElementById('final-score').innerText = score;
            
            // Pequena animação de explosão (opcional)
            player.visible = false;
        }

        function restartGame() {
            // Limpar obstáculos
            obstacles.forEach(o => scene.remove(o));
            obstacles = [];
            
            // Reset variáveis
            score = 0;
            speed = 0.5;
            targetX = 0;
            player.position.set(0, 0, 0);
            player.visible = true;
            gameActive = true;
            
            document.getElementById('score').innerText = '0';
            document.getElementById('game-over').style.display = 'none';
        }

        function animate() {
            requestAnimationFrame(animate);
            
            if(gameActive) {
                updateGame();
            }
            
            renderer.render(scene, camera);
        }

        // Iniciar tudo
        init();

    </script>
</body>
</html>
