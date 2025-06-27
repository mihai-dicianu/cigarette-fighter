<template>
  <div ref="container" class="cigarette-container" @click="handleClick">
    <div class="health-bar">
      <div class="health-fill" :style="{ width: healthPercentage + '%' }"></div>
      <div class="health-text">{{ health }} / {{ maxHealth }}</div>
    </div>

    <!-- Victory overlay -->
    <div v-if="showVictory" class="victory-overlay">
      <div class="victory-content">
        <h1 class="victory-title">🎉 Congratulations! 🎉</h1>
        <p class="victory-message">You defeated the cigarette!</p>
        <button class="retry-button" @click="resetGame">Play Again</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'

const container = ref<HTMLDivElement>()

let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let cigarette: THREE.Group
let animationId: number
let raycaster: THREE.Raycaster
let mouse: THREE.Vector2
let hitMarkers: THREE.Group[] = []

// Health system
const health = ref(0)
const maxHealth = 100
const healthPercentage = ref(0)
const showVictory = ref(false)

// Movement system
let cigaretteVelocity = new THREE.Vector3(0.012, 0.006, 0)
let cigarettePosition = new THREE.Vector3(0, 0, 0)
let movementBounds = { x: 2, y: 1.5, z: 0 }
let isMoving = true

const init = () => {
  if (!container.value) return

  // Scene setup
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x1a1a1a)

  // Camera setup
  const width = container.value.clientWidth
  const height = container.value.clientHeight
  camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000)
  camera.position.set(0, 0, 5)

  // Renderer setup
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(width, height)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  container.value.appendChild(renderer.domElement)

  // Raycaster for click detection
  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()

  // Lighting
  const ambientLight = new THREE.AmbientLight(0x404040, 0.6)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(5, 5, 5)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048
  scene.add(directionalLight)

  const pointLight = new THREE.PointLight(0xff6b35, 1, 100)
  pointLight.position.set(0, 2, 2)
  scene.add(pointLight)

  // Create cigarette
  createCigarette()

  // Start animation
  animate()

  // Handle window resize
  window.addEventListener('resize', onWindowResize)
}

const createCigarette = () => {
  cigarette = new THREE.Group()

  // Cigarette body (white paper) - made wider
  const bodyGeometry = new THREE.CylinderGeometry(0.12, 0.12, 2, 8)
  const bodyMaterial = new THREE.MeshLambertMaterial({
    color: 0xf5f5f5,
    transparent: true,
    opacity: 0.9,
  })
  const body = new THREE.Mesh(bodyGeometry, bodyMaterial)
  body.castShadow = true
  body.receiveShadow = true
  cigarette.add(body)

  // Filter (orange/brown) - made wider to match body and positioned more reliably
  const filterGeometry = new THREE.CylinderGeometry(0.12, 0.12, 0.3, 8)
  const filterMaterial = new THREE.MeshLambertMaterial({
    color: 0xd2691e,
    transparent: false,
    opacity: 1,
  })
  const filter = new THREE.Mesh(filterGeometry, filterMaterial)
  // Position filter at the top of the cigarette body
  filter.position.y = 1.15
  filter.castShadow = true
  filter.receiveShadow = true
  // Ensure filter is always visible by setting render order
  filter.renderOrder = 1
  cigarette.add(filter)

  // Add invisible hit zones for easier targeting
  const hitZoneGeometry = new THREE.CylinderGeometry(0.18, 0.18, 2.5, 8)
  const hitZoneMaterial = new THREE.MeshBasicMaterial({
    transparent: true,
    opacity: 0, // Invisible
    side: THREE.DoubleSide,
  })
  const hitZone = new THREE.Mesh(hitZoneGeometry, hitZoneMaterial)
  cigarette.add(hitZone)

  // Add some texture to the cigarette
  const textureLoader = new THREE.TextureLoader()

  // Create a subtle texture for the paper
  const paperTexture = new THREE.CanvasTexture(createPaperTexture())
  bodyMaterial.map = paperTexture

  scene.add(cigarette)
}

const createPaperTexture = () => {
  const canvas = document.createElement('canvas')
  canvas.width = 64
  canvas.height = 64
  const ctx = canvas.getContext('2d')!

  // Create subtle paper texture
  ctx.fillStyle = '#f5f5f5'
  ctx.fillRect(0, 0, 64, 64)

  // Add some noise
  for (let i = 0; i < 100; i++) {
    ctx.fillStyle = `rgba(200, 200, 200, ${Math.random() * 0.1})`
    ctx.fillRect(Math.random() * 64, Math.random() * 64, Math.random() * 2, Math.random() * 2)
  }

  return canvas
}

const createHitMarker = (position: THREE.Vector3) => {
  const hitMarker = new THREE.Group()

  // Create expanding ring effect
  const ringGeometry = new THREE.RingGeometry(0.1, 0.15, 8)
  const ringMaterial = new THREE.MeshBasicMaterial({
    color: 0xff4444,
    transparent: true,
    opacity: 0.8,
    side: THREE.DoubleSide,
  })
  const ring = new THREE.Mesh(ringGeometry, ringMaterial)
  hitMarker.add(ring)

  // Create crosshair effect
  const crosshairGeometry = new THREE.BoxGeometry(0.02, 0.1, 0.02)
  const crosshairMaterial = new THREE.MeshBasicMaterial({ color: 0xff4444 })

  const horizontal = new THREE.Mesh(crosshairGeometry, crosshairMaterial)
  const vertical = new THREE.Mesh(crosshairGeometry, crosshairMaterial)
  vertical.rotation.z = Math.PI / 2

  hitMarker.add(horizontal)
  hitMarker.add(vertical)

  // Create damage number
  const damageGeometry = new THREE.PlaneGeometry(0.3, 0.1)
  const damageMaterial = new THREE.MeshBasicMaterial({
    color: 0xffffff,
    transparent: true,
    opacity: 0.9,
  })
  const damage = new THREE.Mesh(damageGeometry, damageMaterial)
  damage.position.y = 0.2
  hitMarker.add(damage)

  hitMarker.position.copy(position)
  hitMarker.position.z += 0.1 // Slightly in front

  scene.add(hitMarker)
  hitMarkers.push(hitMarker)

  // Animate the hit marker
  animateHitMarker(hitMarker)
}

const animateHitMarker = (hitMarker: THREE.Group) => {
  let scale = 1
  let opacity = 1
  let yOffset = 0

  const animate = () => {
    scale += 0.05
    opacity -= 0.02
    yOffset += 0.01

    hitMarker.scale.set(scale, scale, scale)
    hitMarker.position.y += 0.01

    // Update materials opacity
    hitMarker.children.forEach((child) => {
      if (child instanceof THREE.Mesh && child.material instanceof THREE.MeshBasicMaterial) {
        child.material.opacity = opacity
      }
    })

    if (opacity > 0) {
      requestAnimationFrame(animate)
    } else {
      // Remove from scene and array
      scene.remove(hitMarker)
      const index = hitMarkers.indexOf(hitMarker)
      if (index > -1) {
        hitMarkers.splice(index, 1)
      }
    }
  }

  animate()
}

const handleClick = (event: MouseEvent) => {
  if (!container.value || showVictory.value) return

  // Calculate mouse position in normalized device coordinates
  const rect = container.value.getBoundingClientRect()
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  // Update the picking ray with the camera and mouse position
  raycaster.setFromCamera(mouse, camera)

  // Calculate objects intersecting the picking ray
  const intersects = raycaster.intersectObjects(cigarette.children, true)

  if (intersects.length > 0) {
    // Hit the cigarette!
    const hitPoint = intersects[0].point

    // Create hit marker at the hit point
    createHitMarker(hitPoint)

    // Increase health
    const damage = Math.floor(Math.random() * 10) + 5 // Random damage between 5-15
    health.value = Math.min(health.value + damage, maxHealth)
    healthPercentage.value = (health.value / maxHealth) * 100

    // Check for victory
    if (health.value >= maxHealth) {
      triggerVictory()
      return
    }

    // Change cigarette direction when hit
    changeCigaretteDirection()

    // Add screen shake effect
    addScreenShake()

    // Add cigarette recoil effect
    addCigaretteRecoil()
  }
}

const triggerVictory = () => {
  showVictory.value = false
  isMoving = false

  // Create snapping animation
  snapCigarette()

  // Add victory particles
  createVictoryParticles()

  // Show popup after a delay (e.g., 1.2s)
  setTimeout(() => {
    showVictory.value = true
  }, 1200)
}

const snapCigarette = () => {
  if (!cigarette) return

  // Break the cigarette into pieces
  const pieces: THREE.Mesh[] = []

  // Create multiple pieces with better proportions
  for (let i = 0; i < 6; i++) {
    const isFilter = i === 0 // First piece is the filter
    const pieceGeometry = new THREE.CylinderGeometry(0.12, 0.12, isFilter ? 0.3 : 0.4, 8)
    const pieceMaterial = new THREE.MeshLambertMaterial({
      color: isFilter ? 0xd2691e : 0xf5f5f5,
      transparent: true,
      opacity: isFilter ? 0.8 : 0.9,
    })
    const piece = new THREE.Mesh(pieceGeometry, pieceMaterial)

    // Position pieces around the original cigarette
    piece.position.copy(cigarette.position)
    piece.position.x += (Math.random() - 0.5) * 0.8
    piece.position.y += (Math.random() - 0.5) * 0.8
    piece.position.z += (Math.random() - 0.5) * 0.8

    // Random rotation
    piece.rotation.x = Math.random() * Math.PI * 2
    piece.rotation.y = Math.random() * Math.PI * 2
    piece.rotation.z = Math.random() * Math.PI * 2

    // Ensure pieces are visible
    piece.renderOrder = 2
    piece.castShadow = true
    piece.receiveShadow = true

    scene.add(piece)
    pieces.push(piece)
  }

  // Hide original cigarette
  cigarette.visible = false

  // Animate pieces flying apart
  pieces.forEach((piece, index) => {
    const velocity = new THREE.Vector3(
      (Math.random() - 0.5) * 0.15,
      (Math.random() - 0.5) * 0.15,
      (Math.random() - 0.5) * 0.15,
    )

    let opacity = 1
    const animate = () => {
      piece.position.add(velocity)
      piece.rotation.x += 0.15
      piece.rotation.y += 0.15
      piece.rotation.z += 0.15

      opacity -= 0.015
      if (piece.material instanceof THREE.MeshLambertMaterial) {
        piece.material.opacity = Math.max(0, opacity)
      }

      if (opacity > 0) {
        requestAnimationFrame(animate)
      } else {
        scene.remove(piece)
      }
    }

    setTimeout(() => animate(), index * 80)
  })
}

const createVictoryParticles = () => {
  for (let i = 0; i < 20; i++) {
    setTimeout(() => {
      const particleGeometry = new THREE.SphereGeometry(0.02, 8, 8)
      const particleMaterial = new THREE.MeshBasicMaterial({
        color: Math.random() > 0.5 ? 0xffd700 : 0xff6b35,
        transparent: true,
        opacity: 1,
      })
      const particle = new THREE.Mesh(particleGeometry, particleMaterial)

      // Random position around the center
      particle.position.set(
        (Math.random() - 0.5) * 4,
        (Math.random() - 0.5) * 3,
        (Math.random() - 0.5) * 2,
      )

      scene.add(particle)

      // Animate particle
      const velocity = new THREE.Vector3(
        (Math.random() - 0.5) * 0.05,
        Math.random() * 0.05 + 0.02,
        (Math.random() - 0.5) * 0.05,
      )

      let opacity = 1
      const animate = () => {
        particle.position.add(velocity)
        particle.rotation.x += 0.1
        particle.rotation.y += 0.1

        opacity -= 0.01
        particleMaterial.opacity = opacity

        if (opacity > 0) {
          requestAnimationFrame(animate)
        } else {
          scene.remove(particle)
        }
      }

      animate()
    }, i * 50)
  }
}

const changeCigaretteDirection = () => {
  // Randomly change velocity direction
  const speed = cigaretteVelocity.length()
  const angle = Math.random() * Math.PI * 2

  cigaretteVelocity.x = Math.cos(angle) * speed * (0.8 + Math.random() * 0.4) // Vary speed slightly
  cigaretteVelocity.y = Math.sin(angle) * speed * (0.8 + Math.random() * 0.4)

  // Occasionally add some z movement for more unpredictability
  if (Math.random() < 0.3) {
    cigaretteVelocity.z = (Math.random() - 0.5) * 0.01
  } else {
    cigaretteVelocity.z = 0
  }

  // Increase speed slightly as health increases (gets harder)
  const speedMultiplier = 1 + (health.value / maxHealth) * 0.5
  cigaretteVelocity.multiplyScalar(speedMultiplier)
}

const addScreenShake = () => {
  const originalPosition = camera.position.clone()
  let shakeIntensity = 0.1
  let shakeCount = 0

  const shake = () => {
    camera.position.x = originalPosition.x + (Math.random() - 0.5) * shakeIntensity
    camera.position.y = originalPosition.y + (Math.random() - 0.5) * shakeIntensity

    shakeIntensity *= 0.9
    shakeCount++

    if (shakeCount < 10) {
      requestAnimationFrame(shake)
    } else {
      camera.position.copy(originalPosition)
    }
  }

  shake()
}

const addCigaretteRecoil = () => {
  const originalRotation = cigarette.rotation.clone()
  let recoilIntensity = 0.2
  let recoilCount = 0

  const recoil = () => {
    cigarette.rotation.x = originalRotation.x + (Math.random() - 0.5) * recoilIntensity
    cigarette.rotation.z = originalRotation.z + (Math.random() - 0.5) * recoilIntensity

    recoilIntensity *= 0.8
    recoilCount++

    if (recoilCount < 8) {
      requestAnimationFrame(recoil)
    } else {
      cigarette.rotation.copy(originalRotation)
    }
  }

  recoil()
}

const animate = () => {
  animationId = requestAnimationFrame(animate)

  if (cigarette && isMoving) {
    // Update position based on velocity
    cigarettePosition.add(cigaretteVelocity)

    // Bounce off boundaries
    if (Math.abs(cigarettePosition.x) > movementBounds.x) {
      cigaretteVelocity.x *= -1
      cigarettePosition.x = Math.sign(cigarettePosition.x) * movementBounds.x
    }

    if (Math.abs(cigarettePosition.y) > movementBounds.y) {
      cigaretteVelocity.y *= -1
      cigarettePosition.y = Math.sign(cigarettePosition.y) * movementBounds.y
    }

    if (Math.abs(cigarettePosition.z) > movementBounds.z) {
      cigaretteVelocity.z *= -1
      cigarettePosition.z = Math.sign(cigarettePosition.z) * movementBounds.z
    }

    // Apply position to cigarette
    cigarette.position.copy(cigarettePosition)

    // Gentle rotation
    cigarette.rotation.y += 0.005

    // Add slight wobble based on movement
    cigarette.rotation.x = Math.sin(Date.now() * 0.002) * 0.1
    cigarette.rotation.z = Math.cos(Date.now() * 0.003) * 0.05
  }

  renderer.render(scene, camera)
}

const onWindowResize = () => {
  if (!container.value) return

  const width = container.value.clientWidth
  const height = container.value.clientHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

const resetGame = () => {
  // Reset health
  health.value = 0
  healthPercentage.value = 0
  showVictory.value = false

  // Reset cigarette
  if (cigarette) {
    cigarette.visible = true
    cigarette.position.set(0, 0, 0)
    cigarette.rotation.set(0, 0, 0)
  }

  // Reset movement
  cigarettePosition.set(0, 0, 0)
  cigaretteVelocity.set(0.012, 0.006, 0)
  isMoving = true

  // Clear any remaining hit markers
  hitMarkers.forEach((marker) => {
    scene.remove(marker)
  })
  hitMarkers = []
}

onMounted(() => {
  init()
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  window.removeEventListener('resize', onWindowResize)
  if (renderer) {
    renderer.dispose()
  }
})
</script>

<style scoped>
.cigarette-container {
  width: 100%;
  height: 100vh;
  background: linear-gradient(135deg, #1a1a1a 0%, #2d2d2d 100%);
  overflow: hidden;
  cursor: crosshair;
  position: relative;
}

.health-bar {
  position: absolute;
  top: 20px;
  left: 20px;
  width: 200px;
  height: 30px;
  background: rgba(0, 0, 0, 0.7);
  border: 2px solid #333;
  border-radius: 15px;
  overflow: hidden;
  z-index: 100;
}

.health-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff4444 0%, #ffaa44 50%, #44ff44 100%);
  transition: width 0.3s ease;
  border-radius: 13px;
}

.health-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #00ff00;
  font-family: 'Arial', sans-serif;
  font-weight: bold;
  font-size: 12px;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.9);
  z-index: 101;
}

.victory-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 102;
}

.victory-content {
  background: linear-gradient(135deg, #2c3e50 0%, #34495e 100%);
  padding: 40px;
  border-radius: 15px;
  text-align: center;
  border: 3px solid #3498db;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
}

.victory-title {
  font-size: 32px;
  font-weight: bold;
  margin-bottom: 15px;
  color: #00ff00;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.8);
}

.victory-message {
  font-size: 20px;
  margin-bottom: 30px;
  color: #ffffff;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.8);
}

.retry-button {
  padding: 15px 30px;
  font-size: 18px;
  background: linear-gradient(45deg, #e74c3c, #c0392b);
  color: #ffffff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
}

.retry-button:hover {
  background: linear-gradient(45deg, #c0392b, #a93226);
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.4);
}
</style>
