<template>
  <main class="game">
    <div class="game__score">Score: {{ gameState.score }}</div>
    <div class="game__body" v-if="gameState.score == 0">
      This is a simple game of asteroids experiment. Use your keyboard to move the ship and spacebar. The ship will be controlled autonomously after 5 seconds if you do nothing.
    </div>
    <canvas ref="gameCanvas"></canvas>
  </main>
</template>

<script setup lang="ts">
import * as Tone from 'tone'
import { onMounted, reactive, ref } from 'vue'

const gameCanvas = ref<HTMLCanvasElement>()

type GameObject = {
  angle: number
  speed: number
  x: number
  y: number
}

type Astroid = GameObject & {
  size: number
  speed: number
  vertices: { x: number; y: number }[]
}

type Bullet = GameObject & {}

/**
 * A reactive state object that holds the state of the game. This includes the status of the player and
 * non-player objects like asteroids and bullets, and other relevant game metrics such as score and time.
 */
const gameState = reactive<{
  aiShootCooldown: number // Time interval between AI-controlled shots.
  asteroids: Astroid[] // An array of current asteroids in the game.
  bullets: Bullet[] // An array of bullets that have been fired.

  // An object representing the current state of keyboard inputs.
  keys: {
    [key: string]: boolean
  }
  lastAIShootTime: number // Timestamp of the last shot made by the AI.
  level: number // The current level of the game.
  player: GameObject // GameObject representing the player's ship, including position and angle.
  score: number // The current score of the player.
  timePlaying: number // The amount of time elapsed since the start of the game.
  timeStart: number // The timestamp when the game was started or reset.

  userHasInteracted: boolean //  A flag indicating whether the player has interacted with the game.
}>({
  aiShootCooldown: 200,
  asteroids: [],
  bullets: [],
  keys: {
    ArrowLeft: false,
    ArrowRight: false,
    ArrowUp: false,
    Space: false,
  },
  lastAIShootTime: 0,
  level: 1,
  player: {
    angle: 0,
    speed: 0,
    x: 0,
    y: 0,
  },
  score: 0,
  timePlaying: 0,
  timeStart: 0,

  userHasInteracted: false,
})

let bulletSynth: Tone.Synth
let explosionSynth: Tone.MembraneSynth
let gameOverSynth: Tone.MetalSynth

onMounted(() => {
  initCanvas()
  initSound()
  bindEvents()
  resetGame()

  gameLoop()
})

/**
 * Initialises the canvas element for the game. Sets the canvas dimensions
 * to match the current browser window size. This function should be called
 * when the component is mounted to ensure the canvas is correctly sized.
 */
function initCanvas() {
  if (!gameCanvas.value) return
  gameCanvas.value.width = window.innerWidth
  gameCanvas.value.height = window.innerHeight
}

function bindEvents() {
  window.addEventListener('keydown', keyDownHandler)
  window.addEventListener('keyup', keyUpHandler)
}

/**
 * Initialises the sound synthesizers for the game using Tone.js. This includes
 * sounds for various game actions like shooting bullets and explosions.
 */
function initSound() {
  bulletSynth = new Tone.Synth().toDestination()
  explosionSynth = new Tone.MembraneSynth().toDestination()
  gameOverSynth = new Tone.MetalSynth().toDestination()
}

function resetGame() {
  if (!gameCanvas.value) return
  gameState.score = 0
  gameState.player.x = gameCanvas.value.width / 2
  gameState.player.y = gameCanvas.value.height / 2
  gameState.player.angle = 0
  gameState.player.speed = 0
  gameState.bullets = []
  gameState.asteroids = []
  gameState.timeStart = 0
  gameState.timePlaying = 0
  gameState.lastAIShootTime = 0
  gameState.aiShootCooldown = 200
  gameState.userHasInteracted = false
  createInitialAsteroids()
}

/**
 * Creates a specified number of asteroids at random positions on the canvas.
 * This function is used to populate the game with initial obstacles when the
 * game starts or after a reset.
 */
function createInitialAsteroids() {
  if (!gameCanvas.value) return

  for (let index = 0; index < gameState.level; index++) {
    let x, y
    do {
      x = Math.random() * gameCanvas.value.width
      y = Math.random() * gameCanvas.value.height
    } while (Math.hypot(gameState.player.x - x, gameState.player.y - y) < 100) // Avoid spawning near the ship
    createAsteroid(x, y)
  }
}

/**
 * Generates a single asteroid with random properties. The asteroid's size, speed,
 * and starting position are determined randomly. This function is used both in
 * initialising the game and in creating new asteroids during gameplay.
 *
 * @param {number} x - The x-coordinate for the asteroid's initial position.
 * @param {number} y - The y-coordinate for the asteroid's initial position.
 * @param {number} size - Optional. The size of the asteroid.
 */
function createAsteroid(x: number, y: number, size?: number) {
  if (!gameCanvas.value) return

  if (!size) {
    size = Math.random() * 20 + 15
  }

  gameState.asteroids.push({
    angle: Math.random() * Math.PI * 2,
    size,
    speed: Math.random() * 2 + 1,
    vertices: generateAsteroidVertices(size),
    x: x || Math.random() * gameCanvas.value.width,
    y: y || Math.random() * gameCanvas.value.height,
  })
}

/**
 * Generates the vertices for an asteroid's polygonal shape. This function creates a set
 * of points (vertices) that define the irregular shape of an asteroid, based on its size.
 * The number of vertices and their positions are randomised to give each asteroid a unique shape.
 *
 * @param {number} size - The size of the asteroid, which influences the radius of the vertices.
 * @returns {Array<{x: number, y: number}>} An array of points representing the vertices of the asteroid.
 */
function generateAsteroidVertices(size: number) {
  const vertices = []
  for (let index = 0; index < 6; index++) {
    const angle = (index / 6) * Math.PI * 2
    const radius = size * (0.5 + Math.random() * 0.5)
    vertices.push({
      x: Math.cos(angle) * radius,
      y: Math.sin(angle) * radius,
    })
  }
  return vertices
}

function keyDownHandler(event: KeyboardEvent) {
  gameState.userHasInteracted = true
  if (event.code === 'Space') {
    // Spacebar
    gameState.keys.Space = true
  } else if (event.code in gameState.keys) {
    gameState.keys[event.code] = true
  }
}

function keyUpHandler(event: KeyboardEvent) {
  if (event.code === 'Space') {
    // Spacebar
    gameState.keys.Space = false
  } else if (event.code in gameState.keys) {
    gameState.keys[event.code] = false
  }
}

/**
 * Main game logic for each frame. This updates the state of all game objects,
 * including the player's ship, bullets, and asteroids. It handles movements,
 * collisions, and game state changes.
 */
function update() {
  if (!gameCanvas.value) return

  // Player movement
  if (gameState.keys.ArrowUp) {
    gameState.player.speed = 5
  } else {
    gameState.player.speed *= 0.99 // friction
  }

  if (gameState.keys.ArrowLeft) {
    gameState.player.angle -= 0.05
  }

  if (gameState.keys.ArrowRight) {
    gameState.player.angle += 0.05
  }

  gameState.player.x += gameState.player.speed * Math.cos(gameState.player.angle)
  gameState.player.y += gameState.player.speed * Math.sin(gameState.player.angle)

  wrapAround(gameState.player)

  // Shooting
  if (gameState.keys.Space && gameState.bullets.length < 10) {
    shootBullet()
    gameState.keys.Space = false
  }

  // Update bullets
  for (const [index, bullet] of gameState.bullets.entries()) {
    bullet.x += bullet.speed * Math.cos(bullet.angle)
    bullet.y += bullet.speed * Math.sin(bullet.angle)

    if (bullet.x < 0 || bullet.x > gameCanvas.value.width || bullet.y < 0 || bullet.y > gameCanvas.value.height) {
      gameState.bullets.splice(index, 1)
    }
  }

  // Update asteroids
  for (const asteroid of gameState.asteroids) {
    asteroid.x += asteroid.speed * Math.cos(asteroid.angle)
    asteroid.y += asteroid.speed * Math.sin(asteroid.angle)
    wrapAround(asteroid)
  }

  // Check for ship-asteroid collisions
  for (const asteroid of gameState.asteroids) {
    const distance = Math.hypot(gameState.player.x - asteroid.x, gameState.player.y - asteroid.y)
    if (distance < asteroid.size + 12) {
      // Approximate radius of the ship
      try {
        gameOverSynth.triggerAttackRelease('G2', '8n')
      } catch {
        // console.log(e)
      }
      resetGame()
    }
  }

  // Check for bullet-asteroid collisions
  for (let index = gameState.bullets.length - 1; index >= 0; index--) {
    for (let index_ = gameState.asteroids.length - 1; index_ >= 0; index_--) {
      const bullet = gameState.bullets[index]
      const asteroid = gameState.asteroids[index_]
      const distance = Math.hypot(bullet.x - asteroid.x, bullet.y - asteroid.y)
      if (distance < asteroid.size) {
        if (asteroid.size > 10) {
          for (let k = 0; k < 3; k++) {
            createAsteroid(asteroid.x, asteroid.y, asteroid.size / 2)
          }
        }
        try {
          explosionSynth.triggerAttackRelease('C2', '8n')
        } catch {
          // console.log(e)
        }
        gameState.bullets.splice(index, 1)
        gameState.asteroids.splice(index_, 1)
        gameState.score += 10
        break
      }
    }
  }

  // Increase difficulty if all asteroids are destroyed
  if (gameState.asteroids.length === 0) {
    gameState.level = gameState.level + 1
    createInitialAsteroids()
  }
}
/**
 * Ensures that a game object wraps around the edges of the canvas.
 * When the object moves off one edge of the canvas, it reappears on the opposite edge.
 * This creates a continuous, looping space effect, commonly used in arcade-style games.
 *
 * @param {GameObject} gameObject - The game object to apply wrap-around logic to. It must have `x` and `y` properties.
 */
function wrapAround(gameObject: GameObject) {
  if (!gameCanvas.value) return

  if (gameObject.x < 0) gameObject.x = gameCanvas.value.width
  if (gameObject.x > gameCanvas.value.width) gameObject.x = 0
  if (gameObject.y < 0) gameObject.y = gameCanvas.value.height
  if (gameObject.y > gameCanvas.value.height) gameObject.y = 0
}

/**
 * Renders all game objects on the canvas. This includes drawing the player's ship,
 * bullets, and asteroids. It's called after each game state update.
 */
function draw() {
  if (!gameCanvas.value) return

  const context = gameCanvas.value.getContext('2d')

  if (!context) {
    return
  }

  context.clearRect(0, 0, gameCanvas.value.width, gameCanvas.value.height)

  // Draw player (triangle)
  context.save()
  context.translate(gameState.player.x, gameState.player.y)
  context.rotate(gameState.player.angle)
  context.beginPath()
  context.moveTo(-10, -12)
  context.lineTo(10, 0)
  context.lineTo(-10, 12)
  context.closePath()
  context.fillStyle = 'white'
  context.fill()
  context.restore()

  // Draw bullets
  for (const bullet of gameState.bullets) {
    context.beginPath()
    context.arc(bullet.x, bullet.y, 2, 0, 2 * Math.PI)
    context.fillStyle = 'white'
    context.fill()
  }

  // Draw asteroids
  for (const asteroid of gameState.asteroids) {
    context.beginPath()
    context.moveTo(asteroid.x + asteroid.vertices[0].x, asteroid.y + asteroid.vertices[0].y)
    for (const vertex of asteroid.vertices) {
      context.lineTo(asteroid.x + vertex.x, asteroid.y + vertex.y)
    }
    context.closePath()
    context.fillStyle = 'white'
    context.fill()
  }
}

/**
 * Linearly interpolates between two angles. This function is used to smoothly transition
 * the angle of an object (like the player's ship) from its current angle to a target angle.
 * The interpolation factor 't' determines the amount of change per call.
 *
 * @param {number} currentAngle - The current angle of the object.
 * @param {number} targetAngle - The desired target angle to reach.
 * @param {number} t - The interpolation factor, a value between 0 and 1.
 * @returns {number} The interpolated angle.
 */
function lerpAngle(currentAngle: number, targetAngle: number, t: number): number {
  const delta = targetAngle - currentAngle

  // Normalize the delta angle within the range [-PI, PI]
  const normalisedDelta = ((delta + Math.PI) % (2 * Math.PI)) - Math.PI
  return currentAngle + normalisedDelta * t
}

/**
 * Basic automatic control of the player's ship. This is designed to take
 * control of the ship when there is player inactivity. It steers and shoots
 * based on the positions of asteroids and other game factors.
 */
function controlShip() {
  const nearestAsteroid = findNearestAsteroid()

  if (nearestAsteroid) {
    const angleToAsteroid = Math.atan2(
      nearestAsteroid.asteroid.y - gameState.player.y,
      nearestAsteroid.asteroid.x - gameState.player.x,
    )
    const distanceToAsteroid = Math.hypot(
      nearestAsteroid.asteroid.x - gameState.player.x,
      nearestAsteroid.asteroid.y - gameState.player.y,
    )

    let targetAngle

    if (distanceToAsteroid < 150) {
      // If too close to the asteroid
      targetAngle = angleToAsteroid + Math.PI // Calculate angle to steer away
    } else {
      targetAngle = angleToAsteroid // Calculate angle to steer towards the asteroid
    }

    // Smoothly adjust the player's angle towards the target angle
    gameState.player.angle = lerpAngle(gameState.player.angle, targetAngle, 0.05)

    // Adjust speed
    gameState.player.speed = Math.min(5, distanceToAsteroid / 30)
  }
}

function maybeShoot() {
  const currentTime = Date.now()
  if (currentTime - gameState.lastAIShootTime > gameState.aiShootCooldown) {
    const nearestAsteroid = findNearestAsteroid()
    if (
      nearestAsteroid &&
      Math.hypot(nearestAsteroid.asteroid.x - gameState.player.x, nearestAsteroid.asteroid.y - gameState.player.y) < 200
    ) {
      // Shooting range
      shootBullet()
      gameState.lastAIShootTime = currentTime // Update the last shoot time
    }
  }
}

type NearestAsteroid = {
  asteroid: Astroid
  distance: number
}

/**
 * Finds the nearest asteroid to the player's ship. It iterates through all the asteroids
 * in the game state and calculates the distance to each one from the player's ship.
 * Returns the asteroid that is closest to the player.
 *
 * @returns {Astroid | null} The closest asteroid to the player, or null if there are no asteroids.
 */
function findNearestAsteroid() {
  return gameState.asteroids.reduce((nearest: NearestAsteroid | null, asteroid: Astroid) => {
    const distanceToAsteroid = Math.hypot(asteroid.x - gameState.player.x, asteroid.y - gameState.player.y)
    if (!nearest || distanceToAsteroid < nearest.distance) {
      return { asteroid, distance: distanceToAsteroid }
    }
    return nearest
  }, null)
}

/**
 * The main loop for the game, responsible for continuously updating and rendering the game state.
 * This function serves as the heartbeat of the game, repeatedly calling the update and draw functions
 * to reflect changes in the game world. It uses `requestAnimationFrame` for rendering timing.
 * The game loop also handles AI-controlled actions if the user is inactive.
 */
function gameLoop() {
  if (gameState.timeStart === 0) {
    gameState.timeStart = Date.now()
  }

  gameState.timePlaying = Date.now() - gameState.timeStart

  update()
  if (!gameState.userHasInteracted && gameState.timePlaying > 5000) {
    controlShip()
    maybeShoot()
  }
  draw()
  requestAnimationFrame(gameLoop)
}

function shootBullet() {
  gameState.bullets.push({
    angle: gameState.player.angle,
    speed: 10,
    x: gameState.player.x,
    y: gameState.player.y,
  })
  try {
    bulletSynth.triggerAttackRelease('C4', '8n')
  } catch (error) {
    console.error(error)
  }
}
</script>

<style lang="scss">
@use './Index.scss';
</style>