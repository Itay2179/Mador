# Mador
Game of shooting
<html><head><base href="https://websim.neocities.org/">
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>משחק יריות 3D - מתקפת הזומבים עם רובה נראה לעין</title>
<style>
  body { margin: 0; overflow: hidden; font-family: Arial, sans-serif; }
  #gameContainer { position: absolute; width: 100%; height: 100%; }
  #hud { position: absolute; top: 10px; left: 10px; color: white; font-size: 18px; text-shadow: 2px 2px 2px rgba(0,0,0,0.5); }
  #pauseButton {
    position: absolute;
    top: 10px;
    right: 10px;
    background: rgba(255, 255, 255, 0.7);
    border: none;
    border-radius: 5px;
    padding: 5px 10px;
    font-size: 20px;
    cursor: pointer;
    transition: background 0.3s;
  }
  #pauseButton:hover {
    background: rgba(255, 255, 255, 0.9);
  }
  #crosshair { position: absolute; top: 50%; left: 50%; width: 20px; height: 20px; transform: translate(-50%, -50%); }
  #gameOver { display: none; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: rgba(0,0,0,0.7); padding: 20px; border-radius: 10px; text-align: center; color: white; }
  #winMessage {
    display: none;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgba(0,0,0,0.7);
    padding: 20px;
    border-radius: 10px;
    text-align: center;
    color: white;
  }
  #instructions { position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%); color: white; text-align: center; text-shadow: 2px 2px 2px rgba(0,0,0,0.5); }
  #pasiMessage { display: none; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: rgba(0,0,0,0.7); padding: 20px; border-radius: 10px; text-align: center; color: white; }
</style>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/cannon.js/0.6.2/cannon.min.js"></script>
</head>
<body>
<div id="gameContainer"></div>
<div id="hud">נקודות: <span id="score">0</span> | חיים: <span id="health">100</span></div>
<button id="pauseButton">⏸️</button>
<svg id="crosshair" viewBox="0 0 100 100">
  <circle cx="50" cy="50" r="45" stroke="white" stroke-width="2" fill="none"/>
  <line x1="50" y1="0" x2="50" y2="100" stroke="white" stroke-width="2"/>
  <line x1="0" y1="50" x2="100" y2="50" stroke="white" stroke-width="2"/>
</svg>
<div id="gameOver">
  <h2>המשחק נגמר</h2>
  <p>הניקוד שלך: <span id="finalScore"></span></p>
  <button onclick="restartGame()">שחק שוב</button>
</div>
<div id="winMessage">
  <h2>You Won!</h2>
  <button onclick="restartGame()">Play Again</button>
</div>
<div id="instructions">
  <p>WASD - תזוזה | עכבר - כיוון ימינה ושמאלה | קליק שמאלי - ירייה</p>
</div>
<div id="pasiMessage">
  <h2>Pasi gay</h2>
</div>
<audio id="painSound" src="https://assets.mixkit.co/active_storage/sfx/2221/2221-preview.mp3" preload="auto"></audio>
<script>
let scene, camera, renderer, world, playerBody, timeStep = 1/60;
let enemies = [], bullets = [];
let score = 0, health = 100;
let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false;
let playerVelocity = new THREE.Vector3();
let playerDirection = new THREE.Vector3();
let gunModel;
let isPaused = false;

function init() {
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  renderer = new THREE.WebGLRenderer();
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.getElementById('gameContainer').appendChild(renderer.domElement);

  world = new CANNON.World();
  world.gravity.set(0, -9.82, 0);

  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(0, 1, 0);
  scene.add(light);

  const ambientLight = new THREE.AmbientLight(0x404040);
  scene.add(ambientLight);

  const skyGeometry = new THREE.SphereGeometry(500, 32, 32);
  const skyTexture = new THREE.TextureLoader().load('https://i.imgur.com/ZH3tgTc.jpg');
  const skyMaterial = new THREE.MeshBasicMaterial({ map: skyTexture, side: THREE.BackSide });
  const sky = new THREE.Mesh(skyGeometry, skyMaterial);
  scene.add(sky);

  const floorGeometry = new THREE.PlaneGeometry(100, 100);
  const floorTexture = new THREE.TextureLoader().load('https://i.imgur.com/JcLe0mF.jpg');
  floorTexture.wrapS = THREE.RepeatWrapping;
  floorTexture.wrapT = THREE.RepeatWrapping;
  floorTexture.repeat.set(10, 10);
  const floorMaterial = new THREE.MeshLambertMaterial({ map: floorTexture, side: THREE.DoubleSide });
  const floorMesh = new THREE.Mesh(floorGeometry, floorMaterial);
  floorMesh.rotation.x = -Math.PI / 2;
  scene.add(floorMesh);

  const floorShape = new CANNON.Plane();
  const floorBody = new CANNON.Body({ mass: 0 });
  floorBody.addShape(floorShape);
  floorBody.quaternion.setFromAxisAngle(new CANNON.Vec3(1, 0, 0), -Math.PI / 2);
  world.addBody(floorBody);

  playerBody = new CANNON.Body({
    mass: 5,
    shape: new CANNON.Sphere(1),
    position: new CANNON.Vec3(0, 2, 5)
  });
  world.addBody(playerBody);

  camera.position.set(0, 2, 5);

  createGunModel();

  document.addEventListener('click', shoot);
  document.addEventListener('mousemove', onMouseMove);
  document.addEventListener('keydown', onKeyDown);
  document.getElementById('pauseButton').addEventListener('click', togglePause);

  spawnEnemy();
}

function togglePause() {
  isPaused = !isPaused;
  updatePauseButton();
  if (isPaused) {
    cancelAnimationFrame(animationId);
  } else {
    animate();
  }
}

function updatePauseButton() {
  const pauseButton = document.getElementById('pauseButton');
  pauseButton.textContent = isPaused ? '▶️' : '⏸️';
  pauseButton.setAttribute('aria-label', isPaused ? 'Resume' : 'Pause');
}

function createGunModel() {
  const gunGroup = new THREE.Group();

  const gunBodyGeometry = new THREE.BoxGeometry(0.1, 0.1, 0.5);
  const gunBodyMaterial = new THREE.MeshPhongMaterial({ color: 0x333333 });
  const gunBody = new THREE.Mesh(gunBodyGeometry, gunBodyMaterial);
  gunGroup.add(gunBody);

  const barrelGeometry = new THREE.CylinderGeometry(0.02, 0.02, 0.3, 16);
  const barrelMaterial = new THREE.MeshPhongMaterial({ color: 0x666666 });
  const barrel = new THREE.Mesh(barrelGeometry, barrelMaterial);
  barrel.rotation.x = Math.PI / 2;
  barrel.position.z = -0.4;
  gunGroup.add(barrel);

  const sightGeometry = new THREE.BoxGeometry(0.02, 0.05, 0.02);
  const sightMaterial = new THREE.MeshPhongMaterial({ color: 0x000000 });
  const sight = new THREE.Mesh(sightGeometry, sightMaterial);
  sight.position.y = 0.075;
  sight.position.z = -0.1;
  gunGroup.add(sight);

  const handleGeometry = new THREE.BoxGeometry(0.08, 0.12, 0.03);
  const handleMaterial = new THREE.MeshPhongMaterial({ color: 0x8B4513 });
  const handle = new THREE.Mesh(handleGeometry, handleMaterial);
  handle.position.y = -0.08;
  handle.position.z = 0.1;
  gunGroup.add(handle);

  gunGroup.position.set(0.2, -0.2, -0.5);
  camera.add(gunGroup);
  scene.add(camera);

  gunModel = gunGroup;
}

function createSwordMesh() {
  const swordGroup = new THREE.Group();

  const bladeGeometry = new THREE.BoxGeometry(0.05, 0.5, 0.01);
  const bladeMaterial = new THREE.MeshPhongMaterial({ color: 0xcccccc });
  const blade = new THREE.Mesh(bladeGeometry, bladeMaterial);
  blade.position.y = 0.25;
  swordGroup.add(blade);

  const handleGeometry = new THREE.CylinderGeometry(0.02, 0.02, 0.15, 8);
  const handleMaterial = new THREE.MeshPhongMaterial({ color: 0x8B4513 });
  const handle = new THREE.Mesh(handleGeometry, handleMaterial);
  handle.position.y = -0.075;
  swordGroup.add(handle);

  const guardGeometry = new THREE.BoxGeometry(0.15, 0.02, 0.02);
  const guardMaterial = new THREE.MeshPhongMaterial({ color: 0xFFD700 });
  const guard = new THREE.Mesh(guardGeometry, guardMaterial);
  swordGroup.add(guard);

  return swordGroup;
}

function createHumanoidMesh() {
  const group = new THREE.Group();

  const bodyGeometry = new THREE.CylinderGeometry(0.3, 0.3, 1, 8);
  const bodyMaterial = new THREE.MeshPhongMaterial({ color: 0x964B00 });
  const body = new THREE.Mesh(bodyGeometry, bodyMaterial);
  body.position.y = 0.5;
  group.add(body);

  const headGeometry = new THREE.SphereGeometry(0.25, 16, 16);
  const headMaterial = new THREE.MeshPhongMaterial({ color: 0xffcccc });
  const head = new THREE.Mesh(headGeometry, headMaterial);
  head.position.y = 1.25;
  group.add(head);

  const armGeometry = new THREE.CylinderGeometry(0.08, 0.08, 0.5, 8);
  const armMaterial = new THREE.MeshPhongMaterial({ color: 0x964B00 });

  const leftArm = new THREE.Mesh(armGeometry, armMaterial);
  leftArm.position.set(-0.4, 0.7, 0);
  leftArm.rotation.z = Math.PI / 4;
  group.add(leftArm);

  const rightArm = new THREE.Mesh(armGeometry, armMaterial);
  rightArm.position.set(0.4, 0.7, 0);
  rightArm.rotation.z = -Math.PI / 3;
  group.add(rightArm);

  const sword = createSwordMesh();
  sword.position.set(0.6, 0.5, 0.2); 
  sword.rotation.z = Math.PI / 2; 
  group.add(sword);

  const legGeometry = new THREE.CylinderGeometry(0.1, 0.1, 0.5, 8);
  const legMaterial = new THREE.MeshPhongMaterial({ color: 0x964B00 });

  const leftLeg = new THREE.Mesh(legGeometry, legMaterial);
  leftLeg.position.set(-0.2, -0.25, 0);
  group.add(leftLeg);

  const rightLeg = new THREE.Mesh(legGeometry, legMaterial);
  rightLeg.position.set(0.2, -0.25, 0);
  group.add(rightLeg);

  return group;
}

function spawnEnemy() {
  const enemy = createHumanoidMesh();
  enemy.position.set(
    Math.random() * 80 - 40,
    1,
    Math.random() * 80 - 40
  );
  scene.add(enemy);
  enemies.push(enemy);

  setTimeout(spawnEnemy, 2000);
}

function playPainSound() {
  const painSound = document.getElementById('painSound');
  painSound.currentTime = 0;
  painSound.play();
}

function shoot() {
  const bulletGeometry = new THREE.SphereGeometry(0.05);
  const bulletMaterial = new THREE.MeshBasicMaterial({ color: 0xffff00 });
  const bullet = new THREE.Mesh(bulletGeometry, bulletMaterial);
  bullet.position.set(
    camera.position.x,
    camera.position.y,
    camera.position.z
  );
  bullet.velocity = new THREE.Vector3(0, 0, -1).applyQuaternion(camera.quaternion);
  scene.add(bullet);
  bullets.push(bullet);

  gunModel.position.z += 0.1;
  setTimeout(() => {
    gunModel.position.z -= 0.1;
  }, 50);
}

function onMouseMove(event) {
  if (document.pointerLockElement === renderer.domElement) {
    const movementX = event.movementX || event.mozMovementX || event.webkitMovementX || 0;
    camera.rotation.y -= movementX * 0.002;
  }
}

function onKeyDown(event) {
  switch (event.code) {
    case 'KeyW': moveForward = true; break;
    case 'KeyS': moveBackward = true; break;
    case 'KeyA': playerWon(); break; 
    case 'KeyD': moveRight = true; break;
    case 'Space': togglePause(); break; 
    case 'KeyP': showPasiMessage(); break;
  }
}

function onKeyUp(event) {
  switch (event.code) {
    case 'KeyW': moveForward = false; break;
    case 'KeyS': moveBackward = false; break;
    case 'KeyA': moveLeft = false; break;
    case 'KeyD': moveRight = false; break;
  }
}

function showPasiMessage() {
  const pasiMessage = document.getElementById('pasiMessage');
  pasiMessage.style.display = 'block';
  setTimeout(() => {
    pasiMessage.style.display = 'none';
  }, 2000);
}

function playerWon() {
  document.getElementById('winMessage').style.display = 'block';
  cancelAnimationFrame(animationId);
  document.exitPointerLock();
}

function updatePlayerMovement() {
  playerVelocity.x = 0;
  playerVelocity.z = 0;

  if (moveForward) playerVelocity.z = -1;
  if (moveBackward) playerVelocity.z = 1;
  if (moveLeft) playerVelocity.x = -1;
  if (moveRight) playerVelocity.x = 1;

  playerVelocity.normalize().multiplyScalar(0.1);

  playerDirection.copy(playerVelocity).applyAxisAngle(new THREE.Vector3(0, 1, 0), camera.rotation.y);

  playerBody.velocity.x = playerDirection.x * 5;
  playerBody.velocity.z = playerDirection.z * 5;

  camera.position.copy(playerBody.position);
}

function updateBullets() {
  for (let i = bullets.length - 1; i >= 0; i--) {
    bullets[i].position.add(bullets[i].velocity);

    if (bullets[i].position.length() > 100) {
      scene.remove(bullets[i]);
      bullets.splice(i, 1);
      continue;
    }

    for (let j = enemies.length - 1; j >= 0; j--) {
      if (bullets[i].position.distanceTo(enemies[j].position) < 0.5) {
        scene.remove(bullets[i]);
        bullets.splice(i, 1);
        scene.remove(enemies[j]);
        enemies.splice(j, 1);
        score += 10;
        document.getElementById('score').textContent = score;
        break;
      }
    }
  }
}

function updateEnemies() {
  for (let i = enemies.length - 1; i >= 0; i--) {
    const direction = new THREE.Vector3().subVectors(camera.position, enemies[i].position);
    direction.y = 0;
    direction.normalize();
    enemies[i].position.add(direction.multiplyScalar(0.05));
    enemies[i].lookAt(camera.position);
    
    if (enemies[i].position.distanceTo(camera.position) < 1) {
      scene.remove(enemies[i]);
      enemies.splice(i, 1);
      const oldHealth = health;
      health -= 10;
      document.getElementById('health').textContent = health;
      
      // Play pain sound at specific thresholds
      if ((oldHealth > 90 && health <= 90) || 
          (oldHealth > 70 && health <= 70) || 
          (oldHealth > 50 && health <= 50) || 
          (oldHealth > 30 && health <= 30) || 
          (oldHealth > 10 && health <= 10)) {
        playPainSound();
      }
      
      if (health <= 0) {
        gameOver();
      }
    }
  }
}

function gameOver() {
  document.getElementById('gameOver').style.display = 'block';
  document.getElementById('finalScore').textContent = score;
  cancelAnimationFrame(animationId);
  document.exitPointerLock();
}

function restartGame() {
  isPaused = false;
  updatePauseButton();
  score = 0;
  health = 100;
  document.getElementById('score').textContent = score;
  document.getElementById('health').textContent = health;
  document.getElementById('gameOver').style.display = 'none';
  document.getElementById('winMessage').style.display = 'none';

  for (let enemy of enemies) {
    scene.remove(enemy);
  }
  enemies = [];

  for (let bullet of bullets) {
    scene.remove(bullet);
  }
  bullets = [];

  playerBody.position.set(0, 2, 5);
  playerBody.velocity.set(0, 0, 0);
  camera.position.copy(playerBody.position);
  camera.rotation.set(0, 0, 0);

  animate();
}

let animationId;

function animate() {
  if (!isPaused) {
    animationId = requestAnimationFrame(animate);
    world.step(timeStep);
    updatePlayerMovement();
    updateBullets();
    updateEnemies();
    renderer.render(scene, camera);
  }
}

init();
animate();

renderer.domElement.requestPointerLock = renderer.domElement.requestPointerLock ||
                                         renderer.domElement.mozRequestPointerLock ||
                                         renderer.domElement.webkitRequestPointerLock;

document.addEventListener('click', function() {
  renderer.domElement.requestPointerLock();
});
</script>
</body>
</html>
