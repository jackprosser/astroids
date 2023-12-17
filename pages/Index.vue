<template>
  <main class="game">
    <div class="game__score">Score: {{ gameState.score }}</div>
    <div v-if="gameState.score == 0" class="game__body">
      This is a simple game of asteroids experiment. Use your keyboard to move the ship and spacebar. The ship will be controlled autonomously after 5 seconds if you do nothing.
    </div>
    <canvas ref="gameCanvas"></canvas>
  </main>
</template>

<script>
import * as Tone from 'tone'
import { onMounted, ref, reactive } from 'vue'

export default {
  setup() {
    const gameCanvas = ref(null)
    const gameState = reactive({
      score: 0,
      player: {
        x: 0,
        y: 0,
        angle: 0,
        speed: 0,
      },
      bullets: [],
      asteroids: [],
      keys: {
        ArrowUp: false,
        ArrowLeft: false,
        ArrowRight: false,
        Space: false,
      },
      timeStart: 0,
      timePlaying: 0,
      lastAIShootTime: 0,
      aiShootCooldown: 200,
      userHasInteracted: false,
    })

    let bulletSynth
    let explosionSynth
    let gameOverSynth

    onMounted(() => {
      const ctx = gameCanvas.value.getContext('2d')
      gameCanvas.value.width = window.innerWidth
      gameCanvas.value.height = window.innerHeight

      // Sound effect synthesizers
      bulletSynth = new Tone.Synth().toDestination()
      explosionSynth = new Tone.MembraneSynth().toDestination()
      gameOverSynth = new Tone.MetalSynth().toDestination()

      window.addEventListener('keydown', keyDownHandler)
      window.addEventListener('keyup', keyUpHandler)

      resetGame()

      gameLoop()
    })

    function resetGame() {
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
      createInitialAsteroids(5)
    }

    function createInitialAsteroids(count) {
      for (let i = 0; i < count; i++) {
        let x, y
        do {
          x = Math.random() * gameCanvas.value.width
          y = Math.random() * gameCanvas.value.height
        } while (Math.hypot(gameState.player.x - x, gameState.player.y - y) < 100) // Avoid spawning near the ship
        createAsteroid(x, y)
      }
    }

    function createAsteroid(x, y, size) {
      if (!size) size = Math.random() * 20 + 15
      gameState.asteroids.push({
        x: x || Math.random() * gameCanvas.value.width,
        y: y || Math.random() * gameCanvas.value.height,
        angle: Math.random() * Math.PI * 2,
        size,
        speed: Math.random() * 2 + 1,
        vertices: generateAsteroidVertices(size),
      })
    }

    function generateAsteroidVertices(size) {
      const vertices = []
      for (let i = 0; i < 6; i++) {
        const angle = (i / 6) * Math.PI * 2
        const radius = size * (0.5 + Math.random() * 0.5)
        vertices.push({
          x: Math.cos(angle) * radius,
          y: Math.sin(angle) * radius,
        })
      }
      return vertices
    }

    function keyDownHandler(e) {
      gameState.userHasInteracted = true
      if (e.code === 'Space') {
        // Spacebar
        gameState.keys.Space = true
      } else if (e.code in gameState.keys) {
        gameState.keys[e.code] = true
      }
    }

    function keyUpHandler(e) {
      if (e.code === 'Space') {
        // Spacebar
        gameState.keys.Space = false
      } else if (e.code in gameState.keys) {
        gameState.keys[e.code] = false
      }
    }

    function update() {
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
      gameState.bullets.forEach((bullet, index) => {
        bullet.x += bullet.speed * Math.cos(bullet.angle)
        bullet.y += bullet.speed * Math.sin(bullet.angle)

        if (bullet.x < 0 || bullet.x > gameCanvas.value.width || bullet.y < 0 || bullet.y > gameCanvas.value.height) {
          gameState.bullets.splice(index, 1)
        }
      })

      // Update asteroids
      gameState.asteroids.forEach((asteroid) => {
        asteroid.x += asteroid.speed * Math.cos(asteroid.angle)
        asteroid.y += asteroid.speed * Math.sin(asteroid.angle)
        wrapAround(asteroid)
      })

      // Check for ship-asteroid collisions
      gameState.asteroids.forEach((asteroid) => {
        const distance = Math.hypot(gameState.player.x - asteroid.x, gameState.player.y - asteroid.y)
        if (distance < asteroid.size + 12) {
          // Approximate radius of the ship
          try {
            gameOverSynth.triggerAttackRelease('G2', '8n')
          } catch (e) {
            // console.log(e)
          }
          resetGame()
        }
      })

      // Check for bullet-asteroid collisions
      for (let i = gameState.bullets.length - 1; i >= 0; i--) {
        for (let j = gameState.asteroids.length - 1; j >= 0; j--) {
          const bullet = gameState.bullets[i]
          const asteroid = gameState.asteroids[j]
          const distance = Math.hypot(bullet.x - asteroid.x, bullet.y - asteroid.y)
          if (distance < asteroid.size) {
            if (asteroid.size > 10) {
              for (let k = 0; k < 3; k++) {
                createAsteroid(asteroid.x, asteroid.y, asteroid.size / 2)
              }
            }
            try {
              explosionSynth.triggerAttackRelease('C2', '8n')
            } catch (e) {
              // console.log(e)
            }
            gameState.bullets.splice(i, 1)
            gameState.asteroids.splice(j, 1)
            gameState.score += 10
            break
          }
        }
      }

      // Increase difficulty if all asteroids are destroyed
      if (gameState.asteroids.length === 0) {
        createInitialAsteroids(gameState.asteroids.length + 1)
      }
    }

    function wrapAround(object) {
      if (object.x < 0) object.x = gameCanvas.value.width
      if (object.x > gameCanvas.value.width) object.x = 0
      if (object.y < 0) object.y = gameCanvas.value.height
      if (object.y > gameCanvas.value.height) object.y = 0
    }

    function draw() {
      const ctx = gameCanvas.value.getContext('2d')
      ctx.clearRect(0, 0, gameCanvas.value.width, gameCanvas.value.height)

      // Draw player (triangle)
      ctx.save()
      ctx.translate(gameState.player.x, gameState.player.y)
      ctx.rotate(gameState.player.angle)
      ctx.beginPath()
      ctx.moveTo(-10, -12)
      ctx.lineTo(10, 0)
      ctx.lineTo(-10, 12)
      ctx.closePath()
      ctx.fillStyle = 'white'
      ctx.fill()
      ctx.restore()

      // Draw bullets
      gameState.bullets.forEach((bullet) => {
        ctx.beginPath()
        ctx.arc(bullet.x, bullet.y, 2, 0, 2 * Math.PI)
        ctx.fillStyle = 'white'
        ctx.fill()
      })

      // Draw asteroids
      gameState.asteroids.forEach((asteroid) => {
        ctx.beginPath()
        ctx.moveTo(asteroid.x + asteroid.vertices[0].x, asteroid.y + asteroid.vertices[0].y)
        asteroid.vertices.forEach((vertex) => {
          ctx.lineTo(asteroid.x + vertex.x, asteroid.y + vertex.y)
        })
        ctx.closePath()
        ctx.fillStyle = 'white'
        ctx.fill()
      })
    }

    function controlShip() {
      const nearestAsteroid = findNearestAsteroid()

      if (nearestAsteroid) {
        const angleToAsteroid = Math.atan2(
          nearestAsteroid.y - gameState.player.y,
          nearestAsteroid.x - gameState.player.x,
        )
        const distanceToAsteroid = Math.hypot(
          nearestAsteroid.x - gameState.player.x,
          nearestAsteroid.y - gameState.player.y,
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

    function lerpAngle(currentAngle, targetAngle, t) {
      const delta = targetAngle - currentAngle
      // Normalize the delta angle within the range [-PI, PI]
      const normalizedDelta = ((delta + Math.PI) % (2 * Math.PI)) - Math.PI
      return currentAngle + normalizedDelta * t
    }

    function maybeShoot() {
      const currentTime = Date.now()
      if (currentTime - gameState.lastAIShootTime > gameState.aiShootCooldown) {
        const nearestAsteroid = findNearestAsteroid()
        if (
          nearestAsteroid &&
          Math.hypot(nearestAsteroid.x - gameState.player.x, nearestAsteroid.y - gameState.player.y) < 200
        ) {
          // Shooting range
          shootBullet()
          gameState.lastAIShootTime = currentTime // Update the last shoot time
        }
      }
    }

    function findNearestAsteroid() {
      return gameState.asteroids.reduce((nearest, asteroid) => {
        const distanceToAsteroid = Math.hypot(asteroid.x - gameState.player.x, asteroid.y - gameState.player.y)
        if (!nearest || distanceToAsteroid < nearest.distance) {
          return { asteroid, distance: distanceToAsteroid }
        }
        return nearest
      }, null)?.asteroid
    }

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
        x: gameState.player.x,
        y: gameState.player.y,
        angle: gameState.player.angle,
        speed: 10,
      })
      try {
        bulletSynth.triggerAttackRelease('C4', '8n')
      } catch (e) {
        // console.log(e)
      }
    }

    return { gameCanvas, gameState, shootBullet }
  },
}
</script>

<style lang="scss">
@use './Index.scss';
</style>