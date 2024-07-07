<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';

// Constants for game parameters
const ASTEROID_GENERATE_RATE = 1000; // Rate at which asteroids are generated (in milliseconds)
const UPDATE_RATE = 200; // Update rate for game loop (in milliseconds)
const START_POINT = 150; // Initial position of the player
const PLAYER_MOVEMENT_AMOUNT = 15


const position = ref(START_POINT); // Current position of the player
const bullets = ref([]); // Array to store bullets fired by the player
const asteroids = ref([]); // Array to store asteroids on the screen
const isGameOver = ref(false); // Flag to indicate if the game is over
const isGameStarted = ref(false); // Flag to indicate if the game has started
const score = ref(0); // Variable to store the player's score
const lives = ref(3); // Number of lives for the player
const audio = ref(); // Reference to game background music
const effectsAudio = ref(); // Reference to sound effects

let gameLoopId; // ID for the game loop interval
let asteroidGenerationId; // ID for the asteroid generation interval

// Function to move the player left by a specified amount of pixels
const movePlayerLeft = (amount) => {
  if (position.value > 0) {
    position.value -= amount;
  }
};

// Function to move the player right by a specified amount of pixels
const movePlayerRight = (amount) => {
  if (position.value < 380) {
    position.value += amount;
  }
};

// Function to generate a random number between min and max (inclusive)
const randomBetween = (min, max) => {
  return Math.floor(Math.random() * (max - min + 1)) + min;
};

// Function to start the game
const startGame = () => {
  isGameStarted.value = true; // Set game as started
  isGameOver.value = false; // Reset game over flag
  audio.value?.play(); // Start background music (if available)

  // Clear previous asteroid generation interval if exists
  if (asteroidGenerationId) {
    clearInterval(asteroidGenerationId);
  }

  // Start asteroid generation with setInterval
  asteroidGenerationId = setInterval(generateAsteroid, ASTEROID_GENERATE_RATE);

  // Start game loop with setInterval
  gameLoopId = setInterval(updateGame, UPDATE_RATE);
};

// Function to restart the game
const restartGame = () => {
  // Reset game variables
  position.value = START_POINT; // Reset player position
  bullets.value = []; // Clear bullets array
  asteroids.value = []; // Clear asteroids array
  isGameOver.value = false; // Reset game over flag
  score.value = 0; // Reset player's score
  lives.value = 3; // Reset player's lives
  audio.value?.play(); // Start background music (if available)

  // Clear previous intervals if exist
  if (gameLoopId) {
    clearInterval(gameLoopId);
  }
  if (asteroidGenerationId) {
    clearInterval(asteroidGenerationId);
  }

  // Start asteroid generation with setInterval
  asteroidGenerationId = setInterval(generateAsteroid, ASTEROID_GENERATE_RATE);

  // Start game loop with setInterval
  gameLoopId = setInterval(updateGame, UPDATE_RATE);
};

// Function to handle keydown events for player movement and firing bullets
const handleKeyDown = (event) => {
  if (event.keyCode === 32) {
    // Spacebar key
    fireBullet(); // Fire bullet
  } else if (event.keyCode === 37) {
    // Left arrow key
    movePlayerLeft(PLAYER_MOVEMENT_AMOUNT); // Move player left by 10 pixels
  } else if (event.keyCode === 39) {
    // Right arrow key
    movePlayerRight(PLAYER_MOVEMENT_AMOUNT); // Move player right by 10 pixels
  }
};

// Lifecycle hook to add event listener for keydown events when component is mounted
onMounted(() => {
  window.addEventListener("keydown", handleKeyDown);
});

// Lifecycle hook to remove event listener and clear intervals when component is about to be unmounted
onBeforeUnmount(() => {
  window.removeEventListener("keydown", handleKeyDown);
  clearInterval(gameLoopId); // Clear game loop interval
  clearInterval(asteroidGenerationId); // Clear asteroid generation interval
});

// Function to move the player left by 10 pixels (used by UI buttons)
const moveLeft = () => {
  movePlayerLeft(10);
};

// Function to move the player right by 10 pixels (used by UI buttons)
const moveRight = () => {
  movePlayerRight(10);
};

// Function to fire a bullet from the player's current position (used by UI button)
const fireBullet = () => {
  bullets.value.push({ top: 360, left: position.value }); // Add bullet to bullets array
  // bullets.value.push({ top: 360, left: position.value + 10 }); // Add bullet to bullets array
};

// Function to generate asteroids at the top of the screen
const generateAsteroid = () => {
  if (isGameStarted.value && !isGameOver.value) {
    const left = randomBetween(0, 380); // Random horizontal position within game area
    const size = randomBetween(15, 40); // Random size between 10 and 40 pixels
    asteroids.value.push({ top: 0, left, size }); // Add new asteroid to asteroids array
  }
};

// Function to update the position of bullets and asteroids in the game loop
const updateGame = () => {
  if (!isGameStarted.value || isGameOver.value) {
    return; // Exit function if game is not started or game over
  }

  // Update bullets position
  bullets.value = bullets.value
    .map((bullet) => {
      return { ...bullet, top: bullet.top - 5 }; // Move bullet up by 5 pixels in each update
    })
    .filter((bullet) => bullet.top > 0); // Remove bullets that are out of bounds

  // Update asteroids position
  asteroids.value = asteroids.value
    .map((asteroid) => {
      return { ...asteroid, top: asteroid.top + 5 }; // Move asteroid down by 5 pixels in each update
    })
    .filter((asteroid) => {
      if (asteroid.top >= 380) {
        lives.value--; // Decrement player lives if asteroid reaches bottom of screen
        if (lives.value === 0) {
          isGameOver.value = true; // Set game over if no lives left
        }
        return false; // Remove asteroid from array
      }
      return asteroid.top < 400; // Keep asteroids within visible area
    });

  checkPlayerCollision(); // Check for collisions between player and asteroids
  checkCollisions(); // Check for collisions between bullets and asteroids
};

// Function to check collisions between player and asteroids
const checkPlayerCollision = () => {
  asteroids.value.forEach((asteroid) => {
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
        effectsAudio.value?.play(); // Play sound effect for asteroid destruction
        bullets.value.splice(bulletIndex, 1); // Remove bullet from bullets array
        score.value += Math.ceil(asteroid.size / 10); // Increase score based on asteroid size
        asteroids.value.splice(asteroidIndex, 1); // Remove asteroid from asteroids array
      }
    });
  });
};
</script>

<template>
  <div>
    <!-- Game area and UI elements -->
    <div class="main-play-area">
      <!-- Player's spaceship -->
      <div
        v-if="isGameStarted && !isGameOver"
        class="player-control-shooter"
        :style="{ left: `${position}px` }"
      ></div>
      <!-- Bullets fired by the player -->
      <div
        v-for="(bullet, index) in bullets"
        :key="'bullet-' + index"
        class="bullet"
        :style="{ top: bullet.top + 'px', left: bullet.left + 'px' }"
      ></div>
      <!-- Asteroids on the screen -->
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
      <!-- Game over message -->
      <div v-if="isGameOver" class="game-over-message">Game Over</div>
      <!-- Restart game button -->
      <button
        v-if="isGameOver"
        @click="restartGame"
        class="restart-button"
      >
        Restart Game
      </button>
      <!-- Start game button -->
      <button
        v-if="!isGameStarted"
        @click="startGame"
        class="start-button"
      >
        Start Game
      </button>
    </div>
    <!-- Audio elements for game background music and sound effects -->
    <audio hidden ref="audio" autoplay loop>
      <source src="../assets/music/game_theme.mp3" type="audio/mpeg" />
    </audio>
    <audio hidden ref="effectsAudio" autoplay loop>
      <source src="../assets/music/explosion.wav" type="audio/mpeg" />
    </audio>
    <!-- Controls for player movement and firing bullets -->
    <div class="controls" v-if="isGameStarted && !isGameOver">
      <button @click="moveLeft" :disabled="isGameOver">Left</button>
      <button @click="moveRight" :disabled="isGameOver">Right</button>
      <button @click="fireBullet" :disabled="isGameOver">Fire</button>
    </div>
    <!-- Display score and remaining lives during gameplay -->
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
width: 0;
  height: 0;
  border-left: 10px solid transparent;
  border-right: 10px solid transparent;
  border-bottom: 20px solid white;
  position: absolute;
  bottom: 0;
  transform: translateX(-50%);
  transition:
    transform 0.3s ease; /* Transition only transform property */
  }

  /* Styling for bullets fired by the player */
  .bullet {
    width: 5px;
    height: 10px;
    background-color: red;
    position: absolute;
  }

  /* Styling for asteroids on the screen */
  .asteroid {
    background-color: gray;
    position: absolute;
  }

  /* Styling for game over message */
  .game-over-message {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 24px;
    color: white;
    font-weight: bold;
  }

  /* Styling for restart game button */
  .restart-button,
  .start-button {
    position: absolute;
    top: 60%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 18px;
    padding: 10px 20px;
    background-color: green;
    color: white;
    border: none;
    cursor: pointer;
  }

  /* Styling for score and lives display */
  .score-lives {
    position: absolute;
    top: 10px;
    right: 10px;
    color: white;
    font-size: 16px;

    p{
      margin: 5px 0;
    }
  }
}

/* Styling for player controls */
.controls {
  margin-top: 10px;
  display: flex;
  justify-content: center;

  button {
    margin: 0 5px;
  }
}
</style>
