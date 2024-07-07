<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";

const ASTEROID_GENERATE_RATE = 1000;
const UPDATE_RATE = 200; // Update rate for bullets and asteroids in milliseconds
const START_POINT = 150; // Initial position of the player
const position = ref(START_POINT); // Current position of the player
const bullets = ref([]); // Array to store bullets
const asteroids = ref([]); // Array to store asteroids
const isGameOver = ref(false); // Game over flag
const isGameStarted = ref(false); // Game start flag
const score = ref(0); // Variable to store the score
const lives = ref(3); // Number of lives for the player
const audio = ref();
const effectsAudio = ref()

let updateGameInterval;
let generateAsteroidInterval;

// Function to move the player left by 10 pixels
const moveLeft = () => {
  if (position.value > 0) {
    position.value -= 10;
  }
};

// Function to move the player right by 10 pixels
const moveRight = () => {
  if (position.value < 380) {
    position.value += 10;
  }
};

// Function to fire a bullet from the current player position
const fireBullet = () => {
  bullets.value.push({ top: 360, left: position.value + 10 }); // Center bullet relative to player
};

// Function to generate a random number between min and max
const randomBetween = (min, max) => {
  return Math.floor(Math.random() * (max - min + 1)) + min;
};

// Function to generate asteroids at the top of the screen
const generateAsteroid = () => {
  if (!isGameStarted.value || isGameOver.value) {
    return;
  }
  const left = randomBetween(0, 380); // Random position along the width of the screen
  const size = randomBetween(10, 40); // Random size between 10 and 40 pixels
  asteroids.value.push({ top: 0, left, size }); // Start asteroid at top with random horizontal position and size
};

// Event handler for keydown events to handle player movement and firing bullets
const handleKeyDown = (event) => {
  if (isGameOver.value) return;
  if (event.keyCode === 37) {
    // Left arrow key
    moveLeft();
  } else if (event.keyCode === 39) {
    // Right arrow key
    moveRight();
  } else if (event.keyCode === 32) {
    // Space bar key
    fireBullet();
  }
};

// Function to update the position of bullets and asteroids
const updateGame = () => {
  if (!isGameStarted.value || isGameOver.value) {
    return;
  }

  // Update bullets position
  bullets.value = bullets.value
    .map((bullet) => {
      return { ...bullet, top: bullet.top - 5 }; // Move bullet up by 5 pixels
    })
    .filter((bullet) => bullet.top > 0); // Remove bullets that go out of bounds

  // Update asteroids position
  asteroids.value = asteroids.value
    .map((asteroid) => {
      return { ...asteroid, top: asteroid.top + 5 }; // Move asteroid down by 5 pixels
    })
    .filter((asteroid) => {
      if (asteroid.top >= 380) {
        lives.value--; // Decrement lives if asteroid reaches bottom
        if (lives.value === 0) {
          isGameOver.value = true; // Set game over if no lives left
        }
        return false;
      }
      return asteroid.top < 400; // Remove asteroids that go out of bounds
    });

  // Check for collisions between player and asteroids
  checkPlayerCollision();
  // Check for collisions between bullets and asteroids
  checkCollisions();
};

// Function to check collisions between player and asteroids
const checkPlayerCollision = () => {
  asteroids.value.forEach((asteroid, asteroidIndex) => {
    if (
      asteroid.top + asteroid.size >= 360 && // Asteroid hits bottom of player
      asteroid.top <= 400 && // Asteroid is within player height
      asteroid.left >= position.value - asteroid.size && // Asteroid hits left side of player
      asteroid.left <= position.value + asteroid.size // Asteroid hits right side of player
    ) {
      lives.value--; // Decrement lives
      if (lives.value === 0) {
        isGameOver.value = true; // Set game over if no lives left
      } else {
        position.value = START_POINT; // Reset player position
      }
    }
  });
};

// Function to check collisions between bullets and asteroids
const checkCollisions = () => {
  bullets.value.forEach((bullet, bulletIndex) => {
    asteroids.value.forEach((asteroid, asteroidIndex) => {
      if (
        bullet.top <= asteroid.top + asteroid.size && // Bullet hits top of asteroid
        bullet.top >= asteroid.top && // Bullet hits bottom of asteroid
        bullet.left >= asteroid.left && // Bullet is to the right of the left edge of asteroid
        bullet.left <= asteroid.left + asteroid.size // Bullet is to the left of the right edge of asteroid
      ) {
        // Remove the asteroid and the bullet upon collision
        effectsAudio.value?.play();
        bullets.value.splice(bulletIndex, 1);
        score.value += Math.ceil(asteroid.size / 10); // Increase score based on asteroid size (rounded up)
        asteroids.value.splice(asteroidIndex, 1);
      }
    });
  });
};

// Function to start the game
const startGame = () => {
  isGameStarted.value = true;
  isGameOver.value = false;
  audio.value?.play();

  // Start intervals
  updateGameInterval = setInterval(updateGame, UPDATE_RATE);
  generateAsteroidInterval = setInterval(
    generateAsteroid,
    ASTEROID_GENERATE_RATE
  );
};

// Function to restart the game
const restartGame = () => {
  // Reset game variables
  position.value = START_POINT;
  bullets.value = [];
  asteroids.value = [];
  isGameOver.value = false;
  score.value = 0; // Reset score of asteroids destroyed
  lives.value = 3; // Reset lives
  audio.value?.play();

  // Restart intervals
  updateGameInterval = setInterval(updateGame, UPDATE_RATE);
  generateAsteroidInterval = setInterval(
    generateAsteroid,
    ASTEROID_GENERATE_RATE
  );
};

// Lifecycle hook to add event listener for keydown events when component is mounted
onMounted(() => {
  window.addEventListener("keydown", handleKeyDown);
});

// Lifecycle hook to remove event listener and clear intervals when component is about to be unmounted
onBeforeUnmount(() => {
  window.removeEventListener("keydown", handleKeyDown);
  clearInterval(updateGameInterval);
  clearInterval(generateAsteroidInterval);
});
</script>

<template>
  <div>
    <div class="main-play-area">
      <div
        v-if="isGameStarted && !isGameOver"
        class="player-control-shooter"
        :style="{ left: `${position}px` }"
      ></div>
      <div
        v-for="(bullet, index) in bullets"
        :key="'bullet-' + index"
        class="bullet"
        :style="{ top: bullet.top + 'px', left: bullet.left + 'px' }"
      ></div>
      <div
        v-for="(asteroid, index) in asteroids"
        :key="'asteroid-' + index"
        class="asteroid"
        :style="{
          top: asteroid.top + 'px',
          left: asteroid.left + 'px',
          width: asteroid.size + 'px',
          height: asteroid.size + 'px',
        }"
      ></div>
      <div v-if="isGameOver" class="game-over-message">Game Over</div>
      <button v-if="isGameOver" @click="restartGame" class="restart-button">
        Restart Game
      </button>
      <button v-if="!isGameStarted" @click="startGame" class="start-button">
        Start Game
      </button>
    </div>
    <audio hidden="true" ref="audio" autoplay="true" loop="true">
      <source src="../assets/music/game_theme.mp3" type="audio/mpeg" />
    </audio>
    <audio hidden="true" ref="effectsAudio" autoplay="true" loop="true">
      <source src="../assets/music/explosion.wav" type="audio/mpeg" />
    </audio>
    <div class="controls" v-if="isGameStarted && !isGameOver">
      <button @click="moveLeft" :disabled="isGameOver">Left</button>
      <button @click="moveRight" :disabled="isGameOver">Right</button>
      <button @click="fireBullet" :disabled="isGameOver">Fire</button>
    </div>
    <div class="score-lives" v-if="isGameStarted">
      <p>Score: {{ score }}</p>
      <p>Lives: {{ lives }}</p>
    </div>
  </div>
</template>

<style scoped lang="scss">
.main-play-area {
  background-color: black;
  width: 400px;
  height: 400px;
  position: relative;
  overflow: hidden;

  .player-control-shooter {
    width: 20px;
    height: 40px;
    background: url("path/to/your/spaceship-image.png") no-repeat center center; /* Add your spaceship image */
    background-size: cover;
    position: absolute;
    bottom: 0;
  }

  .bullet {
    width: 5px;
    height: 10px;
    background-color: red;
    position: absolute;
  }

  .asteroid {
    background-color: gray;
    position: absolute;
  }

  .game-over-message {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 24px;
    color: white;
    font-weight: bold;
  }

  .restart-button,
  .start-button {
    position: absolute;
    top: 60%; /* Adjust vertical position as needed */
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 18px;
    padding: 10px 20px;
    background-color: green;
    color: white;
    border: none;
    cursor: pointer;
  }

  .score-lives {
    position: absolute;
    top: 10px;
    right: 10px;
    color: white;
    font-size: 16px;

    p {
      margin: 5px 0;
    }
  }
}

.controls {
  margin-top: 10px;
  display: flex;
  justify-content: center;

  button {
    margin: 0 5px;
  }
}
</style>
 