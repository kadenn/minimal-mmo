<template>
  <div ref="mountRef" class="canvas-container">
    <div class="ui">
      <p>
        Level: {{ level }} | Damage:
        {{ playerStats.baseDamage }}
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
  baseDamage: 1, // Base damage per hit
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

// Function to create a simple tree
const createTree = () => {
  const tree = new THREE.Group();

  // Trunk
  const trunkGeometry = new THREE.CylinderGeometry(0.2, 0.2, 2, 8);
  const trunkMaterial = new THREE.MeshStandardMaterial({ color: 0x8b4513 });
  const trunk = new THREE.Mesh(trunkGeometry, trunkMaterial);
  trunk.position.y = 1; // Half of trunk height
  tree.add(trunk);

  // Leaves
  const leavesGeometry = new THREE.ConeGeometry(1, 3, 8);
  const leavesMaterial = new THREE.MeshStandardMaterial({ color: 0x228b22 });
  const leaves = new THREE.Mesh(leavesGeometry, leavesMaterial);
  leaves.position.y = 3; // Top of the trunk
  tree.add(leaves);

  return tree;
};

// Function to add forest boundaries
const addForestBoundaries = (scene) => {
  const boundaryDistance = 100; // Matches the ground size
  const treeSpacing = 10; // Distance between trees

  // Place trees along the perimeter
  for (let x = -boundaryDistance; x <= boundaryDistance; x += treeSpacing) {
    const z1 = -boundaryDistance;
    const z2 = boundaryDistance;
    const tree1 = createTree();
    tree1.position.set(x, 0, z1);
    scene.add(tree1);

    const tree2 = createTree();
    tree2.position.set(x, 0, z2);
    scene.add(tree2);
  }

  for (let z = -boundaryDistance; z <= boundaryDistance; z += treeSpacing) {
    const x1 = -boundaryDistance;
    const x2 = boundaryDistance;
    const tree1 = createTree();
    tree1.position.set(x1, 0, z);
    scene.add(tree1);

    const tree2 = createTree();
    tree2.position.set(x2, 0, z);
    scene.add(tree2);
  }
};

// Function to create a health bar sprite with monster name
const createHealthBar = (name, initialHealth, maxHealth) => {
  const canvas = document.createElement("canvas");
  canvas.width = 128;
  canvas.height = 32; // Increased height to accommodate text and health bar
  const context = canvas.getContext("2d");

  // Draw monster name
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
    1, // Half of height to sit on ground
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
  keys[key] = true;
  if (key === " ") {
    // Spacebar to jump
    handleJump();
  }
};

// Handle key up events
const handleKeyUp = (event) => {
  const { key } = event;
  keys[key] = false;
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

// Function to create player with name label
const createPlayer = (scene) => {
  player = new THREE.Mesh(
    new THREE.SphereGeometry(1, 32, 32),
    new THREE.MeshStandardMaterial({ color: 0x0000ff }) // Blue
  );
  player.position.set(0, 1, 0);
  scene.add(player);
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

  // Add forest boundaries
  addForestBoundaries(scene);

  // Create player with name label
  createPlayer(scene);

  // Spawn initial monsters
  spawnInitialMonsters();

  // Keyboard event listeners
  window.addEventListener("keydown", handleKeyDown);
  window.addEventListener("keyup", handleKeyUp);
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
      player.position.addScaledVector(direction, -moveSpeed);
    }
    if (keys.s) {
      player.position.addScaledVector(direction, moveSpeed);
    }
    if (keys.a) {
      player.position.addScaledVector(right, moveSpeed);
    }
    if (keys.d) {
      player.position.addScaledVector(right, -moveSpeed);
    }

    // Prevent player from moving beyond boundaries
    const boundaryLimit = 90; // Slightly less than boundaryDistance to prevent overlap
    player.position.x = Math.max(
      -boundaryLimit,
      Math.min(boundaryLimit, player.position.x)
    );
    player.position.z = Math.max(
      -boundaryLimit,
      Math.min(boundaryLimit, player.position.z)
    );

    // Update player position reactive state
    playerPosition.x = player.position.x;
    playerPosition.y = player.position.y;
    playerPosition.z = player.position.z;

    // Collision detection with monsters
    monsters.forEach((monster) => {
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
      player.position.y += 0.2;
      jumpHeight += 0.2;
      if (jumpHeight >= 10) {
        jump = false;
      }
    } else if (player.position.y > 1) {
      player.position.y -= 0.2;
      if (player.position.y <= 1) {
        player.position.y = 1;
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

    // Make health bars face the camera
    monsters.forEach((monster) => {
      if (monster.healthBar) {
        monster.healthBar.quaternion.copy(camera.quaternion);
      }
    });

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