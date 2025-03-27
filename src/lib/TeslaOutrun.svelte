<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import * as THREE from 'three';

  let gameContainer: HTMLElement;
  let scene: THREE.Scene;
  let camera: THREE.PerspectiveCamera;
  let renderer: THREE.WebGLRenderer;
  let road: THREE.Group;
  let player: THREE.Group;
  let obstacles: THREE.Object3D[] = [];
  let trees: { trunk: THREE.Mesh; foliage: THREE.Mesh; z: number }[] = [];
  let buildings: { mesh: THREE.Mesh; z: number }[] = [];
  let roadSegments: { mesh: THREE.Mesh; z: number }[] = [];
  let roadStripes: { mesh: THREE.Mesh; z: number }[] = []; // Track road stripes separately for animation
  
  // Game state variables
  let score = 0;
  let speed = 0;
  let maxSpeed = 500;
  let acceleration = 2;
  let brakeForce = 5;
  let deceleration = 0.95;
  let gameActive = false;
  let lane = 1; // 0: left, 1: center, 2: right
  let lanePositions = [-4, 0, 4];
  let roadLength = 2000;
  let segmentLength = 20;
  let segmentCount = roadLength / segmentLength;
  let cameraHeight = 5;
  let cameraDistance = -15;
  let elapsedTime = 0;
  let lastObstacleTime = 0;
  let obstacleInterval = 2; // Seconds between obstacles
  let carsOvertaken = 0;
  
  // Lives system
  let lives = 3;
  let lifeIcons: THREE.Group[] = [];
  
  // High score system
  let highScores: {name: string, score: number}[] = [];
  let playerName = '';
  let showHighScores = false;
  
  // Jump variables
  let isJumping = false;
  let jumpHeight = 0;
  let jumpVelocity = 0;
  let jumpGravity = 0.01; // Optimized gravity for parabolic arc
  let maxJumpHeight = 5; // Double the car's height (1.5 units * 2)
  let jumpInitialVelocity = 15; // Increased velocity for smoother ascent
  let jumpDuration = 2; // Reduced duration for quicker jumps
  let jumpStartTime = 0; // When the jump started
  
  // Game pause variables
  let isPaused = false;
  
  // Smoke particles
  let smokeParticles: THREE.Mesh[] = [];
  let lastSmokeTime = 0;
  let smokeInterval = 0.05; // Seconds between smoke particles
  let smokeActive = false;
  let smokeStartTime = 0;
  let smokeDuration = 0; // Random duration up to 10 seconds
  
  // Sun
  let sun: THREE.Mesh | null = null;
  let sunLight: THREE.DirectionalLight | null = null;
  let sunVisible = false;
  
  // Lightning effect
  let lightningActive = false;
  let lightningTime = 0;
  let lightningDuration = 0.5; // Duration of lightning effect in seconds
  let lightningBolt: THREE.Group | null = null;
  let lightningFlash: THREE.DirectionalLight | null = null;
  
  // Colors
  const skyColor = 0x87CEEB; // Light blue sky
  const roadColor = 0x333333; // Dark gray road
  const stripColor = 0xFFFFFF; // White stripes
  const grassColor = 0x3CB371; // Medium sea green for side areas
  
  // Animation
  let animationFrameId: number;
  let clock: THREE.Clock;
  
  onMount(() => {
    init();
    // Add event listeners
    if (typeof document !== 'undefined') {
      document.addEventListener('keydown', onKeyDown);
      window.addEventListener('resize', onWindowResize);
    }
  });
  
  onDestroy(() => {
    // Clean up
    if (animationFrameId) {
      cancelAnimationFrame(animationFrameId);
    }
    if (typeof document !== 'undefined') {
      document.removeEventListener('keydown', onKeyDown);
      window.removeEventListener('resize', onWindowResize);
    }
  });
  
  function init() {
    // Create scene
    scene = new THREE.Scene();
    scene.background = new THREE.Color(skyColor);
    scene.fog = new THREE.Fog(skyColor, 50, 600);
    
    // Create camera
    const aspectRatio = typeof window !== 'undefined' ? window.innerWidth / window.innerHeight : 16/9;
    camera = new THREE.PerspectiveCamera(75, aspectRatio, 0.1, 1000);
    camera.position.set(0, cameraHeight, cameraDistance);
    camera.lookAt(0, 0, 100);
    
    // Create renderer
    renderer = new THREE.WebGLRenderer({ antialias: true });
    if (typeof window !== 'undefined') {
      renderer.setSize(window.innerWidth, window.innerHeight);
    }
    renderer.shadowMap.enabled = true;
    gameContainer.appendChild(renderer.domElement);
    
    // Add lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);
    
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(100, 100, 50);
    directionalLight.castShadow = true;
    directionalLight.shadow.mapSize.width = 2048;
    directionalLight.shadow.mapSize.height = 2048;
    scene.add(directionalLight);
    
    // Create road
    createRoad();
    
    // Create player car
    createPlayerCar();
    
    // Create life display
    createLifeDisplay();
    
    // Load high scores
    loadHighScores();
    
    // Initialize clock
    clock = new THREE.Clock();
    
    // Start animation loop
    animate();
  }
  
  function createLifeDisplay() {
    // Clear existing life icons
    lifeIcons.forEach(icon => scene.remove(icon));
    lifeIcons = [];
    
    // Create new life icons (small Tesla cars)
    for (let i = 0; i < lives; i++) {
      const lifeIcon = new THREE.Group();
      
      // Car body (simplified)
      const bodyGeometry = new THREE.BoxGeometry(0.5, 0.2, 1);
      const bodyMaterial = new THREE.MeshStandardMaterial({ color: 0xe82127 }); // Tesla red
      const carBody = new THREE.Mesh(bodyGeometry, bodyMaterial);
      carBody.position.y = 0.2;
      lifeIcon.add(carBody);
      
      // Position in top left corner of screen
      // Convert from screen space to world space
      const vector = new THREE.Vector3();
      vector.set(-0.85 + i * 0.12, 0.9, 0.5); // x, y in normalized device coordinates (-1 to +1)
      vector.unproject(camera);
      
      const dir = vector.sub(camera.position).normalize();
      const distance = -camera.position.z / dir.z;
      const pos = camera.position.clone().add(dir.multiplyScalar(distance));
      
      lifeIcon.position.copy(pos);
      lifeIcon.lookAt(camera.position);
      
      scene.add(lifeIcon);
      lifeIcons.push(lifeIcon);
    }
  }
  
  function updateLifeDisplay() {
    // Remove all life icons
    lifeIcons.forEach(icon => scene.remove(icon));
    lifeIcons = [];
    
    // Recreate life icons based on current lives
    createLifeDisplay();
    
    // Update the lives display in the DOM
    if (typeof document !== 'undefined') {
      const livesDisplay = document.getElementById('lives-display');
      if (livesDisplay) livesDisplay.textContent = `Lives: ${lives}`;
    }
  }
  
  function loadHighScores() {
    if (typeof localStorage !== 'undefined') {
      const storedScores = localStorage.getItem('teslaOutrunHighScores');
      if (storedScores) {
        highScores = JSON.parse(storedScores);
      }
    }
  }
  
  function saveHighScore(name: string, score: number) {
    // Add new score
    highScores.push({ name, score: Math.floor(score) });
    
    // Sort by score (descending)
    highScores.sort((a, b) => b.score - a.score);
    
    // Keep only top 5
    if (highScores.length > 5) {
      highScores = highScores.slice(0, 5);
    }
    
    // Save to localStorage
    if (typeof localStorage !== 'undefined') {
      localStorage.setItem('teslaOutrunHighScores', JSON.stringify(highScores));
    }
  }
  
  function animate() {
    animationFrameId = requestAnimationFrame(animate);
    
    const deltaTime = clock.getDelta();
    
    if (gameActive) {
      updateGame(deltaTime);
    }
    
    renderer.render(scene, camera);
  }
  
  function updateGame(deltaTime: number) {
    // Update elapsed time
    elapsedTime += deltaTime;
    
    // If game is paused, don't update anything
    if (isPaused) {
      return;
    }
    
    // Auto-accelerate if speed is low
    if (speed < maxSpeed * 0.4) {
      speed += acceleration;
    }
    
    // Apply deceleration
    speed *= deceleration;
    
    // Clamp speed
    speed = Math.max(0, Math.min(maxSpeed, speed));
    
    // Update displays
    if (typeof document !== 'undefined') {
      const speedDisplay = document.getElementById('speed-display');
      if (speedDisplay) speedDisplay.textContent = `Speed: ${Math.round(speed)} km/h`;
    }
    
    // Handle jumping
    if (isJumping) {
      // Apply jump physics with smoother parabolic arc
      jumpHeight += jumpVelocity;
      jumpVelocity -= jumpGravity;
      
      // Enforce maximum jump height
      if (jumpHeight > maxJumpHeight) {
        jumpHeight = maxJumpHeight;
        jumpVelocity = 0; // Start falling when reaching max height
      }
      
      // Check if jump is complete (returned to ground)
      if (jumpHeight <= 0) {
        isJumping = false;
        jumpHeight = 0;
        jumpVelocity = 0;
      }
      
      // Update player's y position based on jump height
      player.position.y = jumpHeight;
    } else {
      // Make sure player is on the ground
      player.position.y = 0;
    }
    
    // Move the car towards target lane
    const targetX = lanePositions[lane];
    player.position.x += (targetX - player.position.x) * 0.1;
    
    // Check collisions
    checkCollisions();
    
    // Update score
    updateScore(deltaTime);
    
    // Handle smoke effect
    if (smokeActive && elapsedTime - lastSmokeTime > smokeInterval) {
      createSmokeParticle();
      lastSmokeTime = elapsedTime;
    }
    
    // Update existing smoke particles
    updateSmokeParticles(deltaTime);
    
    // Handle lightning effect
    if (lightningActive) {
      updateLightningEffect(deltaTime);
    }
    
    // Move obstacles and road
    const moveDistance = speed * deltaTime;
    
    // Move existing obstacles
    for (let i = obstacles.length - 1; i >= 0; i--) {
      const obstacle = obstacles[i];
      obstacle.position.z -= moveDistance;
      
      // Remove obstacles that are behind the camera
      if (obstacle.position.z < cameraDistance) {
        scene.remove(obstacle);
        obstacles.splice(i, 1);
      }
    }
    
    // Check if it's time to spawn a new obstacle
    if (elapsedTime - lastObstacleTime > obstacleInterval - (speed / maxSpeed) * (obstacleInterval - 0.5)) {
      spawnObstacle();
      lastObstacleTime = elapsedTime;
    }
    
    // Move road segments
    for (let i = 0; i < roadSegments.length; i++) {
      const segment = roadSegments[i];
      segment.z -= moveDistance;
      segment.mesh.position.z -= moveDistance;
      
      // If road segment is behind camera, move it to the front
      if (segment.z < cameraDistance - segmentLength / 2) {
        segment.z += roadLength;
        segment.mesh.position.z += roadLength;
      }
    }
    
    // Animate road stripes based on car speed
    for (let i = 0; i < roadStripes.length; i++) {
      const stripe = roadStripes[i];
      stripe.z -= moveDistance;
      stripe.mesh.position.z -= moveDistance;
      
      // If stripe is behind camera, move it to the front
      if (stripe.z < cameraDistance - 20) {
        stripe.z += roadLength;
        stripe.mesh.position.z += roadLength;
      }
    }
    
    // Move trees and buildings
    for (let i = 0; i < trees.length; i++) {
      const tree = trees[i];
      tree.trunk.position.z -= moveDistance;
      tree.foliage.position.z -= moveDistance;
      
      // Reset position if behind camera
      if (tree.trunk.position.z < cameraDistance - 20) {
        const resetDistance = roadLength + (cameraDistance - tree.trunk.position.z);
        tree.trunk.position.z += resetDistance;
        tree.foliage.position.z += resetDistance;
      }
    }
    
    for (let i = 0; i < buildings.length; i++) {
      const building = buildings[i];
      building.mesh.position.z -= moveDistance;
      
      // Reset position if behind camera
      if (building.mesh.position.z < cameraDistance - 20) {
        const resetDistance = roadLength + (cameraDistance - building.mesh.position.z);
        building.mesh.position.z += resetDistance;
      }
    }
  }
  
  function spawnObstacle() {
    // Random lane
    const obstacleLane = Math.floor(Math.random() * 3);
    
    // Random obstacle type
    const obstacleType = Math.random();
    let obstacle: THREE.Object3D;
    
    if (obstacleType < 0.7) {
      // Other car with random Tesla model design
      const carColors = [0x333333, 0x555555, 0xCCCCCC, 0x2C3E50];
      const carColor = carColors[Math.floor(Math.random() * carColors.length)];
      const car = new THREE.Group();
      
      // Car body
      const bodyGeometry = new THREE.BoxGeometry(2.5, 1.5, 5);
      const bodyMaterial = new THREE.MeshStandardMaterial({ color: carColor });
      const carBody = new THREE.Mesh(bodyGeometry, bodyMaterial);
      car.add(carBody);
      
      // Car top
      const topGeometry = new THREE.BoxGeometry(2.3, 1, 3);
      const topMaterial = new THREE.MeshStandardMaterial({ color: carColor });
      const carTop = new THREE.Mesh(topGeometry, topMaterial);
      carTop.position.y = 1;
      car.add(carTop);
      
      obstacle = car;
      obstacle.userData.isCar = true;
      obstacle.userData.overtaken = false;
    } else if (obstacleType < 0.85) {
      // Barrier
      const barrierGeometry = new THREE.BoxGeometry(3, 1, 1);
      const barrierMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
      obstacle = new THREE.Mesh(barrierGeometry, barrierMaterial);
      obstacle.userData.isBarrier = true;
    } else {
      // SpaceX rocket
      const rocket = new THREE.Group();
      
      // Rocket body
      const bodyGeometry = new THREE.CylinderGeometry(1, 1, 8, 16);
      const bodyMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff });
      const rocketBody = new THREE.Mesh(bodyGeometry, bodyMaterial);
      rocketBody.rotation.x = Math.PI / 2;
      rocket.add(rocketBody);
      
      // Rocket nose
      const noseGeometry = new THREE.ConeGeometry(1, 2, 16);
      const noseMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
      const rocketNose = new THREE.Mesh(noseGeometry, noseMaterial);
      rocketNose.rotation.x = Math.PI / 2;
      rocketNose.position.z = 5;
      rocket.add(rocketNose);
      
      // SpaceX logo
      const logoGeometry = new THREE.BoxGeometry(0.5, 2, 0.1);
      const logoMaterial = new THREE.MeshStandardMaterial({ color: 0x000000 });
      const logo = new THREE.Mesh(logoGeometry, logoMaterial);
      logo.rotation.x = Math.PI / 2;
      logo.position.y = 0.6;
      logo.position.z = 0;
      rocket.add(logo);
      
      obstacle = rocket;
      obstacle.userData.isRocket = true;
      obstacle.userData.overtaken = false;
    }
    
    // Position the obstacle
    obstacle.position.set(lanePositions[obstacleLane], 1, roadLength / 2);
    obstacle.castShadow = true;
    
    // Store obstacle size for collision detection
    obstacle.userData.width = 3;
    obstacle.userData.height = 5;
    
    scene.add(obstacle);
    obstacles.push(obstacle);
  }
  
  function checkCollisions() {
    // If jumping high enough, no collisions with ground obstacles
    if (isJumping && jumpHeight > 2) {
      return;
    }
    
    // Player bounding box
    const playerLeft = player.position.x - player.userData.width / 2;
    const playerRight = player.position.x + player.userData.width / 2;
    const playerFront = player.position.z + player.userData.height / 2;
    const playerBack = player.position.z - player.userData.height / 2;
    const playerBottom = player.position.y;
    const playerTop = player.position.y + 2; // Approximate height of car
    
    // Check collision with each obstacle
    for (let i = 0; i < obstacles.length; i++) {
      const obstacle = obstacles[i];
      
      const obstacleLeft = obstacle.position.x - obstacle.userData.width / 2;
      const obstacleRight = obstacle.position.x + obstacle.userData.width / 2;
      const obstacleFront = obstacle.position.z + obstacle.userData.height / 2;
      const obstacleBack = obstacle.position.z - obstacle.userData.height / 2;
      const obstacleBottom = obstacle.position.y;
      const obstacleTop = obstacle.position.y + 2; // Approximate height of obstacle
      
      // Check for overlap in all dimensions
      if (
        playerRight > obstacleLeft &&
        playerLeft < obstacleRight &&
        playerFront > obstacleBack &&
        playerBack < obstacleFront &&
        playerTop > obstacleBottom &&
        playerBottom < obstacleTop
      ) {
        // Collision detected
        handleCollision(obstacle);
        break;
      }
    }
  }
  
  function handleCollision(obstacle: THREE.Object3D) {
    // Remove the obstacle
    scene.remove(obstacle);
    obstacles.splice(obstacles.indexOf(obstacle), 1);
    
    // Reduce lives
    lives--;
    updateLifeDisplay();
    
    // Create explosion at collision point
    createExplosion(player.position.x, player.position.y, player.position.z);
    
    // Pause the game
    isPaused = true;
    
    // If no lives left, game over
    if (lives <= 0) {
      gameOver();
    } else {
      // Show message to press space to continue
      if (typeof document !== 'undefined') {
        const pauseMessage = document.createElement('div');
        pauseMessage.id = 'pause-message';
        pauseMessage.textContent = 'Press SPACE to continue';
        pauseMessage.style.position = 'absolute';
        pauseMessage.style.top = '50%';
        pauseMessage.style.left = '50%';
        pauseMessage.style.transform = 'translate(-50%, -50%)';
        pauseMessage.style.color = 'white';
        pauseMessage.style.fontSize = '24px';
        pauseMessage.style.fontWeight = 'bold';
        pauseMessage.style.textShadow = '2px 2px 4px #000';
        document.body.appendChild(pauseMessage);
        
        // Remove the message when the game is resumed
        const checkPauseState = setInterval(() => {
          if (!isPaused) {
            const msg = document.getElementById('pause-message');
            if (msg) document.body.removeChild(msg);
            clearInterval(checkPauseState);
          }
        }, 100);
      }
    }
  }
  
  function createExplosion(x: number, y: number, z: number) {
    // Create a group for the explosion
    const explosion = new THREE.Group();
    explosion.position.set(x, y, z);
    scene.add(explosion);
    
    // Create fire particles
    const particleCount = 50;
    const fireColors = [0xff0000, 0xff5500, 0xff8800, 0xffaa00]; // Red to orange
    
    for (let i = 0; i < particleCount; i++) {
      // Create a particle
      const size = Math.random() * 0.5 + 0.5;
      const geometry = new THREE.SphereGeometry(size, 8, 8);
      const material = new THREE.MeshBasicMaterial({ 
        color: fireColors[Math.floor(Math.random() * fireColors.length)],
        transparent: true,
        opacity: 0.8
      });
      
      const particle = new THREE.Mesh(geometry, material);
      
      // Random position within explosion radius
      const radius = Math.random() * 3;
      const theta = Math.random() * Math.PI * 2;
      const phi = Math.random() * Math.PI;
      
      particle.position.x = radius * Math.sin(phi) * Math.cos(theta);
      particle.position.y = radius * Math.sin(phi) * Math.sin(theta);
      particle.position.z = radius * Math.cos(phi);
      
      // Random velocity
      particle.userData.velocity = new THREE.Vector3(
        (Math.random() - 0.5) * 10,
        (Math.random() - 0.5) * 10,
        (Math.random() - 0.5) * 10
      );
      
      // Add to explosion group
      explosion.add(particle);
    }
    
    // Add a point light for the explosion glow
    const light = new THREE.PointLight(0xff5500, 5, 10);
    light.position.set(0, 0, 0);
    explosion.add(light);
    
    // Animate the explosion
    let explosionTime = 0;
    const explosionDuration = 2; // seconds
    
    const animateExplosion = () => {
      explosionTime += 0.016; // Approximately 60fps
      
      // Update particles
      explosion.children.forEach((child) => {
        if (child instanceof THREE.Mesh) {
          // Move particle based on velocity
          child.position.add(child.userData.velocity.clone().multiplyScalar(0.016));
          
          // Apply "gravity" to make particles fall
          child.userData.velocity.y -= 9.8 * 0.016;
          
          // Fade out particles
          if (child.material instanceof THREE.MeshBasicMaterial) {
            child.material.opacity = Math.max(0, 1 - (explosionTime / explosionDuration));
          }
        } else if (child instanceof THREE.PointLight) {
          // Fade out light
          child.intensity = Math.max(0, 5 * (1 - (explosionTime / (explosionDuration * 0.5))));
        }
      });
      
      // Continue animation until duration is reached
      if (explosionTime < explosionDuration) {
        requestAnimationFrame(animateExplosion);
      } else {
        // Remove explosion from scene
        scene.remove(explosion);
      }
    };
    
    // Start animation
    animateExplosion();
    
    // Create firework effects
    setTimeout(() => createFireworks(x, y, z), 200);
  }
  
  function createFireworks(x: number, y: number, z: number) {
    // Create firework particles
    const particleCount = 30;
    const fireworkColors = [0xff0000, 0x00ff00, 0x0000ff, 0xffff00, 0xff00ff, 0x00ffff];
    
    // Create a group for the firework
    const firework = new THREE.Group();
    firework.position.set(x, y + 5, z); // Start above the explosion
    scene.add(firework);
    
    for (let i = 0; i < particleCount; i++) {
      // Create a particle
      const geometry = new THREE.SphereGeometry(0.2, 8, 8);
      const material = new THREE.MeshBasicMaterial({ 
        color: fireworkColors[Math.floor(Math.random() * fireworkColors.length)],
        transparent: true,
        opacity: 1
      });
      
      const particle = new THREE.Mesh(geometry, material);
      
      // All particles start at center
      particle.position.set(0, 0, 0);
      
      // Random velocity in all directions
      const speed = Math.random() * 5 + 5;
      const angle = Math.random() * Math.PI * 2;
      const elevation = Math.random() * Math.PI - Math.PI/2;
      
      particle.userData.velocity = new THREE.Vector3(
        speed * Math.cos(angle) * Math.cos(elevation),
        speed * Math.sin(elevation),
        speed * Math.sin(angle) * Math.cos(elevation)
      );
      
      // Add to firework group
      firework.add(particle);
    }
    
    // Add a point light for the firework glow
    const light = new THREE.PointLight(fireworkColors[Math.floor(Math.random() * fireworkColors.length)], 2, 10);
    light.position.set(0, 0, 0);
    firework.add(light);
    
    // Animate the firework
    let fireworkTime = 0;
    const fireworkDuration = 1.5; // seconds
    
    const animateFirework = () => {
      fireworkTime += 0.016; // Approximately 60fps
      
      // Update particles
      firework.children.forEach((child) => {
        if (child instanceof THREE.Mesh) {
          // Move particle based on velocity
          child.position.add(child.userData.velocity.clone().multiplyScalar(0.016));
          
          // Apply gravity
          child.userData.velocity.y -= 9.8 * 0.016;
          
          // Fade out particles
          if (child.material instanceof THREE.MeshBasicMaterial) {
            child.material.opacity = Math.max(0, 1 - (fireworkTime / fireworkDuration));
          }
        } else if (child instanceof THREE.PointLight) {
          // Fade out light
          child.intensity = Math.max(0, 2 * (1 - (fireworkTime / (fireworkDuration * 0.5))));
        }
      });
      
      // Continue animation until duration is reached
      if (fireworkTime < fireworkDuration) {
        requestAnimationFrame(animateFirework);
      } else {
        // Remove firework from scene
        scene.remove(firework);
      }
    };
    
    // Start animation
    animateFirework();
  }
  
  function updateScore(deltaTime: number) {
    // Increase score based on speed and time
    score += deltaTime * speed / 10;
    
    // Check for overtaking cars
    for (let i = 0; i < obstacles.length; i++) {
      const obstacle = obstacles[i];
      if ((obstacle.userData.isCar || obstacle.userData.isRocket) && 
          !obstacle.userData.overtaken && 
          obstacle.position.z < player.position.z) {
        // Award points for overtaking
        score += 100;
        carsOvertaken++;
        obstacle.userData.overtaken = true;
        if (typeof document !== 'undefined') {
          const overtakenDisplay = document.getElementById('overtaken-display');
          if (overtakenDisplay) overtakenDisplay.textContent = `Cars Overtaken: ${carsOvertaken}`;
        }
      }
    }
    
    if (typeof document !== 'undefined') {
      const scoreDisplay = document.getElementById('score-display');
      if (scoreDisplay) scoreDisplay.textContent = `Score: ${Math.floor(score)}`;
    }
  }
  
  function startGame() {
    if (typeof document !== 'undefined') {
      const startScreen = document.getElementById('start-screen');
      if (startScreen) startScreen.style.display = 'none';
      
      const highScoreScreen = document.getElementById('high-score-screen');
      if (highScoreScreen) highScoreScreen.style.display = 'none';
    }
    gameActive = true;
    speed = 0;
    score = 0;
    carsOvertaken = 0;
    elapsedTime = 0;
    lastObstacleTime = 0;
    lives = 3; // Reset lives
    showHighScores = false;
    clock.start();
    
    // Update displays
    if (typeof document !== 'undefined') {
      const overtakenDisplay = document.getElementById('overtaken-display');
      if (overtakenDisplay) overtakenDisplay.textContent = 'Cars Overtaken: 0';
      
      const livesDisplay = document.getElementById('lives-display');
      if (livesDisplay) livesDisplay.textContent = `Lives: ${lives}`;
    }
    
    // Update life display
    createLifeDisplay();
  }
  
  function gameOver() {
    gameActive = false;
    
    if (typeof document !== 'undefined') {
      const finalScore = document.getElementById('final-score');
      if (finalScore) finalScore.textContent = Math.floor(score).toString();
      
      const finalOvertaken = document.getElementById('final-overtaken');
      if (finalOvertaken) finalOvertaken.textContent = carsOvertaken.toString();
      
      // Show name input for high score
      const nameInput = document.getElementById('player-name-input');
      if (nameInput) {
        (nameInput as HTMLInputElement).value = '';
      }
      
      const gameOverElement = document.getElementById('game-over');
      if (gameOverElement) gameOverElement.style.display = 'block';
    }
  }
  
  function submitHighScore() {
    if (typeof document !== 'undefined') {
      const nameInput = document.getElementById('player-name-input') as HTMLInputElement;
      if (nameInput && nameInput.value.trim() !== '') {
        playerName = nameInput.value.trim();
        saveHighScore(playerName, score);
        showHighScoreList();
      }
    }
  }
  
  function showHighScoreList() {
    showHighScores = true;
    
    if (typeof document !== 'undefined') {
      // Hide game over screen
      const gameOverElement = document.getElementById('game-over');
      if (gameOverElement) gameOverElement.style.display = 'none';
      
      // Show high score screen
      const highScoreScreen = document.getElementById('high-score-screen');
      if (highScoreScreen) {
        highScoreScreen.style.display = 'block';
        
        // Update high score list
        const highScoreList = document.getElementById('high-score-list');
        if (highScoreList) {
          highScoreList.innerHTML = '';
          
          if (highScores.length === 0) {
            highScoreList.innerHTML = '<li>No high scores yet!</li>';
          } else {
            highScores.forEach((entry, index) => {
              const listItem = document.createElement('li');
              listItem.textContent = `${index + 1}. ${entry.name}: ${entry.score}`;
              if (entry.name === playerName && entry.score === Math.floor(score)) {
                listItem.className = 'current-score';
              }
              highScoreList.appendChild(listItem);
            });
          }
        }
      }
    }
  }
  
  function restartGame() {
    // Clear obstacles
    for (let i = obstacles.length - 1; i >= 0; i--) {
      scene.remove(obstacles[i]);
    }
    obstacles = [];
    
    // Reset player position
    player.position.set(0, 0, 0);
    lane = 1;
    
    // Reset game state
    if (typeof document !== 'undefined') {
      const gameOverElement = document.getElementById('game-over');
      if (gameOverElement) gameOverElement.style.display = 'none';
      
      const highScoreScreen = document.getElementById('high-score-screen');
      if (highScoreScreen) highScoreScreen.style.display = 'none';
    }
    
    // Start game
    startGame();
  }
  
  function onKeyDown(event: KeyboardEvent) {
    // If game is over, don't process any keys
    if (!gameActive && !isPaused) return;
    
    // If game is paused and space is pressed, resume the game
    if (isPaused && event.key === ' ') {
      isPaused = false;
      return;
    }
    
    // Don't process movement keys if the game is paused
    if (isPaused) return;
    
    switch (event.key) {
      case 'ArrowRight':
        if (lane > 0) lane--;
        break;
      case 'ArrowLeft':
        if (lane < 2) lane++;
        break;
      case 'ArrowUp':
        speed = Math.min(maxSpeed, speed + acceleration * 5);
        break;
      case 'ArrowDown':
        speed = Math.max(0, speed - brakeForce * 5);
        break;
      case ' ':
        // Space bar for braking
        speed = Math.max(0, speed - brakeForce * 5);
        // Also allow space to jump
        if (!isJumping) {
          isJumping = true;
          jumpVelocity = jumpInitialVelocity;
          jumpHeight = 0; // Start from the ground
          jumpStartTime = elapsedTime;
        }
        break;
      case 'j':
      case 'J':
        // Jump functionality
        if (!isJumping) {
          isJumping = true;
          jumpVelocity = jumpInitialVelocity;
          jumpHeight = 0; // Start from the ground
          jumpStartTime = elapsedTime;
        }
        break;
      case 's':
      case 'S':
        smokeActive = true;
        smokeStartTime = elapsedTime;
        smokeDuration = Math.random() * 10; // Random duration up to 10 seconds
        break;
      case 'f':
      case 'F':
        if (!sunVisible) {
          createSun();
          sunVisible = true;
        } else {
          removeSun();
          sunVisible = false;
        }
        break;
      case 'l':
      case 'L':
        if (!lightningActive) {
          createLightningEffect();
          lightningActive = true;
          lightningTime = 0;
        }
        break;
    }
  }
  
  function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }
  
  function createRoad() {
    // Create a group for the road
    road = new THREE.Group();
    scene.add(road);
    
    // Create initial road segments
    for (let i = 0; i < segmentCount; i++) {
      createRoadSegment(i * segmentLength);
    }
    
    // Create road stripes separately for better animation
    createRoadStripes();
    
    // Add some environment elements
    createEnvironment();
  }
  
  function createRoadStripes() {
    // Clear any existing stripes
    roadStripes.forEach(stripe => road.remove(stripe.mesh));
    roadStripes = [];
    
    // Create new stripes at regular intervals
    const stripeSpacing = 20; // Distance between stripes
    const stripeLength = 10;  // Length of each stripe
    
    for (let z = 0; z < roadLength; z += stripeSpacing) {
      const stripeGeometry = new THREE.PlaneGeometry(0.5, stripeLength);
      const stripeMaterial = new THREE.MeshStandardMaterial({ color: stripColor });
      const stripe = new THREE.Mesh(stripeGeometry, stripeMaterial);
      stripe.rotation.x = -Math.PI / 2;
      stripe.position.set(0, 0.01, z);
      stripe.receiveShadow = true;
      road.add(stripe);
      roadStripes.push({ mesh: stripe, z: z });
    }
  }
  
  function createRoadSegment(z: number) {
    // Road
    const roadGeometry = new THREE.PlaneGeometry(15, segmentLength);
    const roadMaterial = new THREE.MeshStandardMaterial({ color: roadColor });
    const roadSegment = new THREE.Mesh(roadGeometry, roadMaterial);
    roadSegment.rotation.x = -Math.PI / 2;
    roadSegment.position.set(0, 0, z + segmentLength / 2);
    roadSegment.receiveShadow = true;
    road.add(roadSegment);
    roadSegments.push({ mesh: roadSegment, z: z });
    
    // We'll handle road stripes separately in a dedicated function
    // to allow for smoother animation based on car speed
    
    // Side grass (left)
    const grassLeftGeometry = new THREE.PlaneGeometry(100, segmentLength);
    const grassMaterial = new THREE.MeshStandardMaterial({ color: grassColor });
    const grassLeft = new THREE.Mesh(grassLeftGeometry, grassMaterial);
    grassLeft.rotation.x = -Math.PI / 2;
    grassLeft.position.set(-57.5, 0, z + segmentLength / 2);
    grassLeft.receiveShadow = true;
    road.add(grassLeft);
    
    // Side grass (right)
    const grassRightGeometry = new THREE.PlaneGeometry(100, segmentLength);
    const grassRight = new THREE.Mesh(grassRightGeometry, grassMaterial);
    grassRight.rotation.x = -Math.PI / 2;
    grassRight.position.set(57.5, 0, z + segmentLength / 2);
    grassRight.receiveShadow = true;
    road.add(grassRight);
  }
  
  function createEnvironment() {
    // Add trees and buildings along the road
    for (let z = 50; z < roadLength; z += Math.random() * 100 + 50) {
      // Left side trees
      if (Math.random() > 0.3) {
        createTree(-20 - Math.random() * 20, z);
      }
      
      // Right side trees
      if (Math.random() > 0.3) {
        createTree(20 + Math.random() * 20, z);
      }
      
      // Buildings occasionally
      if (Math.random() > 0.7) {
        const side = Math.random() > 0.5 ? 1 : -1;
        createBuilding(side * (30 + Math.random() * 20), z);
      }
    }
    
    // Add Tesla charging stations
    for (let z = 200; z < roadLength; z += 600) {
      createChargingStation(-25, z);
      createChargingStation(25, z + 300);
    }
    
    // Add SpaceX rockets in the background
    for (let z = 300; z < roadLength; z += 800) {
      createSpaceXRocket(-80 - Math.random() * 40, z + Math.random() * 200);
      createSpaceXRocket(80 + Math.random() * 40, z + Math.random() * 200);
    }
    
    // Add a skybox
    createSkybox();
  }
  
  function createTree(x: number, z: number) {
    const trunkGeometry = new THREE.CylinderGeometry(0.5, 0.7, 3, 8);
    const trunkMaterial = new THREE.MeshStandardMaterial({ color: 0x8B4513 });
    const trunk = new THREE.Mesh(trunkGeometry, trunkMaterial);
    trunk.position.set(x, 1.5, z);
    trunk.castShadow = true;
    scene.add(trunk);
    
    const foliageGeometry = new THREE.ConeGeometry(2, 4, 8);
    const foliageMaterial = new THREE.MeshStandardMaterial({ color: 0x228B22 });
    const foliage = new THREE.Mesh(foliageGeometry, foliageMaterial);
    foliage.position.set(x, 5, z);
    foliage.castShadow = true;
    scene.add(foliage);
    
    trees.push({ trunk, foliage, z });
  }
  
  function createBuilding(x: number, z: number) {
    const height = Math.random() * 20 + 10;
    const width = Math.random() * 10 + 5;
    const depth = Math.random() * 10 + 5;
    
    const buildingGeometry = new THREE.BoxGeometry(width, height, depth);
    const buildingMaterial = new THREE.MeshStandardMaterial({ 
      color: Math.random() > 0.5 ? 0xA9A9A9 : 0xD3D3D3 
    });
    const building = new THREE.Mesh(buildingGeometry, buildingMaterial);
    building.position.set(x, height / 2, z);
    building.castShadow = true;
    scene.add(building);
    
    buildings.push({ mesh: building, z });
  }
  
  function createChargingStation(x: number, z: number) {
    // Base platform
    const baseGeometry = new THREE.BoxGeometry(8, 0.5, 8);
    const baseMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc });
    const base = new THREE.Mesh(baseGeometry, baseMaterial);
    base.position.set(x, 0.25, z);
    scene.add(base);
    
    // Charging post
    const postGeometry = new THREE.BoxGeometry(1, 3, 1);
    const postMaterial = new THREE.MeshStandardMaterial({ color: 0xe82127 }); // Tesla red
    const post = new THREE.Mesh(postGeometry, postMaterial);
    post.position.set(x, 1.5, z);
    scene.add(post);
    
    // Tesla logo
    const logoGeometry = new THREE.BoxGeometry(0.5, 1, 0.1);
    const logoMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff });
    const logo = new THREE.Mesh(logoGeometry, logoMaterial);
    logo.position.set(x, 2, z - 0.51);
    scene.add(logo);
    
    trees.push({ trunk: base, foliage: post, z });
  }
  
  function createSpaceXRocket(x: number, z: number) {
    const rocketGroup = new THREE.Group();
    
    // Rocket body
    const bodyGeometry = new THREE.CylinderGeometry(2, 2, 30, 16);
    const bodyMaterial = new THREE.MeshStandardMaterial({ color: 0xffffff });
    const rocketBody = new THREE.Mesh(bodyGeometry, bodyMaterial);
    rocketBody.position.y = 15;
    rocketGroup.add(rocketBody);
    
    // Rocket nose
    const noseGeometry = new THREE.ConeGeometry(2, 5, 16);
    const noseMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 });
    const rocketNose = new THREE.Mesh(noseGeometry, noseMaterial);
    rocketNose.position.y = 32.5;
    rocketGroup.add(rocketNose);
    
    // Rocket fins
    const finGeometry = new THREE.BoxGeometry(1, 5, 3);
    const finMaterial = new THREE.MeshStandardMaterial({ color: 0x333333 });
    
    for (let i = 0; i < 4; i++) {
      const fin = new THREE.Mesh(finGeometry, finMaterial);
      fin.position.y = 2.5;
      fin.position.x = Math.sin(i * Math.PI / 2) * 3;
      fin.position.z = Math.cos(i * Math.PI / 2) * 3;
      fin.rotation.y = i * Math.PI / 2;
      rocketGroup.add(fin);
    }
    
    // SpaceX logo
    const logoGeometry = new THREE.BoxGeometry(0.5, 5, 0.1);
    const logoMaterial = new THREE.MeshStandardMaterial({ color: 0x000000 });
    const logo = new THREE.Mesh(logoGeometry, logoMaterial);
    logo.position.y = 15;
    logo.position.z = 2.1;
    rocketGroup.add(logo);
    
    // Launch pad
    const padGeometry = new THREE.BoxGeometry(10, 1, 10);
    const padMaterial = new THREE.MeshStandardMaterial({ color: 0x888888 });
    const pad = new THREE.Mesh(padGeometry, padMaterial);
    pad.position.y = -0.5;
    rocketGroup.add(pad);
    
    // Position the rocket
    rocketGroup.position.set(x, 0, z);
    scene.add(rocketGroup);
    
    // Add to buildings array for movement
    buildings.push({ mesh: rocketGroup, z });
  }
  
  function createSkybox() {
    // Simple skybox with gradient
    const verticalRadius = 400;
    const horizontalRadius = 600;
    const sphereGeometry = new THREE.SphereGeometry(1, 32, 32, 0, Math.PI * 2, 0, Math.PI / 2);
    const skyMaterial = new THREE.MeshBasicMaterial({
      color: 0x87CEEB,
      side: THREE.BackSide,
      fog: false
    });
    
    const sky = new THREE.Mesh(sphereGeometry, skyMaterial);
    sky.scale.set(horizontalRadius, verticalRadius, horizontalRadius);
    sky.position.y = 0;
    scene.add(sky);
  }
  
  function createSun() {
    // Create the sun sphere
    const sunGeometry = new THREE.SphereGeometry(20, 32, 32);
    const sunMaterial = new THREE.MeshBasicMaterial({
      color: 0xffdd00,
      emissive: 0xffdd00,
      emissiveIntensity: 1
    });
    
    sun = new THREE.Mesh(sunGeometry, sunMaterial);
    sun.position.set(200, 150, 300); // Position in the sky
    scene.add(sun);
    
    // Add a directional light to simulate sunlight
    sunLight = new THREE.DirectionalLight(0xffffcc, 1.5);
    sunLight.position.copy(sun.position);
    sunLight.castShadow = true;
    sunLight.shadow.mapSize.width = 2048;
    sunLight.shadow.mapSize.height = 2048;
    scene.add(sunLight);
    
    // Add lens flare effect (simplified with a small glow sphere)
    const glowGeometry = new THREE.SphereGeometry(25, 32, 32);
    const glowMaterial = new THREE.MeshBasicMaterial({
      color: 0xffffaa,
      transparent: true,
      opacity: 0.3
    });
    
    const glow = new THREE.Mesh(glowGeometry, glowMaterial);
    glow.position.copy(sun.position);
    scene.add(glow);
  }
  
  function removeSun() {
    // Remove sun and its light
    if (sun) {
      scene.remove(sun);
      sun = null;
    }
    
    if (sunLight) {
      scene.remove(sunLight);
      sunLight = null;
    }
  }
  
  function createSmokeParticle() {
    // Create a smoke particle at the car's exhaust position
    const size = Math.random() * 0.5 + 0.5;
    const smokeGeometry = new THREE.SphereGeometry(size, 8, 8);
    const smokeMaterial = new THREE.MeshBasicMaterial({
      color: 0xdddddd,
      transparent: true,
      opacity: 0.7
    });
    
    const smoke = new THREE.Mesh(smokeGeometry, smokeMaterial);
    
    // Position behind the car
    smoke.position.set(
      player.position.x + (Math.random() * 2 - 1), // Random spread
      player.position.y + 0.5 + (Math.random() * 0.5), // Slightly above exhaust
      player.position.z - 3 // Behind the car
    );
    
    // Add to scene and particles array
    scene.add(smoke);
    smokeParticles.push(smoke);
    
    // Limit the number of particles to prevent performance issues
    if (smokeParticles.length > 50) {
      const oldestParticle = smokeParticles.shift();
      if (oldestParticle) scene.remove(oldestParticle);
    }
  }
  
  function updateSmokeParticles(deltaTime: number) {
    // Update each smoke particle
    for (let i = smokeParticles.length - 1; i >= 0; i--) {
      const particle = smokeParticles[i];
      
      // Move particle
      particle.position.y += deltaTime * 2; // Rise up
      particle.position.x += (Math.random() - 0.5) * deltaTime; // Random drift
      particle.scale.multiplyScalar(1 + deltaTime * 0.5); // Grow larger
      
      // Fade out
      if (particle.material instanceof THREE.MeshBasicMaterial) {
        particle.material.opacity -= deltaTime * 0.8;
        
        // Remove if fully transparent
        if (particle.material.opacity <= 0) {
          scene.remove(particle);
          smokeParticles.splice(i, 1);
        }
      }
    }
    
    // Check if smoke effect has been active for longer than the random duration
    if (smokeActive && elapsedTime - smokeStartTime > smokeDuration) {
      smokeActive = false;
    }
  }
  
  function createLightningEffect() {
    // Create lightning bolt
    lightningBolt = new THREE.Group();
    
    // Create a zigzag pattern for the lightning
    const points = [];
    const segments = 8;
    const height = 100;
    
    // Start from above the scene
    points.push(new THREE.Vector3(player.position.x + (Math.random() * 40 - 20), height, player.position.z + (Math.random() * 40 - 20)));
    
    // Create zigzag pattern down to ground
    for (let i = 1; i < segments; i++) {
      const t = i / segments;
      const x = points[0].x + (Math.random() * 10 - 5) * (1 - t);
      const y = height * (1 - t);
      const z = points[0].z + (Math.random() * 10 - 5) * (1 - t);
      points.push(new THREE.Vector3(x, y, z));
    }
    
    // End at ground level
    points.push(new THREE.Vector3(points[points.length - 1].x, 0, points[points.length - 1].z));
    
    // Create lightning segments
    for (let i = 0; i < points.length - 1; i++) {
      const start = points[i];
      const end = points[i + 1];
      
      // Calculate direction and length
      const direction = new THREE.Vector3().subVectors(end, start);
      const length = direction.length();
      
      // Create cylinder for this segment
      const geometry = new THREE.CylinderGeometry(0.2, 0.2, length, 6);
      const material = new THREE.MeshBasicMaterial({ 
        color: 0x80f0ff,
        emissive: 0x80f0ff,
        emissiveIntensity: 1
      });
      
      const segment = new THREE.Mesh(geometry, material);
      
      // Position and rotate to connect points
      segment.position.copy(start);
      segment.position.add(direction.multiplyScalar(0.5));
      segment.lookAt(end);
      segment.rotateX(Math.PI / 2);
      
      lightningBolt.add(segment);
    }
    
    scene.add(lightningBolt);
    
    // Create flash light
    lightningFlash = new THREE.DirectionalLight(0xffffff, 2);
    lightningFlash.position.set(0, 50, 0);
    scene.add(lightningFlash);
  }
  
  function updateLightningEffect(deltaTime: number) {
    lightningTime += deltaTime;
    
    // Flash intensity based on time
    if (lightningFlash) {
      const flashIntensity = Math.max(0, 2 * (1 - lightningTime / lightningDuration));
      lightningFlash.intensity = flashIntensity * (Math.random() * 0.5 + 0.5); // Flicker effect
    }
    
    // Remove lightning after duration
    if (lightningTime >= lightningDuration) {
      if (lightningBolt) {
        scene.remove(lightningBolt);
        lightningBolt = null;
      }
      
      if (lightningFlash) {
        scene.remove(lightningFlash);
        lightningFlash = null;
      }
      
      lightningActive = false;
    }
  }
  
  function createPlayerCar() {
    // Tesla car model
    const car = new THREE.Group();
    
    // Main body
    const bodyGeometry = new THREE.BoxGeometry(3, 1, 6);
    const bodyMaterial = new THREE.MeshStandardMaterial({ color: 0xe82127 }); // Tesla red
    const carBody = new THREE.Mesh(bodyGeometry, bodyMaterial);
    carBody.position.y = 1;
    carBody.castShadow = true;
    car.add(carBody);
    
    // Top/cabin
    const topGeometry = new THREE.BoxGeometry(2.8, 1, 3);
    const glassMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x111111,
      metalness: 0.9,
      roughness: 0.1
    });
    const carTop = new THREE.Mesh(topGeometry, glassMaterial);
    carTop.position.set(0, 1.8, -0.5);
    carTop.castShadow = true;
    car.add(carTop);
    
    // Wheels
    const wheelGeometry = new THREE.CylinderGeometry(0.7, 0.7, 0.5, 16);
    const wheelMaterial = new THREE.MeshStandardMaterial({ color: 0x333333 });
    
    const wheelFL = new THREE.Mesh(wheelGeometry, wheelMaterial);
    wheelFL.rotation.z = Math.PI / 2;
    wheelFL.position.set(-1.5, 0.7, 1.5);
    car.add(wheelFL);
    
    const wheelFR = new THREE.Mesh(wheelGeometry, wheelMaterial);
    wheelFR.rotation.z = Math.PI / 2;
    wheelFR.position.set(1.5, 0.7, 1.5);
    car.add(wheelFR);
    
    const wheelBL = new THREE.Mesh(wheelGeometry, wheelMaterial);
    wheelBL.rotation.z = Math.PI / 2;
    wheelBL.position.set(-1.5, 0.7, -1.5);
    car.add(wheelBL);
    
    const wheelBR = new THREE.Mesh(wheelGeometry, wheelMaterial);
    wheelBR.rotation.z = Math.PI / 2;
    wheelBR.position.set(1.5, 0.7, -1.5);
    car.add(wheelBR);
    
    // Headlights
    const headlightGeometry = new THREE.SphereGeometry(0.3, 16, 16);
    const headlightMaterial = new THREE.MeshStandardMaterial({ 
      color: 0xffffcc,
      emissive: 0xffffcc,
      emissiveIntensity: 0.5
    });
    
    const headlightL = new THREE.Mesh(headlightGeometry, headlightMaterial);
    headlightL.position.set(-1, 1, 3);
    car.add(headlightL);
    
    const headlightR = new THREE.Mesh(headlightGeometry, headlightMaterial);
    headlightR.position.set(1, 1, 3);
    car.add(headlightR);
    
    // Position the car
    car.position.set(0, 0, 0);
    scene.add(car);
    
    player = car;
    player.userData.width = 3; // For collision detection
    player.userData.height = 6; // For collision detection
  }
</script>

<div class="game-container">
  <div bind:this={gameContainer} class="canvas-container"></div>
  
  <div id="score-display">Score: 0</div>
  <div id="overtaken-display">Cars Overtaken: 0</div>
  <div id="speed-display">Speed: 0 km/h</div>
  <div id="lives-display">Lives: 3</div>
  
  <div id="game-over">
    <h2>Game Over</h2>
    <p>Final Score: <span id="final-score">0</span></p>
    <p>Cars Overtaken: <span id="final-overtaken">0</span></p>
    <div class="name-input-container">
      <p>Enter your name for the high score:</p>
      <input type="text" id="player-name-input" maxlength="15" placeholder="Your name">
      <button id="submit-score-button" on:click={submitHighScore}>Submit Score</button>
    </div>
    <button id="restart-button" on:click={restartGame}>Restart</button>
  </div>
  
  <div id="high-score-screen">
    <h2>High Scores</h2>
    <ul id="high-score-list">
      <!-- High scores will be populated here -->
    </ul>
    <button id="play-again-button" on:click={restartGame}>Play Again</button>
  </div>
  
  <div id="start-screen">
    <h1>TESLA OUTRUN</h1>
    <p>Drive your Tesla through the futuristic highway</p>
    <p>You have 3 lives - try to get the highest score!</p>
    <p>Controls: Left/Right arrows to steer, Up arrow to accelerate, Down arrow to brake</p>
    <p>Press S for smoke effects (lasts randomly up to 10 seconds)</p>
    <p>Press F to toggle the sun and L for lightning strikes!</p>
    <button id="start-button" on:click={startGame}>START GAME</button>
  </div>
</div>

<style>
  .game-container {
    position: relative;
    width: 100vw;
    height: 100vh;
  }
  
  .canvas-container {
    width: 100%;
    height: 100%;
  }
  
  #score-display {
    position: absolute;
    top: 20px;
    right: 20px;
    color: white;
    font-size: 24px;
    z-index: 10;
  }
  
  #overtaken-display {
    position: absolute;
    top: 60px;
    right: 20px;
    color: white;
    font-size: 24px;
    z-index: 10;
  }
  
  #speed-display {
    position: absolute;
    bottom: 20px;
    left: 20px;
    color: white;
    font-size: 24px;
    z-index: 10;
  }
  
  #lives-display {
    position: absolute;
    top: 20px;
    left: 20px;
    color: white;
    font-size: 24px;
    z-index: 10;
  }
  
  #game-over {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: white;
    font-size: 48px;
    text-align: center;
    display: none;
    z-index: 10;
    background-color: rgba(0, 0, 0, 0.8);
    padding: 30px;
    border-radius: 10px;
  }
  
  #high-score-screen {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: white;
    font-size: 24px;
    text-align: center;
    display: none;
    z-index: 10;
    background-color: rgba(0, 0, 0, 0.8);
    padding: 30px;
    border-radius: 10px;
    min-width: 300px;
  }
  
  #high-score-list {
    list-style-type: none;
    padding: 0;
    margin: 20px 0;
    text-align: left;
  }
  
  #high-score-list li {
    padding: 10px;
    border-bottom: 1px solid #444;
    font-size: 20px;
  }
  
  #high-score-list li.current-score {
    color: #e82127; /* Tesla red */
    font-weight: bold;
  }
  
  .name-input-container {
    margin: 20px 0;
  }
  
  #player-name-input {
    padding: 10px;
    font-size: 18px;
    margin: 10px 0;
    width: 80%;
    border: none;
    border-radius: 5px;
  }
  
  #start-screen {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.8);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    color: white;
    z-index: 20;
  }
  
  #start-screen h1 {
    font-size: 48px;
    margin-bottom: 20px;
    color: #e82127; /* Tesla red */
  }
  
  #start-screen p {
    font-size: 24px;
    margin-bottom: 10px;
  }
  
  #start-button, #restart-button, #submit-score-button, #play-again-button {
    padding: 15px 30px;
    font-size: 20px;
    background-color: #e82127; /* Tesla red */
    color: white;
    border: none;
    cursor: pointer;
    border-radius: 5px;
    margin: 10px 0;
  }
  
  #start-button:hover, #restart-button:hover, #submit-score-button:hover, #play-again-button:hover {
    background-color: #b81a1f;
  }
</style>