<script setup lang="ts">
const state = reactive({
  isGameRunning: false,
  coffeeLevel: 0,
  currentVolume: 0,
  cooldownActive: false,
  cooldownSeconds: 30,
  errorMessage: '',
});

let audioContext: AudioContext | null = null;
let analyzer: AnalyserNode | null = null;
let microphone: MediaStreamAudioSourceNode | null = null;
let animationFrameId = 0;
let cooldownTimerId = 0;

const volumeHeight = computed(() =>
  Math.min(state.currentVolume * 100, 100),
);

const gameStatus = computed(() => {
  if (state.coffeeLevel >= 100)
    return 'Selamat! Kopi Penuh!';
  if (state.isGameRunning)
    return 'Berteriaklah!';
  if (state.cooldownActive)
    return 'Istirahat Sejenak...';
  return 'Coffee Shout Game';
});

const buttonText = computed(() =>
  state.cooldownActive
    ? 'Tunggu...'
    : state.isGameRunning
      ? 'Berhenti'
      : 'Mulai Game',
);

async function startGame() {
  if (state.isGameRunning)
    return stopGame();
  if (state.cooldownActive)
    return;

  state.coffeeLevel = 0;
  state.errorMessage = '';
  state.isGameRunning = true;

  if (!audioContext) {
    audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
    analyzer = audioContext.createAnalyser();
    analyzer.fftSize = 256;
  }

  try {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    microphone = audioContext.createMediaStreamSource(stream);
    microphone.connect(analyzer!);
    analyzeAudio();
  }
  catch (err) {
    console.error('Mikrofon tidak dapat diakses:', err);
    state.errorMessage = 'Mikrofon tidak dapat diakses. Pastikan Anda memberikan izin mikrofon.';
    state.isGameRunning = false;
  }
}

function analyzeAudio() {
  if (!state.isGameRunning || !analyzer)
    return;

  const data = new Uint8Array(analyzer.frequencyBinCount);
  analyzer.getByteFrequencyData(data);
  const avg = data.reduce((sum, v) => sum + v, 0) / data.length;
  state.currentVolume = Math.min(avg / 128, 1);

  state.coffeeLevel = state.currentVolume > 0.3
    ? Math.min(state.coffeeLevel + state.currentVolume * 0.6, 100)
    : Math.max(state.coffeeLevel - 0.5, 0);

  if (state.coffeeLevel >= 100)
    return handleWin();
  animationFrameId = requestAnimationFrame(analyzeAudio);
}

function handleWin() {
  stopGame();
  state.cooldownActive = true;
  state.cooldownSeconds = 30;

  cooldownTimerId = window.setInterval(() => {
    state.cooldownSeconds--;
    if (state.cooldownSeconds <= 0) {
      clearInterval(cooldownTimerId);
      state.cooldownActive = false;
      startGame();
    }
  }, 1000);
}

function stopGame() {
  state.isGameRunning = false;
  cancelAnimationFrame(animationFrameId);
  if (microphone) {
    microphone.disconnect();
    microphone = null;
  }
}

onBeforeUnmount(() => {
  stopGame();
  clearInterval(cooldownTimerId);
  if (audioContext) {
    audioContext.close();
    audioContext = null;
  }
});
</script>

<template>
  <NuxtRouteAnnouncer />

  <div class="min-h-screen bg-amber-50 flex flex-col items-center justify-center p-4">
    <h1 class="text-3xl font-bold mb-8 text-amber-800">
      {{ gameStatus }}
    </h1>

    <div v-if="state.errorMessage" class="mb-4 text-red-600 font-semibold">
      {{ state.errorMessage }}
    </div>

    <div class="relative mb-8">
      <!-- Cangkir Kopi -->
      <div class="relative w-64 h-80">
        <div class="absolute bottom-0 w-full h-64 bg-amber-200 rounded-b-3xl rounded-t-lg border-4 border-amber-700 overflow-hidden">
          <div
            class="absolute bottom-0 w-full bg-amber-800 transition-all duration-300 ease-out"
            :style="{ height: `${state.coffeeLevel}%` }"
          />
        </div>
        <div class="absolute right-[-20px] top-20 w-16 h-24 border-4 border-amber-700 rounded-r-full border-l-0" />
      </div>

      <!-- Indikator Volume -->
      <div class="absolute top-0 left-[-40px] h-64 w-8 bg-gray-200 rounded-lg overflow-hidden">
        <div
          class="absolute bottom-0 w-full bg-green-500 transition-height duration-100"
          :style="{ height: `${volumeHeight}%` }"
        />
      </div>
    </div>

    <button
      class="px-6 py-3 bg-amber-600 text-white rounded-lg text-lg font-bold hover:bg-amber-700 disabled:opacity-50 disabled:cursor-not-allowed"
      :disabled="state.isGameRunning || state.cooldownActive"
      @click="startGame"
    >
      {{ buttonText }}
    </button>

    <div v-if="state.cooldownActive" class="text-xl font-bold text-amber-800 mt-4">
      Cooldown: {{ state.cooldownSeconds }}s
    </div>

    <div class="text-sm text-gray-600 mt-6 max-w-md text-center space-y-2">
      <p>Berteriaklah secara konsisten untuk mengisi cangkir kopi.</p>
      <p>Semakin keras teriakanmu, semakin cepat cangkir terisi!</p>
      <p>Jika berhenti berteriak, kopi akan turun kembali.</p>
    </div>
  </div>
</template>

<style scoped>
.transition-height {
  transition: height 0.1s linear;
}
</style>
