<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';

// Constants for game parameters
const ASTEROID_GENERATE_RATE = 1000;
const UPDATE_RATE = 200;
const START_POINT = 150;
const PLAYER_MOVEMENT_AMOUNT = 15;
const CHECK_KEYS_INTERVAL = 50; // Interval to check pressed keys (in milliseconds)

const position = ref(START_POINT);
const bullets = ref([]);
const asteroids = ref([]);
const isGameOver = ref(false);
const isGameStarted = ref(false);
const score = ref(0);
const lives = ref(3);
const audio = ref();
const effectsAudio = ref();

let gameLoopId;
let asteroidGenerationId;

// Array to store pressed keys
let pressedKeys = [];

const movePlayerLeft = (amount) => {
  if (position.value > 0) {
    position.value -= amount;
  }
};

const movePlayerRight = (amount) => {
  if (position.value < 380) {
    position.value += amount;
  }
};

const randomBetween = (min, max) => {
  return Math.floor(Math.random() * (max - min + 1)) + min;
};

const startGame = () => {
  isGameStarted.value = true;
  isGameOver.value = false;
  audio.value?.play();

  if (asteroidGenerationId) {
    clearInterval(asteroidGenerationId);
  }
  asteroidGenerationId = setInterval(generateAsteroid, ASTEROID_GENERATE_RATE);

  gameLoopId = setInterval(updateGame, UPDATE_RATE);

  // Start checking pressed keys
  startCheckingKeys();
};

const restartGame = () => {
  position.value = START_POINT;
  bullets.value = [];
  asteroids.value = [];
  isGameOver.value = false;
  score.value = 0;
  lives.value = 3;
  audio.value?.play();

  if (gameLoopId) {
    clearInterval(gameLoopId);
  }
  if (asteroidGenerationId) {
    clearInterval(asteroidGenerationId);
  }
  asteroidGenerationId = setInterval(generateAsteroid, ASTEROID_GENERATE_RATE);
  gameLoopId = setInterval(updateGame, UPDATE_RATE);

  // Start checking pressed keys
  startCheckingKeys();
};

const handleKeyDown = (event) => {
  if (!pressedKeys.includes(event.key)) {
    pressedKeys.push(event.key);
  }
};

const handleKeyUp = (event) => {
  pressedKeys = pressedKeys.filter((key) => key !== event.key);
};

const moveLeft = () => {
  movePlayerLeft(10);
};

const moveRight = () => {
  movePlayerRight(10);
};

const fireBullet = () => {
  bullets.value.push({ top: 360, left: position.value });
};

const generateAsteroid = () => {
  if (isGameStarted.value && !isGameOver.value) {
    const left = randomBetween(0, 380);
    const size = randomBetween(15, 40);
    asteroids.value.push({ top: 0, left, size });
  }
};

const updateGame = () => {
  if (!isGameStarted.value || isGameOver.value) {
    return;
  }

  bullets.value = bullets.value
    .map((bullet) => {
      return { ...bullet, top: bullet.top - 5 };
    })
    .filter((bullet) => bullet.top > 0);

  asteroids.value = asteroids.value
    .map((asteroid) => {
      return { ...asteroid, top: asteroid.top + 5 };
    })
    .filter((asteroid) => {
      if (asteroid.top >= 380) {
        lives.value--;
        if (lives.value === 0) {
          isGameOver.value = true;
        }
        return false;
      }
      return asteroid.top < 400;
    });

  checkPlayerCollision();
  checkCollisions();
};

const checkPlayerCollision = () => {
  asteroids.value.forEach((asteroid) => {
    if (
      asteroid.top + asteroid.size >= 360 &&
      asteroid.top <= 400 &&
      asteroid.left >= position.value - asteroid.size &&
      asteroid.left <= position.value + asteroid.size
    ) {
      lives.value--;
      if (lives.value === 0) {
        isGameOver.value = true;
      } else {
        position.value = START_POINT;
      }
    }
  });
};

// Function to check collisions between bullets and asteroids
const checkCollisions = () => {
  bullets.value.forEach((bullet, bulletIndex) => {
    asteroids.value.forEach((asteroid, asteroidIndex) => {
      if (
        bullet.top <= asteroid.top + asteroid.size &&
        bullet.top >= asteroid.top &&
        bullet.left >= asteroid.left &&
        bullet.left <= asteroid.left + asteroid.size
      ) {
        // effectsAudio.value?.play();
        bullets.value.splice(bulletIndex, 1);
        score.value += Math.ceil(asteroid.size / 10);
        asteroids.value.splice(asteroidIndex, 1);
      }
    });
  });
};

const startCheckingKeys = () => {
  setInterval(checkPressedKeys, CHECK_KEYS_INTERVAL);
};

const checkPressedKeys = () => {
  pressedKeys.forEach((key) => {
    switch (key) {
      case 'ArrowLeft':
        movePlayerLeft(PLAYER_MOVEMENT_AMOUNT);
        break;
      case 'ArrowRight':
        movePlayerRight(PLAYER_MOVEMENT_AMOUNT);
        break;
      case ' ':
        fireBullet();
        break;
      default:
        break;
    }
  });
};

onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
  window.addEventListener('keyup', handleKeyUp);
});

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeyDown);
  window.removeEventListener('keyup', handleKeyUp);
  clearInterval(gameLoopId);
  clearInterval(asteroidGenerationId);
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
    <audio hidden ref="audio" autoplay loop>
      <source src="../assets/music/game_theme.mp3" type="audio/mpeg" />
    </audio>
    <audio hidden ref="effectsAudio" autoplay loop>
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
    width: 0;
    height: 0;
    border-left: 10px solid transparent;
    border-right: 10px solid transparent;
    border-bottom: 20px solid white;
    position: absolute;
    bottom: 0;
    transform: translateX(-50%);
    transition: transform 0.3s ease; /* Transition only transform property */
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
