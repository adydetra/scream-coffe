<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';

// Constants
const CARD_VALUES = ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H'];
const GAME_TIME = 60;

// Game state
const cards = ref([]);
const flippedCards = ref([]);
const matchedPairs = ref(0);
const timeLeft = ref(0);
const gameActive = ref(false);
const timerInterval = ref(null);
const showResult = ref(false);
const isWinner = ref(false);
const canFlip = ref(true);

// Computed properties
const resultMessage = computed(() => {
  return isWinner.value
    ? `Selamat! Kamu menemukan semua pasangan dalam ${GAME_TIME - timeLeft.value} detik!`
    : 'Waktu habis! Coba lagi?';
});

// Card colors based on value
const cardColors = {
  A: 'bg-red-500',
  B: 'bg-blue-500',
  C: 'bg-green-500',
  D: 'bg-yellow-500',
  E: 'bg-purple-500',
  F: 'bg-pink-500',
  G: 'bg-teal-500',
  H: 'bg-orange-500',
};

// Methods
function getCardColor(value) {
  return cardColors[value] || 'bg-gray-500';
}

function startNewGame() {
  // Reset game state
  timeLeft.value = GAME_TIME;
  matchedPairs.value = 0;
  flippedCards.value = [];
  showResult.value = false;
  gameActive.value = true;
  canFlip.value = true;

  // Clear existing timer
  if (timerInterval.value) {
    clearInterval(timerInterval.value);
  }

  // Generate shuffled cards
  const values = [...CARD_VALUES, ...CARD_VALUES];
  shuffleArray(values);

  cards.value = values.map(value => ({
    value,
    flipped: false,
    matched: false,
  }));

  // Start timer
  timerInterval.value = setInterval(() => {
    if (timeLeft.value > 0) {
      timeLeft.value--;
    }
    else {
      endGame(false);
    }
  }, 1000);
}

function flipCard(index) {
  // Prevent flipping if game not active or card already flipped or matched
  if (!gameActive.value || !canFlip.value || cards.value[index].flipped || cards.value[index].matched) {
    return;
  }

  // Prevent flipping more than 2 cards at once
  if (flippedCards.value.length >= 2) {
    return;
  }

  // Flip the card
  cards.value[index].flipped = true;
  flippedCards.value.push(index);

  // Check for match when 2 cards are flipped
  if (flippedCards.value.length === 2) {
    const [firstIndex, secondIndex] = flippedCards.value;

    if (cards.value[firstIndex].value === cards.value[secondIndex].value) {
      // Cards match
      cards.value[firstIndex].matched = true;
      cards.value[secondIndex].matched = true;
      matchedPairs.value++;
      flippedCards.value = [];

      // Check for win condition
      if (matchedPairs.value === CARD_VALUES.length) {
        endGame(true);
      }
    }
    else {
      // Cards don't match, flip them back after a delay
      canFlip.value = false;
      setTimeout(() => {
        cards.value[firstIndex].flipped = false;
        cards.value[secondIndex].flipped = false;
        flippedCards.value = [];
        canFlip.value = true;
      }, 1000);
    }
  }
}

function endGame(win) {
  gameActive.value = false;
  clearInterval(timerInterval.value);
  isWinner.value = win;
  showResult.value = true;
}

function shuffleArray(array) {
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
}

// Lifecycle hooks
onMounted(() => {
  // Initialize game but don't auto-start
  cards.value = Array.from({ length: 16 }).fill().map(() => ({
    value: '',
    flipped: false,
    matched: false,
  }));
});

onBeforeUnmount(() => {
  if (timerInterval.value) {
    clearInterval(timerInterval.value);
  }
});
</script>

<template>
  <div class="min-h-screen bg-indigo-50 flex flex-col items-center justify-center p-4">
    <h1 class="text-3xl font-bold mb-2 text-indigo-800">
      Memory Card Game
    </h1>

    <!-- Timer -->
    <div class="mb-6 text-2xl font-bold" :class="timeLeft <= 10 ? 'text-red-600' : 'text-indigo-700'">
      Timer: {{ timeLeft }} detik
    </div>

    <!-- Score Display -->
    <div class="mb-6 text-xl text-indigo-700">
      Pasangan Ditemukan: {{ matchedPairs }} / 8
    </div>

    <!-- Game Board -->
    <div class="grid grid-cols-4 gap-4 mb-8 perspective-500">
      <div
        v-for="(card, index) in cards"
        :key="index"
        class="card-container"
        :class="{ flipped: card.flipped, matched: card.matched }"
        @click="flipCard(index)"
      >
        <div class="card">
          <div class="card-back bg-indigo-600 rounded-lg flex items-center justify-center">
            <span class="text-2xl text-white">?</span>
          </div>
          <div class="card-front rounded-lg flex items-center justify-center" :class="getCardColor(card.value)">
            <span class="text-2xl text-white">{{ card.value }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Game Control Button -->
    <button
      class="px-6 py-3 bg-indigo-600 text-white rounded-lg text-lg font-bold hover:bg-indigo-700 mb-4"
      :disabled="gameActive"
      :class="{ 'opacity-50 cursor-not-allowed': gameActive }"
      @click="startNewGame"
    >
      {{ gameActive ? 'Mulai Lagi' : 'Mulai Game' }}
    </button>

    <!-- Game Results -->
    <div v-if="showResult" class="text-xl font-bold mt-4" :class="isWinner ? 'text-green-600' : 'text-red-600'">
      {{ resultMessage }}
    </div>

    <!-- Game Instructions -->
    <div class="text-sm text-gray-600 mt-6 max-w-md text-center">
      <p>Carilah pasangan kartu yang sama dengan membalikkan dua kartu secara berurutan.</p>
      <p>Temukan semua pasangan sebelum waktu habis!</p>
    </div>
  </div>
</template>

<style scoped>
.perspective-500 {
  perspective: 500px;
}

.card-container {
  width: 80px;
  height: 80px;
  cursor: pointer;
  position: relative;
  transform-style: preserve-3d;
}

.card {
  width: 100%;
  height: 100%;
  position: relative;
  transform-style: preserve-3d;
  transition: transform 0.6s;
}

.card-front, .card-back {
  width: 100%;
  height: 100%;
  position: absolute;
  backface-visibility: hidden;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.card-front {
  transform: rotateY(180deg);
}

.flipped .card {
  transform: rotateY(180deg);
}

.matched .card {
  transform: rotateY(180deg);
}

@media (max-width: 640px) {
  .card-container {
    width: 60px;
    height: 60px;
  }
}
</style>
