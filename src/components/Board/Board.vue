<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";

import { ASTEROID_IMAGES, ASTEROID_GENERATE_RATE, UPDATE_RATE, START_POINT, PLAYER_MOVEMENT_AMOUNT, CHECK_KEYS_INTERVAL, KEY_INACTIVITY_TIMEOUT } from './Board.constants' 

const position = ref(START_POINT);
const bullets = ref([]);
const asteroids = ref([]);
const isGameOver = ref(false);
const isGameStarted = ref(false);
const score = ref(0);
const lives = ref(3);
const audio = ref();
const explodingAsteroids = ref([]);

let gameLoopId;
let asteroidGenerationId;
let keyCheckIntervalId;
let keyInactivityTimeoutId;
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
  audio.value.play();

  clearInterval(asteroidGenerationId);
  asteroidGenerationId = setInterval(generateAsteroid, ASTEROID_GENERATE_RATE);

  clearInterval(gameLoopId);
  gameLoopId = setInterval(updateGame, UPDATE_RATE);

  startCheckingKeys();
};

const restartGame = () => {
  position.value = START_POINT;
  bullets.value = [];
  asteroids.value = [];
  explodingAsteroids.value = [];
  isGameOver.value = false;
  score.value = 0;
  lives.value = 3;
  audio.value.play();

  clearInterval(gameLoopId);
  clearInterval(asteroidGenerationId);
  asteroidGenerationId = setInterval(generateAsteroid, ASTEROID_GENERATE_RATE);
  gameLoopId = setInterval(updateGame, UPDATE_RATE);

  startCheckingKeys();
};

const handleKeyDown = (event) => {
  if (!pressedKeys.includes(event.key)) {
    pressedKeys.push(event.key);
    startCheckingKeys();
  }
};

const handleKeyUp = (event) => {
  pressedKeys = pressedKeys.filter((key) => key !== event.key);
  if (pressedKeys.length === 0) {
    clearInterval(keyCheckIntervalId);
    keyCheckIntervalId = null;
  }
};

const moveLeft = () => {
  movePlayerLeft(PLAYER_MOVEMENT_AMOUNT);
};

const moveRight = () => {
  movePlayerRight(PLAYER_MOVEMENT_AMOUNT);
};

const fireBullet = () => {
  if (!isGameStarted.value || isGameOver.value) {
    return;
  }
  bullets.value.push({ top: 360, left: position.value });
};

const generateAsteroid = () => {
  if (isGameStarted.value && !isGameOver.value) {
    const left = randomBetween(0, 380);
    const size = randomBetween(30, 80); // Increased size
    const image = ASTEROID_IMAGES[randomBetween(0, ASTEROID_IMAGES.length - 1)]; // Randomly select an image
    asteroids.value.push({ top: 0, left, size, image  });
  }
};

const updateGame = () => {
  if (!isGameStarted.value || isGameOver.value) {
    return;
  }

  bullets.value = bullets.value
    .map((bullet) => ({ ...bullet, top: bullet.top - 5 }))
    .filter((bullet) => bullet.top > 0);

  asteroids.value = asteroids.value
    .map((asteroid) => ({ ...asteroid, top: asteroid.top + 5 }))
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

const checkCollisions = () => {
  bullets.value.forEach((bullet, bulletIndex) => {
    asteroids.value.forEach((asteroid, asteroidIndex) => {
      if (
        bullet.top <= asteroid.top + asteroid.size &&
        bullet.top >= asteroid.top &&
        bullet.left >= asteroid.left &&
        bullet.left <= asteroid.left + asteroid.size
      ) {
        bullets.value.splice(bulletIndex, 1);
        score.value += Math.ceil(asteroid.size / 10);
        asteroid.exploding = true;
        explodingAsteroids.value.push(asteroid);
        setTimeout(() => {
          asteroids.value.splice(asteroidIndex, 1);
          explodingAsteroids.value = explodingAsteroids.value.filter(
            (a) => a !== asteroid
          );
        }, 500); // Match the duration of the explosion animation
      }
    });
  });
};

const startCheckingKeys = () => {
  clearInterval(keyCheckIntervalId);
  keyCheckIntervalId = setInterval(checkPressedKeys, CHECK_KEYS_INTERVAL);

  clearTimeout(keyInactivityTimeoutId);
  keyInactivityTimeoutId = setTimeout(() => {
    clearInterval(keyCheckIntervalId);
    keyCheckIntervalId = null;
  }, KEY_INACTIVITY_TIMEOUT);
};

const checkPressedKeys = () => {
  if (!isGameStarted.value || isGameOver.value) {
    return;
  }

  pressedKeys.forEach((key) => {
    switch (key) {
      case "ArrowLeft":
        movePlayerLeft(PLAYER_MOVEMENT_AMOUNT);
        break;
      case "ArrowRight":
        movePlayerRight(PLAYER_MOVEMENT_AMOUNT);
        break;
      case " ":
        fireBullet();
        break;
      default:
        break;
    }
  });

  clearTimeout(keyInactivityTimeoutId);
  keyInactivityTimeoutId = setTimeout(() => {
    clearInterval(keyCheckIntervalId);
    keyCheckIntervalId = null;
  }, KEY_INACTIVITY_TIMEOUT);
};

onMounted(() => {
  window.addEventListener("keydown", handleKeyDown);
  window.addEventListener("keyup", handleKeyUp);
});

onBeforeUnmount(() => {
  window.removeEventListener("keydown", handleKeyDown);
  window.removeEventListener("keyup", handleKeyUp);
  clearInterval(gameLoopId);
  clearInterval(asteroidGenerationId);
  clearInterval(keyCheckIntervalId);
  clearTimeout(keyInactivityTimeoutId);
});
</script>
<template>
  <div class="wrapper">
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
          backgroundImage: `url(${asteroid.image})`,
        }"
      >
      </div>
      <div
        v-for="(asteroid, index) in explodingAsteroids"
        :key="'exploding-asteroid-' + index"
        class="asteroid explosion"
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
    <audio hidden ref="audio">
      <source src="../../assets/music/game_theme.mp3" type="audio/mpeg" />
    </audio>
    <div class="score-lives">
      <p>Score: {{ score }}</p>
      <p>Lives: {{ lives }}</p>
    </div>
  </div>
</template>

<style scoped lang="scss">
@keyframes explode {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  100% {
    transform: scale(2);
    opacity: 0;
  }
}

.explosion {
  animation: explode 0.5s forwards;
}

.wrapper {
  display: flex;
  justify-content: center;
  flex-direction: column;
  height: 100vh;
  align-items: center;
}

.main-play-area {
  width: 400px;
  height: 400px;
  position: relative;
  overflow: hidden;
  background-image: url("../../assets/images/Background.png");
  background-size: cover;

  .player-control-shooter {
    width: 40px;
    height: 40px;
    position: absolute;
    bottom: 4px;
    transform: translateX(-50%);
    transition: transform 0.3s ease;
    background-image: url("../../assets/images/Assaulter.png");
    background-size: cover;
  }

  .bullet {
    width: 5px;
    height: 10px;
    position: absolute;
    background-image: url("../../assets/images/BasicProjectile.png");
    background-size: contain;
  }

  .asteroid {
    position: absolute;
    background-repeat: repeat;
    background-size: cover;
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
}

  .score-lives {
    top: 10px;
    right: 10px;
    font-size: 16px;
    padding: 16px;
    p {
      margin: 5px 0;
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
