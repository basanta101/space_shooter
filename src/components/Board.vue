<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";

const ASTEROID_GENERATE_RATE = 1000;
const UPDATE_RATE = 200;
const START_POINT = 150;
const PLAYER_MOVEMENT_AMOUNT = 15;
const CHECK_KEYS_INTERVAL = 50;
const KEY_INACTIVITY_TIMEOUT = 3000;

const position = ref(START_POINT);
const bullets = ref([]);
const asteroids = ref([]);
const isGameOver = ref(false);
const isGameStarted = ref(false);
const score = ref(0);
const lives = ref(3);
const audio = ref();
const effectsAudio = ref();
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

// TODO: make of use of this values
const asteroidShapes = [
  "M20 50 L35 20 Q50 5 65 20 L80 50 Q65 80 50 65 Q35 80 20 65 Z", // Dragon shape
  "M10 20 Q 25 0 40 20 T 70 20 Z", // Shape 1 (a simplified quadratic curve path)
  "M10 40 C 20 10, 40 10, 50 40 S 70 10, 90 40 Z", // Shape 2 (a simplified cubic curve path)
  "M10 10 L 30 30 L 50 10 L 70 30 L 90 10 Z", // Shape 3 (a simplified zig-zag path)
];

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
    asteroids.value.push({ top: 0, left, size });
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
        effectsAudio.value.play();
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
        }"
      ></div>
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
      <source src="../assets/music/game_theme.mp3" type="audio/mpeg" />
    </audio>
    <audio hidden ref="effectsAudio">
      <source src="../assets/music/explosion.wav" type="audio/mpeg" />
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
  border: 1px solid red;
  display: flex;
  justify-content: center;
  height: 100vh;
  align-items: center;
}

.main-play-area {
  background-color: black;
  width: 400px;
  height: 400px;
  position: relative;
  overflow: hidden;
  background-image: url("../assets/images/Background.png");
  background-size: cover;

  .player-control-shooter {
    width: 40px;
    height: 40px;
    // border-left: 10px solid transparent;
    // border-right: 10px solid transparent;
    // border-bottom: 20px solid white;
    position: absolute;
    bottom: 4px;
    transform: translateX(-50%);
    transition: transform 0.3s ease; /* Transition only transform property */
    background-image: url("../assets/images/Assaulter.png");
    background-size: cover;
  }

  .bullet {
    width: 5px;
    height: 10px;
    background-color: red;
    position: absolute;
    background-image: url("../assets/images/BasicProjectile.png");
    background-size: contain;
  }

  .asteroid {
    position: absolute;
    background-color: gray;
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
    // position: absolute;
    top: 10px;
    right: 10px;
    // color: white;
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
