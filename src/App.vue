<template>
  <div ref="mountRef" class="canvas-container">
    <div class="ui">
      <p>Experience: {{ experience }} | Level: {{ level }}</p>
      <p>Damage: {{ playerStats.baseDamage }}</p>
      <div class="log">
        <ul>
          <li v-for="message in logMessages.slice(0, 4)">
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
  baseDamage: 1, // Base damage per hit
});

// Camera control variables
const cameraAngleAzimuth = ref(0); // Horizontal angle in radians
const cameraAnglePolar = ref(Math.PI / 4); // Vertical angle in radians
const cameraDistance = ref(20); 
const minDistance = 5;
const maxDistance = 100;

// Monster Types
const monsterTypes = [
  { color: 0x00ff00, health: 10 }, // Green
  { color: 0x0000ff, health: 20 }, // Blue
  { color: 0xffff00, health: 30 }, // Yellow
  { color: 0xff00ff, health: 40 }, // Magenta
  { color: 0xff0000, health: 50 }, // Red
];

// Monster class
class Monster {
  constructor(mesh, type) {
    this.mesh = mesh;
    this.health = type.health;
    this.type = type;
  }

  takeDamage(damage) {
    this.health -= damage;
    if (this.health <= 0) {
      // Remove from scene
      globalScene.value.remove(this.mesh);
      // Remove from monsters array
      const index = monsters.indexOf(this);
      if (index > -1) {
        monsters.splice(index, 1);
      }
      // Log the kill and gain experience
      experience.value += 15;
      logMessages.value.unshift(`Monster killed! Gained 15 XP.`);
      // Check for level up
      checkLevelUp();
      // Spawn a new monster
      spawnMonster();
    }
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
    Math.random() * 100 - 50,
    1, // Half of height to sit on ground
    Math.random() * 100 - 50
  );
  globalScene.value.add(monster);
  monsters.push(new Monster(monster, type));
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
  player.scale.set(level.value, level.value, level.value);
};

// Function to increase player damage based on level
const increaseDamage = () => {
  playerStats.baseDamage += 0.5; // Example increment
};

// Player mesh (declared here to access in functions)
let player;

// Player scaling
player = new THREE.Mesh(
  new THREE.SphereGeometry(1, 32, 32),
  new THREE.MeshStandardMaterial({ color: 0x0000ff }) // Blue
);
player.position.y = 1;
globalScene.value?.add(player); // Will be properly added in onMounted

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

onMounted(() => {
  // Create scene
  const scene = new THREE.Scene();
  globalScene.value = scene;
  scene.background = new THREE.Color(0x87ceeb); // Sky blue

  // Create camera
  const camera = new THREE.PerspectiveCamera(
    50,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  );

  // Initial camera position
  const updateCameraPosition = () => {
    const x =
      player.position.x +
      cameraDistance.value *
        Math.sin(cameraAngleAzimuth.value) *
        Math.sin(cameraAnglePolar.value);
    const y =
      player.position.y +
      cameraDistance.value * Math.cos(cameraAnglePolar.value);
    const z =
      player.position.z +
      cameraDistance.value *
        Math.cos(cameraAngleAzimuth.value) *
        Math.sin(cameraAnglePolar.value);
    camera.position.set(x, y, z);
    camera.lookAt(player.position);
  };
  updateCameraPosition();

  // Create renderer
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  mountRef.value.appendChild(renderer.domElement);

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

  // Create player (simple blue sphere)
  player.position.set(0, 1, 0);
  scene.add(player);

  // Spawn initial monsters
  spawnInitialMonsters();

  // Keyboard event listeners
  window.addEventListener("keydown", handleKeyDown);
  window.addEventListener("keyup", handleKeyUp);
  renderer.domElement.addEventListener("wheel", handleWheel, {
    passive: false,
  });

  // Animation loop
  const animate = () => {
    requestAnimationFrame(animate);

    // Player movement relative to camera's horizontal angle
    const moveSpeed = -0.1;
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
      player.position.addScaledVector(direction, moveSpeed);
    }
    if (keys.s) {
      player.position.addScaledVector(direction, -moveSpeed);
    }
    if (keys.a) {
      player.position.addScaledVector(right, -moveSpeed);
    }
    if (keys.d) {
      player.position.addScaledVector(right, moveSpeed);
    }

    // Update player position reactive state
    playerPosition.x = player.position.x;
    playerPosition.y = player.position.y;
    playerPosition.z = player.position.z;

    // Collision detection with monsters
    monsters.forEach((monster, index) => {
      if (monster.health > 0) {
        const distance = player.position.distanceTo(monster.mesh.position);
        if (distance < 2) {
          // Collision threshold
          monster.takeDamage(playerStats.baseDamage);
        }
      }
    });

    // Jump logic
    if (jump) {
      player.position.y += 0.1;
      jumpHeight += 0.1;
      if (jumpHeight >= 5) {
        jump = false;
      }
    } else if (player.position.y > 1) {
      player.position.y -= 0.1;
      if (player.position.y <= 1) {
        player.position.y = 1;
        jumpHeight = 1; // Reset jump height
      }
    }

    // Camera controls
    const rotationSpeed = 0.02;
    const angleStep = rotationSpeed;
    const polarStep = rotationSpeed;

    if (keys.ArrowLeft) {
      cameraAngleAzimuth.value += angleStep;
    }
    if (keys.ArrowRight) {
      cameraAngleAzimuth.value -= angleStep;
    }
    if (keys.ArrowDown) {
      cameraAnglePolar.value -= polarStep;
      if (cameraAnglePolar.value < 0.1) cameraAnglePolar.value = 0.1;
    }
    if (keys.ArrowUp) {
      cameraAnglePolar.value += polarStep;
      if (cameraAnglePolar.value > Math.PI / 2 - 0.1)
        cameraAnglePolar.value = Math.PI / 2 - 0.1;
    }

    // Update camera position based on angles and distance
    const x =
      player.position.x +
      cameraDistance.value *
        Math.sin(cameraAngleAzimuth.value) *
        Math.sin(cameraAnglePolar.value);
    const y =
      player.position.y +
      cameraDistance.value * Math.cos(cameraAnglePolar.value);
    const z =
      player.position.z +
      cameraDistance.value *
        Math.cos(cameraAngleAzimuth.value) *
        Math.sin(cameraAnglePolar.value);
    camera.position.set(x, y, z);
    camera.lookAt(player.position);

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
    window.removeEventListener("resize", handleResize);
    renderer.domElement.removeEventListener("wheel", handleWheel);
    mountRef.value.removeChild(renderer.domElement);
  });
});
</script>

<style scoped>
.canvas-container {
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
}

.ui {
  position: absolute;
  top: 10px;
  left: 10px;
  background: rgba(255, 255, 255, 0.9);
  padding: 15px;
  border-radius: 8px;
  font-family: Arial, sans-serif;
  max-width: 300px;
}

.log {
  margin-top: 10px;
  max-height: 200px;
  overflow-y: auto;
}

.log ul {
  list-style-type: none;
  padding: 0;
  margin: 0;
}

.log li {
  background: #f0f0f0;
  margin-bottom: 5px;
  padding: 5px;
  border-radius: 4px;
  font-size: 14px;
}
</style>
