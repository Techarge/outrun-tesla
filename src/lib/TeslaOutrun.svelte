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
  
  // Jump variables
  let isJumping = false;
  let jumpHeight = 0;
  let jumpVelocity = 0;
  let jumpGravity = 0.2;
  let maxJumpHeight = 5;
  let jumpInitialVelocity = 1;
  
  // Smoke particles
  let smokeParticles: THREE.Mesh[] = [];
  let lastSmokeTime = 0;
  let smokeInterval = 0.05; // Seconds between smoke particles
  
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
    
    // Initialize clock
    clock = new THREE.Clock();
    
    // Start animation loop
    animate();
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
    
    // Move the car towards target lane
    const targetX = lanePositions[lane];
    player.position.x += (targetX - player.position.x) * 0.1;
    
    // Check collisions
    checkCollisions();
    
    // Update score
    updateScore(deltaTime);
    
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
    // Player bounding box
    const playerLeft = player.position.x - player.userData.width / 2;
    const playerRight = player.position.x + player.userData.width / 2;
    const playerFront = player.position.z + player.userData.height / 2;
    const playerBack = player.position.z - player.userData.height / 2;
    
    // Check collision with each obstacle
    for (let i = 0; i < obstacles.length; i++) {
      const obstacle = obstacles[i];
      
      const obstacleLeft = obstacle.position.x - obstacle.userData.width / 2;
      const obstacleRight = obstacle.position.x + obstacle.userData.width / 2;
      const obstacleFront = obstacle.position.z + obstacle.userData.height / 2;
      const obstacleBack = obstacle.position.z - obstacle.userData.height / 2;
      
      // Check for overlap
      if (
        playerRight > obstacleLeft &&
        playerLeft < obstacleRight &&
        playerFront > obstacleBack &&
        playerBack < obstacleFront
      ) {
        // Collision detected
        gameOver();
        break;
      }
    }
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
    }
    gameActive = true;
    speed = 0;
    score = 0;
    carsOvertaken = 0;
    elapsedTime = 0;
    lastObstacleTime = 0;
    clock.start();
    if (typeof document !== 'undefined') {
      const overtakenDisplay = document.getElementById('overtaken-display');
      if (overtakenDisplay) overtakenDisplay.textContent = 'Cars Overtaken: 0';
    }
  }
  
  function gameOver() {
    gameActive = false;
    if (typeof document !== 'undefined') {
      const finalScore = document.getElementById('final-score');
      if (finalScore) finalScore.textContent = Math.floor(score).toString();
      
      const finalOvertaken = document.getElementById('final-overtaken');
      if (finalOvertaken) finalOvertaken.textContent = carsOvertaken.toString();
      
      const gameOverElement = document.getElementById('game-over');
      if (gameOverElement) gameOverElement.style.display = 'block';
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
    }
    
    // Start game
    startGame();
  }
  
  function onKeyDown(event: KeyboardEvent) {
    if (!gameActive) return;
    
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
      case ' ':
        speed = Math.max(0, speed - brakeForce * 5);
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
    
    // Add some environment elements
    createEnvironment();
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
    
    // Road stripes
    if (z % 40 < 20) {  // Only add stripes to every other segment
      const stripeGeometry = new THREE.PlaneGeometry(0.5, 10);
      const stripeMaterial = new THREE.MeshStandardMaterial({ color: stripColor });
      const stripe = new THREE.Mesh(stripeGeometry, stripeMaterial);
      stripe.rotation.x = -Math.PI / 2;
      stripe.position.set(0, 0.01, z + segmentLength / 2);
      stripe.receiveShadow = true;
      road.add(stripe);
    }
    
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
  
  <div id="game-over">
    <h2>Game Over</h2>
    <p>Final Score: <span id="final-score">0</span></p>
    <p>Cars Overtaken: <span id="final-overtaken">0</span></p>
    <button id="restart-button" on:click={restartGame}>Restart</button>
  </div>
  
  <div id="start-screen">
    <h1>TESLA OUTRUN</h1>
    <p>Drive your Tesla through the futuristic highway</p>
    <p>Controls: Left/Right arrows to steer, Up arrow to accelerate, Down arrow to brake</p>
    <p>Press Space to jump and leave a smoke trail!</p>
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
    margin-bottom: 40px;
  }
  
  #start-button, #restart-button {
    padding: 15px 30px;
    font-size: 20px;
    background-color: #e82127; /* Tesla red */
    color: white;
    border: none;
    cursor: pointer;
    border-radius: 5px;
  }
  
  #start-button:hover, #restart-button:hover {
    background-color: #b81a1f;
  }
</style>