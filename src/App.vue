<template>
  <div ref="mountRef" class="canvas-container">
    <div class="ui">
      <p>
        Level: {{ level }} | Damage: {{ playerStats.baseDamage }}
      </p>
      <div class="log">
        <ul>
          <li v-for="message in logMessages.slice(0, 4)" :key="message">
            {{ message }}
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref, reactive } from "vue";
import * as THREE from "three";

// References and reactive state
const mountRef = ref(null);
const globalScene = ref(null);
const playerPosition = reactive({ x: 0, y: 1, z: 0 });

// Experience and Level
const experience = ref(0);
const level = ref(1);

// Log Messages
const logMessages = ref([]);

// Control variables
const keys = reactive({
  w: false,
  a: false,
  s: false,
  d: false,
  ArrowLeft: false,
  ArrowRight: false,
  ArrowUp: false,
  ArrowDown: false,
});

// Jump variables
let jump = false;
let jumpHeight = 10;

// Player Stats
const playerStats = reactive({
  baseDamage: 5, // Base damage per hit
  health: 1000,
  maxHealth: 1000,
  name: "Player",
});

// Camera control variables
const cameraAngleAzimuth = ref(0); // Horizontal angle in radians
const cameraAnglePolar = ref(Math.PI / 4); // Vertical angle in radians
const cameraDistance = ref(40);
const minDistance = 5;
const maxDistance = 100;

// Mouse control variables
let isMouseDown = false;
let previousMousePosition = {
  x: 0,
  y: 0,
};

// Monster Types
const monsterTypes = [
  { color: 0x00ff00, health: 100, name: "Green" },
  { color: 0x0000ff, health: 200, name: "Blue" },
  { color: 0xffff00, health: 300, name: "Yellow" },
  { color: 0xff00ff, health: 400, name: "Magenta" },
  { color: 0xff0000, health: 500, name: "Red" },
];

// Function to create a simple tree with varied sizes
const createTree = () => {
  const tree = new THREE.Group();

  // Trunk
  const trunkGeometry = new THREE.CylinderGeometry(0.2, 0.2, 2, 8);
  const trunkMaterial = new THREE.MeshStandardMaterial({ color: 0x8b4513 });
  const trunk = new THREE.Mesh(trunkGeometry, trunkMaterial);
  trunk.position.y = 1; // Half of trunk height
  tree.add(trunk);

  // Leaves with varying sizes
  const leavesGeometry = new THREE.ConeGeometry(
    1 + Math.random() * 0.5,
    3 + Math.random() * 1,
    8
  );
  const leavesMaterial = new THREE.MeshStandardMaterial({ color: 0x228b22 });
  const leaves = new THREE.Mesh(leavesGeometry, leavesMaterial);
  leaves.position.y = 3; // Top of the trunk
  tree.add(leaves);

  return tree;
};

// Function to add forest boundaries with increased trees and varied sizes
const addForestBoundaries = (scene) => {
  const boundaryDistance = 90; // Matches the ground size
  const treeSpacing = 25; // Decreased spacing for more trees
  const treeVariation = 0.5; // Variation factor for tree size

  // Place trees along the perimeter with some randomness
  for (let x = -boundaryDistance; x <= boundaryDistance; x += treeSpacing) {
    for (let z = -boundaryDistance; z <= boundaryDistance; z += treeSpacing) {
      // Add some randomness to positions
      const offsetX = (Math.random() - 0.5) * treeSpacing * 0.5;
      const offsetZ = (Math.random() - 0.5) * treeSpacing * 0.5;

      const tree = createTree();
      tree.position.set(x + offsetX, 0, z + offsetZ);

      // Randomly scale trees for variety
      const scale = 3 + Math.random() * treeVariation;
      tree.scale.set(scale, scale, scale);

      scene.add(tree);
    }
  }
};

// Function to create a health bar sprite with monster or player name
const createHealthBar = (name, initialHealth, maxHealth) => {
  const canvas = document.createElement("canvas");
  canvas.width = 128;
  canvas.height = 32; // Increased height to accommodate text and health bar
  const context = canvas.getContext("2d");

  // Draw name
  context.font = "bold 14px Arial";
  context.fillStyle = "#ffffff";
  context.textAlign = "center";
  context.fillText(name, canvas.width / 2, 14);

  // Draw health bar background
  context.fillStyle = "#555555";
  context.fillRect(10, 18, canvas.width - 20, 10);

  // Draw initial fill (full health)
  context.fillStyle = "#ff0000";
  context.fillRect(10, 18, canvas.width - 20, 10);

  // Draw numerical health
  context.font = "bold 10px Arial";
  context.fillStyle = "#ffffff";
  context.textAlign = "center";
  context.fillText(`${initialHealth} / ${maxHealth}`, canvas.width / 2, 27);

  const texture = new THREE.CanvasTexture(canvas);
  const spriteMaterial = new THREE.SpriteMaterial({ map: texture });
  const sprite = new THREE.Sprite(spriteMaterial);
  sprite.scale.set(8, 2, 1); // Adjust size as needed

  return { sprite, texture, context, canvas, maxHealth };
};

// Modify the Monster class to include health bar with name
class Monster {
  constructor(mesh, type, scene) {
    this.mesh = mesh;
    this.health = type.health;
    this.type = type;

    // Create health bar with name
    const { sprite, texture, context, canvas, maxHealth } = createHealthBar(
      this.type.name,
      this.health,
      this.type.health
    );
    sprite.position.set(0, 3, 0); // Position above the monster
    this.healthBar = sprite;
    this.healthBarTexture = texture;
    this.healthBarContext = context;
    this.healthBarCanvas = canvas;
    this.maxHealth = maxHealth;
    this.mesh.add(this.healthBar);
  }

  takeDamage(damage) {
    this.health -= damage;
    this.updateHealthBar();
    if (this.health <= 0) {
      // Remove from scene
      globalScene.value.remove(this.mesh);
      // Remove from monsters array
      const index = monsters.indexOf(this);
      if (index > -1) {
        monsters.splice(index, 1);
      }
      // Log the kill and gain experience
      experience.value += 20;
      logMessages.value.unshift(`Monster killed! Gained 20 XP.`);
      // Check for level up
      checkLevelUp();
      // Spawn a new monster
      spawnMonster();
    }
  }

  updateHealthBar() {
    const healthPercentage = Math.max(this.health / this.maxHealth, 0);
    const barWidth = this.healthBarCanvas.width - 20;
    const filledWidth = barWidth * healthPercentage;

    // Clear the health bar area (excluding the name)
    this.healthBarContext.clearRect(0, 18, this.healthBarCanvas.width, 14);

    // Redraw health bar background
    this.healthBarContext.fillStyle = "#555555";
    this.healthBarContext.fillRect(10, 18, barWidth, 10);

    // Redraw filled health
    this.healthBarContext.fillStyle = "#ff0000";
    this.healthBarContext.fillRect(10, 18, filledWidth, 10);

    // Redraw numerical health
    this.healthBarContext.font = "bold 10px Arial";
    this.healthBarContext.fillStyle = "#ffffff";
    this.healthBarContext.textAlign = "center";
    this.healthBarContext.fillText(
      `${Math.max(this.health, 0)} / ${this.maxHealth}`,
      this.healthBarCanvas.width / 2,
      27
    );

    // Update texture
    this.healthBarTexture.needsUpdate = true;
  }
}

// Array to hold monsters
const monsters = [];

// Function to spawn a single monster at a random position with random type
const spawnMonster = () => {
  const type = monsterTypes[Math.floor(Math.random() * monsterTypes.length)];
  const monsterGeometry = new THREE.BoxGeometry(3, 3, 3); // Adjust size as needed
  const monsterMaterial = new THREE.MeshStandardMaterial({
    color: type.color,
  });
  const monster = new THREE.Mesh(monsterGeometry, monsterMaterial);
  monster.position.set(
    Math.random() * 180 - 90, // Keep within boundaryDistance - 10 to prevent spawning near the boundary
    1.5, // Half of height to sit on ground
    Math.random() * 180 - 90
  );
  globalScene.value.add(monster);
  monsters.push(new Monster(monster, type, globalScene.value));
};

// Function to spawn multiple monsters initially
const spawnInitialMonsters = (count = 50) => {
  for (let i = 0; i < count; i++) {
    spawnMonster();
  }
};

// Function to handle player jump
const handleJump = () => {
  if (!jump) {
    jump = true;
  }
};

// Handle key down events
const handleKeyDown = (event) => {
  const { key } = event;
  if (keys.hasOwnProperty(key)) {
    keys[key] = true;
  }
  if (key === " ") {
    // Spacebar to jump
    handleJump();
  }
};

// Handle key up events
const handleKeyUp = (event) => {
  const { key } = event;
  if (keys.hasOwnProperty(key)) {
    keys[key] = false;
  }
};

// Function to reset all keys (e.g., on window blur)
const resetKeys = () => {
  for (const key in keys) {
    if (keys.hasOwnProperty(key)) {
      keys[key] = false;
    }
  }
};

// Function to check and handle level up
const checkLevelUp = () => {
  const xpThreshold = level.value * 100; // Example threshold
  if (experience.value >= xpThreshold) {
    experience.value -= xpThreshold;
    level.value += 1;
    logMessages.value.unshift(`Leveled up to Level ${level.value}!`);
    growPlayer();
    increaseDamage();
  }
};

// Function to grow the player character
const growPlayer = () => {
  playerGroup.scale.set(level.value, level.value, level.value);
};

// Function to increase player damage based on level
const increaseDamage = () => {
  playerStats.baseDamage += 0.5; // Example increment
};

// Player group (declared here to access in functions)
let playerGroup;
let playerHealthBar;
let playerHealthBarTexture;
let playerHealthBarContext;
let playerHealthBarCanvas;
let playerMaxHealth;

// Function to create player with body, head, arms, and health bar
const createPlayer = (scene) => {
  playerGroup = new THREE.Group();

  // Body
  const bodyGeometry = new THREE.BoxGeometry(1, 2, 0.5);
  const bodyMaterial = new THREE.MeshStandardMaterial({ color: 0x008000 }); // Green
  const body = new THREE.Mesh(bodyGeometry, bodyMaterial);
  body.position.y = 1; // Adjust position
  playerGroup.add(body);

  // Head
  const headGeometry = new THREE.SphereGeometry(0.5, 32, 32);
  const headMaterial = new THREE.MeshStandardMaterial({ color: 0xffe0bd }); // Skin color
  const head = new THREE.Mesh(headGeometry, headMaterial);
  head.position.y = 2.3; // On top of the body, raised higher
  playerGroup.add(head);

  // Left Arm
  const leftArmGeometry = new THREE.BoxGeometry(0.3, 1, 0.3);
  const leftArmMaterial = new THREE.MeshStandardMaterial({ color: 0xffe0bd }); // Skin color
  const leftArm = new THREE.Mesh(leftArmGeometry, leftArmMaterial);
  leftArm.position.set(-0.65, 1.3, 0); // Left side of the body
  playerGroup.add(leftArm);

  // Right Arm
  const rightArmGeometry = new THREE.BoxGeometry(0.3, 1, 0.3);
  const rightArmMaterial = new THREE.MeshStandardMaterial({ color: 0xffe0bd }); // Skin color
  const rightArm = new THREE.Mesh(rightArmGeometry, rightArmMaterial);
  rightArm.position.set(0.65, 1.3, 0); // Right side of the body
  playerGroup.add(rightArm);

  // Create health bar with name
  const { sprite, texture, context, canvas, maxHealth } = createHealthBar(
    playerStats.name,
    playerStats.health,
    playerStats.maxHealth
  );
  sprite.position.set(0, 4, 0); // Position above the head
  playerGroup.add(sprite);
  playerHealthBar = sprite;
  playerHealthBarTexture = texture;
  playerHealthBarContext = context;
  playerHealthBarCanvas = canvas;
  playerMaxHealth = maxHealth;

  // Set initial position
  playerGroup.position.set(0, 1, 0);
  scene.add(playerGroup);
};

// Function to update the player's health bar
const updatePlayerHealthBar = () => {
  const healthPercentage = Math.max(playerStats.health / playerStats.maxHealth, 0);
  const barWidth = playerHealthBarCanvas.width - 20;
  const filledWidth = barWidth * healthPercentage;

  // Clear the health bar area (excluding the name)
  playerHealthBarContext.clearRect(0, 18, playerHealthBarCanvas.width, 14);

  // Redraw health bar background
  playerHealthBarContext.fillStyle = "#555555";
  playerHealthBarContext.fillRect(10, 18, barWidth, 10);

  // Redraw filled health
  playerHealthBarContext.fillStyle = "#00ff00"; // Use green for player's health
  playerHealthBarContext.fillRect(10, 18, filledWidth, 10);

  // Redraw numerical health
  playerHealthBarContext.font = "bold 10px Arial";
  playerHealthBarContext.fillStyle = "#ffffff";
  playerHealthBarContext.textAlign = "center";
  playerHealthBarContext.fillText(
    `${Math.max(playerStats.health, 0)} / ${playerStats.maxHealth}`,
    playerHealthBarCanvas.width / 2,
    27
  );

  // Update texture
  playerHealthBarTexture.needsUpdate = true;
};

// Function to create sky with enhanced gradient and clouds
const createSky = (scene) => {
  // Create a large sphere to act as the sky
  const skyGeometry = new THREE.SphereGeometry(500, 32, 32);
  const skyMaterial = new THREE.ShaderMaterial({
    uniforms: {},
    vertexShader: `
      varying vec3 vWorldPosition;
      void main() {
        vec4 worldPosition = modelMatrix * vec4(position, 1.0);
        vWorldPosition = worldPosition.xyz;
        gl_Position = projectionMatrix * viewMatrix * worldPosition;
      }
    `,
    fragmentShader: `
      varying vec3 vWorldPosition;
      void main() {
        float h = normalize(vWorldPosition).y;
        gl_FragColor = vec4(
          mix(vec3(0.0, 0.0, 0.5), vec3(0.5, 0.7, 1.0), pow(max(h, 0.0), 0.5)),
          1.0
        );
      }
    `,
    side: THREE.BackSide,
  });
  const sky = new THREE.Mesh(skyGeometry, skyMaterial);
  scene.add(sky);

  // Add clouds
  const cloudTexture = new THREE.TextureLoader().load(
    "https://threejsfundamentals.org/threejs/resources/images/cloud.png"
  );
  const cloudMaterial = new THREE.SpriteMaterial({
    map: cloudTexture,
    transparent: true,
    opacity: 0.8,
  });

  for (let i = 0; i < 50; i++) {
    const cloud = new THREE.Sprite(cloudMaterial);
    cloud.position.set(
      (Math.random() - 0.5) * 400,
      Math.random() * 50 + 50,
      (Math.random() - 0.5) * 400
    );
    const scale = Math.random() * 20 + 20;
    cloud.scale.set(scale, scale, scale);
    scene.add(cloud);
  }
};

// Handle mouse wheel for zoom
const handleWheel = (event) => {
  event.preventDefault();
  const delta = event.deltaY * 0.05;
  cameraDistance.value += delta;
  cameraDistance.value = Math.max(
    minDistance,
    Math.min(maxDistance, cameraDistance.value)
  );
};

// Mouse event handlers for camera control
const handleMouseDown = (event) => {
  isMouseDown = true;
  previousMousePosition = {
    x: event.clientX,
    y: event.clientY,
  };
};

const handleMouseMove = (event) => {
  if (isMouseDown) {
    const deltaMove = {
      x: event.clientX - previousMousePosition.x,
      y: event.clientY - previousMousePosition.y,
    };

    const azimuthStep = deltaMove.x * 0.005;
    const polarStep = deltaMove.y * 0.005;

    cameraAngleAzimuth.value -= azimuthStep;
    cameraAnglePolar.value -= polarStep;

    // Clamp the polar angle to prevent flipping
    if (cameraAnglePolar.value < 0.1) cameraAnglePolar.value = 0.1;
    if (cameraAnglePolar.value > Math.PI / 2 - 0.1)
      cameraAnglePolar.value = Math.PI / 2 - 0.1;

    previousMousePosition = {
      x: event.clientX,
      y: event.clientY,
    };
  }
};

const handleMouseUp = (event) => {
  isMouseDown = false;
};

onMounted(() => {
  // Create scene
  const scene = new THREE.Scene();
  globalScene.value = scene;

  // Create camera
  const camera = new THREE.PerspectiveCamera(
    50,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  );

  // Create renderer
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  mountRef.value.appendChild(renderer.domElement);

  // Prevent the context menu from appearing on right-click within the canvas
  renderer.domElement.addEventListener("contextmenu", (event) => {
    event.preventDefault();
  });

  // Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
  directionalLight.position.set(5, 10, 7.5);
  scene.add(directionalLight);

  // Create ground
  const groundGeometry = new THREE.PlaneGeometry(200, 200);
  const groundMaterial = new THREE.MeshStandardMaterial({
    color: 0x228b22,
  }); // Forest green
  const ground = new THREE.Mesh(groundGeometry, groundMaterial);
  ground.rotation.x = -Math.PI / 2;
  scene.add(ground);

  // Add sky with enhanced gradient and clouds
  createSky(scene);

  // Add forest boundaries with enhanced trees
  addForestBoundaries(scene);

  // Create player with body, head, arms, and health bar
  createPlayer(scene);

  // Spawn initial monsters
  spawnInitialMonsters();

  // Keyboard event listeners
  window.addEventListener("keydown", handleKeyDown);
  window.addEventListener("keyup", handleKeyUp);
  window.addEventListener("blur", resetKeys);
  renderer.domElement.addEventListener("wheel", handleWheel, {
    passive: false,
  });

  // Mouse event listeners for camera control
  renderer.domElement.addEventListener("mousedown", handleMouseDown);
  renderer.domElement.addEventListener("mousemove", handleMouseMove);
  renderer.domElement.addEventListener("mouseup", handleMouseUp);

  // Animation loop
  const animate = () => {
    requestAnimationFrame(animate);

    // Player movement relative to camera's horizontal angle
    const moveSpeed = 0.1;
    const direction = new THREE.Vector3();

    direction
      .set(
        Math.sin(cameraAngleAzimuth.value),
        0,
        Math.cos(cameraAngleAzimuth.value)
      )
      .normalize();

    const right = new THREE.Vector3();
    right.crossVectors(direction, new THREE.Vector3(0, 1, 0)).normalize();

    if (keys.w) {
      playerGroup.position.addScaledVector(direction, -moveSpeed);
    }
    if (keys.s) {
      playerGroup.position.addScaledVector(direction, moveSpeed);
    }
    if (keys.a) {
      playerGroup.position.addScaledVector(right, moveSpeed);
    }
    if (keys.d) {
      playerGroup.position.addScaledVector(right, -moveSpeed);
    }

    // Prevent player from moving beyond boundaries
    const boundaryLimit = 90; // Slightly less than boundaryDistance to prevent overlap
    playerGroup.position.x = Math.max(
      -boundaryLimit,
      Math.min(boundaryLimit, playerGroup.position.x)
    );
    playerGroup.position.z = Math.max(
      -boundaryLimit,
      Math.min(boundaryLimit, playerGroup.position.z)
    );

    // Update player position reactive state
    playerPosition.x = playerGroup.position.x;
    playerPosition.y = playerGroup.position.y;
    playerPosition.z = playerGroup.position.z;

    // Collision detection with monsters
    monsters.forEach((monster) => {
      const distance = playerGroup.position.distanceTo(monster.mesh.position);
      if (distance < 2) {
        // Collision threshold
        if (monster.health > 0) {
          monster.takeDamage(playerStats.baseDamage);
        }
        // Player takes damage from monster
        if (playerStats.health > 0) {
          playerStats.health -= 1; // Monster damages player
          updatePlayerHealthBar();
          if (playerStats.health <= 0) {
            logMessages.value.unshift("Game Over!");
            playerStats.health = 0;
            // You can implement game over logic here
          }
        }
      }
    });

    // Jump logic
    if (jump) {
      playerGroup.position.y += 0.2;
      jumpHeight += 0.2;
      if (jumpHeight >= 10) {
        jump = false;
      }
    } else if (playerGroup.position.y > 1) {
      playerGroup.position.y -= 0.2;
      if (playerGroup.position.y <= 1) {
        playerGroup.position.y = 1;
        jumpHeight = 1; // Reset jump height
      }
    }

    // Camera controls
    const angleStep = 0.01;
    const polarStep = 0.005;

    if (keys.ArrowLeft) {
      cameraAngleAzimuth.value += angleStep;
    }
    if (keys.ArrowRight) {
      cameraAngleAzimuth.value -= angleStep;
    }
    if (keys.ArrowUp) {
      cameraAnglePolar.value -= polarStep;
      if (cameraAnglePolar.value < 0.1) cameraAnglePolar.value = 0.1;
    }
    if (keys.ArrowDown) {
      cameraAnglePolar.value += polarStep;
      if (cameraAnglePolar.value > Math.PI / 2 - 0.1)
        cameraAnglePolar.value = Math.PI / 2 - 0.1;
    }

    // Update camera position based on angles and distance
    const x =
      playerGroup.position.x +
      cameraDistance.value *
        Math.sin(cameraAngleAzimuth.value) *
        Math.sin(cameraAnglePolar.value);
    const y =
      playerGroup.position.y + cameraDistance.value * Math.cos(cameraAnglePolar.value);
    const z =
      playerGroup.position.z +
      cameraDistance.value *
        Math.cos(cameraAngleAzimuth.value) *
        Math.sin(cameraAnglePolar.value);
    camera.position.set(x, y, z);
    camera.lookAt(playerGroup.position);

    // Make health bars face the camera
    monsters.forEach((monster) => {
      if (monster.healthBar) {
        monster.healthBar.quaternion.copy(camera.quaternion);
      }
    });

    // Make player's health bar face the camera
    if (playerHealthBar) {
      playerHealthBar.quaternion.copy(camera.quaternion);
    }

    renderer.render(scene, camera);
  };
  animate();

  // Handle window resize
  const handleResize = () => {
    const width = window.innerWidth;
    const height = window.innerHeight;
    renderer.setSize(width, height);
    camera.aspect = width / height;
    camera.updateProjectionMatrix();
  };
  window.addEventListener("resize", handleResize);

  // Cleanup on unmount
  onBeforeUnmount(() => {
    window.removeEventListener("keydown", handleKeyDown);
    window.removeEventListener("keyup", handleKeyUp);
    window.removeEventListener("blur", resetKeys);
    window.removeEventListener("resize", handleResize);
    renderer.domElement.removeEventListener("wheel", handleWheel);
    renderer.domElement.removeEventListener("mousedown", handleMouseDown);
    renderer.domElement.removeEventListener("mousemove", handleMouseMove);
    renderer.domElement.removeEventListener("mouseup", handleMouseUp);
    renderer.domElement.removeEventListener("contextmenu", (event) => {
      event.preventDefault();
    });
    mountRef.value.removeChild(renderer.domElement);
  });
});
</script>
