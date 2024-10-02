<template>
  <div ref="mountRef" class="canvas-container">
    <div class="ui">
      <p>
        Player Position: X: {{ playerPosition.x.toFixed(2) }}, Y:
        {{ playerPosition.y.toFixed(2) }}, Z: {{ playerPosition.z.toFixed(2) }}
      </p>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref, reactive } from "vue";
import * as THREE from "three";

const mountRef = ref(null);
const globalScene = ref(null);
const playerPosition = reactive({ x: 0, y: 1, z: 0 });

// Control variables
const keys = reactive({
  w: false,
  a: false,
  s: false,
  d: false,
});

// Jump variables
let jump = false;
let jumpHeight = 1;

// Monster class
class Monster {
  constructor(mesh) {
    this.mesh = mesh;
    this.health = 3; // Initial health
  }

  takeDamage() {
    this.health -= 1;
    if (this.health <= 0) {
      // Remove from scene
      globalScene.value.remove(this.mesh);
    }
  }
}

// Array to hold monsters
const monsters = [];

const handleJump = () => {
  if (!jump) {
    jump = true;
  }
};

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

const handleKeyUp = (event) => {
  const { key } = event;
  if (keys.hasOwnProperty(key)) {
    keys[key] = false;
  }
};

onMounted(() => {
  // Create scene
  const scene = new THREE.Scene();
  globalScene.value = scene;
  scene.background = new THREE.Color(0x87ceeb); // Sky blue

  // Create camera
  const camera = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  );
  camera.position.set(0, 5, 10);
  camera.lookAt(0, 0, 0);

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
  const groundGeometry = new THREE.PlaneGeometry(100, 100);
  const groundMaterial = new THREE.MeshStandardMaterial({ color: 0x228b22 }); // Forest green
  const ground = new THREE.Mesh(groundGeometry, groundMaterial);
  ground.rotation.x = -Math.PI / 2;
  scene.add(ground);

  // Create player (simple blue sphere)
  const playerGeometry = new THREE.SphereGeometry(1, 32, 32);
  const playerMaterial = new THREE.MeshStandardMaterial({ color: 0x0000ff }); // Blue
  const player = new THREE.Mesh(playerGeometry, playerMaterial);
  player.position.y = 1;
  scene.add(player);

  // Create obstacles (red cubes)
  for (let i = 0; i < 10; i++) {
    const obstacleGeometry = new THREE.BoxGeometry(2, 2, 2);
    const obstacleMaterial = new THREE.MeshStandardMaterial({
      color: 0xff0000,
    }); // Red
    const obstacle = new THREE.Mesh(obstacleGeometry, obstacleMaterial);
    obstacle.position.set(Math.random() * 20 - 10, 1, Math.random() * 20 - 10);
    scene.add(obstacle);
    monsters.push(new Monster(obstacle));
  }

  // Keyboard event listeners
  window.addEventListener("keydown", handleKeyDown);
  window.addEventListener("keyup", handleKeyUp);

  // Animation loop
  const animate = () => {
    requestAnimationFrame(animate);

    // Player movement
    const moveSpeed = 0.1;
    if (keys.w) {
      player.position.z -= moveSpeed;
      playerPosition.z = player.position.z;
    }
    if (keys.s) {
      player.position.z += moveSpeed;
      playerPosition.z = player.position.z;
    }
    if (keys.a) {
      player.position.x -= moveSpeed;
      playerPosition.x = player.position.x;
    }
    if (keys.d) {
      player.position.x += moveSpeed;
      playerPosition.x = player.position.x;
    }

    // Collision detection with monsters
    monsters.forEach((monster, index) => {
      if (monster.health > 0) {
        const distance = player.position.distanceTo(monster.mesh.position);
        if (distance < 2) {
          // Collision threshold
          monster.takeDamage();
          console.log(`Monster ${index + 1} health: ${monster.health}`);
        }
      }
    });

    // Jump logic
    if (jump) {
      player.position.y += 0.1;
      jumpHeight += 0.1;
      if (jumpHeight >= 3) {
        jump = false;
      }
    } else if (player.position.y > 1) {
      player.position.y -= 0.1;
      if (player.position.y <= 1) {
        player.position.y = 1;
      }
    }

    // Camera follows player
    camera.position.x = player.position.x;
    camera.position.z = player.position.z + 10;
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
  background: rgba(255, 255, 255, 0.7);
  padding: 10px;
  border-radius: 5px;
  font-family: Arial, sans-serif;
}
</style>