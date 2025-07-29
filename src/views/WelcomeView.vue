<template>
  <div class="welcome-container">
    <h1 class="welcome-title">Memory Game</h1>
    
    <div class="control-panel">
    <div class="mb-3 sm:mb-4">
      <label class="block mb-1 sm:mb-2 font-bold text-gray-700 text-sm sm:text-base">
        Poziom trudności:
      </label>
      <select
        v-model="gameConfig.difficulty"
        class="w-full p-2 rounded border border-gray-300 text-sm"
      >
        <option v-for="(diff, key) in difficulties" :key="key" :value="key">
          {{ diff.name }} ({{ diff.grid }}x{{ diff.grid }})
        </option>
      </select>
    </div>
    <div class="mb-3 sm:mb-4">
      <p class="block mb-1 sm:mb-2 font-bold text-gray-700 text-sm sm:text-base">
        Seed (opcjonalny):
      </p>
      <div class="flex gap-2">
        <input
          id="seedInput"
          type="text"
          v-model="gameConfig.seed"
          placeholder="Wprowadź seed lub pozostaw puste"
          class="flex-1 p-2 rounded border border-gray-300 text-sm w-40"
        />
        <button
          @click="generateRandomSeed"
          class="w-20 px-3 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 text-sm"
        >
          Losowy
        </button>
      </div>
    </div>
    <button
      @click="startNewGame"
      class="w-full p-3 bg-green-500 text-white rounded font-bold text-sm hover:bg-green-600 mb-2"
    >
      Rozpocznij nową Grę
    </button>
    <button
      @click="goToHistory"
      class="w-full p-3 bg-purple-500 text-white rounded font-bold text-sm hover:bg-purple-600"
    >
      Historia Rozgrywek
    </button>
  </div>

  <div class="text-center mb-3 sm:mb-5 text-xs sm:text-sm text-gray-600">
    <div class="grid grid-cols-1 sm:grid-cols-3 gap-1 sm:gap-2">
      <p>Aktualny seed: {{ gameConfig.seed || 'Losowy' }}</p>
      <p>Poziom: {{ difficulties[gameConfig.difficulty].name }}</p>
      <p>Kart: {{ difficulties[gameConfig.difficulty].pairs }} par</p>
    </div>
  </div>

  <div v-if="hasActiveGame" class="text-center mb-3 sm:mb-5 p-3 sm:p-4 bg-blue-100 border border-blue-400 rounded-lg">
    <h2 class="text-base sm:text-lg font-bold text-blue-800 mb-2">Znaleziono zapisaną grę!</h2>
    <p class="text-blue-700 mb-3 text-sm sm:text-base">Masz niedokończoną grę. Czy chcesz ją kontynuować?</p>
    <div class="flex flex-col sm:flex-row gap-2 justify-center">
      <button
        @click="continueSavedGame"
        class="px-3 sm:px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 text-sm sm:text-base"
      >
        Kontynuuj Grę
      </button>
      <button
        @click="clearSavedGame"
        class="px-3 sm:px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600 text-sm sm:text-base"
      >
        Przerwij aktualną grę
      </button>
    </div>
  </div>
  </div>
</template>

<style scoped lang="scss">
.w-20 {
  width: 20% !important;
}

.w-40 {
  width: 40% !important;
}

.welcome {
  &-container {
    background: rgba(255, 255, 255, 0.95);
    border-radius: 16px;
    backdrop-filter: blur(15px);
    box-shadow: 0 12px 40px rgba(0, 0, 0, 0.3);
    padding: 2rem;
    max-width: 500px;
    width: 100%;
    margin: 0 auto;
  }

  &-title {
    font-family: 'CS-Regular', 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    font-size: 2.5rem;
    font-weight: bold;
    color: #2c3e50;
    text-align: center;
    margin-bottom: 2rem;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    letter-spacing: 0.05em;
  }
}

.control-panel {
  background: rgba(243, 244, 246, 0.8);
  padding: 1.5rem;
  border-radius: 12px;
  margin-bottom: 1.5rem;
  border: 1px solid rgba(209, 213, 219, 0.5);

  & label {
    display: block;
    margin-bottom: 0.5rem;
    font-weight: bold;
    color: #374151;
    font-size: 0.875rem;
  }

  & input, select {
    width: 100%;
    padding: 0.75rem;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    font-size: 0.875rem;
    background: white;
    transition: border-color 0.2s;

    &:focus {
      outline: none;
      border-color: #3b82f6;
      box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }
  }

  & button {
    cursor: pointer;
    width: 100%;
    padding: 0.75rem;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    font-size: 0.875rem;
    transition: all 0.2s;

    &:hover {
      transform: translateY(-1px);
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    }

    &:active {
      transform: translateY(0);
    }
  }
}

@media (max-width: 640px) {
  .welcome {
    &-container {
      padding: 1rem;
      margin: 0 0.5rem;
    }

    &-title {
      font-size: 2rem;
    }
  }

  .control-panel {
    padding: 1rem;
  }
}
</style>

<script setup>
import { ref, reactive, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

const difficulties = {
  easy: { grid: 4, pairs: 8, name: 'Łatwy' },
  medium: { grid: 6, pairs: 18, name: 'Średni' },
  hard: { grid: 8, pairs: 32, name: 'Trudny' }
}

const gameConfig = reactive({
  difficulty: 'easy',
  seed: ''
})

const hasActiveGame = ref(false)

// Generuj losowy seed
const generateRandomSeed = () => {
  gameConfig.seed = Math.floor(Math.random() * 1000000).toString()
}

// Rozpocznij nową grę
const startNewGame = () => {
  // Zapisz konfigurację gry w localStorage
  localStorage.setItem('memoryGame_config', JSON.stringify({
    difficulty: gameConfig.difficulty,
    seed: gameConfig.seed || Date.now().toString(),
    isNewGame: true
  }))
  
  // Przejdź do widoku gry
  router.push('/game')
}

// Kontynuuj zapisaną grę
const continueSavedGame = () => {
  router.push('/game')
}

// Przejdź do historii
const goToHistory = () => {
  router.push('/history')
}

// Wyczyść zapisaną grę
const clearSavedGame = () => {
  localStorage.removeItem('memoryGame_activeGame')
  localStorage.removeItem('memoryGame_gameState')
  hasActiveGame.value = false
  // alert('Zapisana gra została usunięta.')
}

// Sprawdź czy istnieje aktywna gra
const checkForActiveGame = () => {
  const activeGame = localStorage.getItem('memoryGame_activeGame')
  const gameData = localStorage.getItem('memoryGame_gameState')
  
  if (activeGame && gameData) {
    try {
      const savedState = JSON.parse(gameData)
      const hoursSinceSave = (Date.now() - savedState.lastSaveTime) / (1000 * 60 * 60)
      
      if (hoursSinceSave <= 24 && savedState.isGameStarted && !savedState.isGameComplete) {
        hasActiveGame.value = true
      } else {
        // Gra jest za stara lub zakończona, wyczyść ją
        clearSavedGame()
      }
    } catch {
      clearSavedGame()
    }
  }
}

onMounted(() => {
  // Generuj losowy seed na start
  generateRandomSeed()
  
  // Sprawdź czy istnieje aktywna gra
  checkForActiveGame()
})
</script> 