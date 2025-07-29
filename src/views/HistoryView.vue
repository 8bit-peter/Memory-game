<template>
  <div class="history-container">
    <div class="history-header">
      <h1 class="history-title">Historia Rozgrywek</h1>
      <button
        @click="goToWelcome"
        class="back-button"
      >
        ← Powrót do Menu
      </button>
    </div>

  <div class="bg-gray-100 p-3 sm:p-4 rounded-lg mb-4 sm:mb-6">
    <h2 class="text-base sm:text-lg font-bold text-gray-800 mb-2 sm:mb-3">Statystyki ogólne</h2>
    <div class="grid grid-cols-2 md:grid-cols-4 gap-2 sm:gap-4 text-center">
      <div class="bg-white p-2 sm:p-3 rounded">
        <div class="text-lg sm:text-2xl font-bold text-blue-600">{{ totalGames }}</div>
        <div class="text-xs sm:text-sm text-gray-600">Łącznie gier</div>
      </div>
      <div class="bg-white p-2 sm:p-3 rounded">
        <div class="text-lg sm:text-2xl font-bold text-green-600">{{ averageMoves }}</div>
        <div class="text-xs sm:text-sm text-gray-600">Średnio ruchów</div>
      </div>
      <div class="bg-white p-2 sm:p-3 rounded">
        <div class="text-lg sm:text-2xl font-bold text-purple-600">{{ averageTime }}</div>
        <div class="text-xs sm:text-sm text-gray-600">Średnio sekund</div>
      </div>
      <div class="bg-white p-2 sm:p-3 rounded">
        <div class="text-lg sm:text-2xl font-bold text-orange-600">{{ bestTime }}</div>
        <div class="text-xs sm:text-sm text-gray-600">Najlepszy czas</div>
      </div>
    </div>
  </div>

  <div class="bg-white rounded-lg shadow overflow-hidden">
    <div class="overflow-x-auto">
      <table class="w-full">
        <thead class="bg-gray-50">
          <tr>
            <th class="px-2 sm:px-4 py-2 sm:py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
              #
            </th>
            <th class="px-2 sm:px-4 py-2 sm:py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
              Data
            </th>
            <th class="px-2 sm:px-4 py-2 sm:py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
              Poziom
            </th>
            <th class="px-2 sm:px-4 py-2 sm:py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
              Ruchy
            </th>
            <th class="px-2 sm:px-4 py-2 sm:py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
              Czas
            </th>
            <th class="px-2 sm:px-4 py-2 sm:py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
              Seed
            </th>
          </tr>
        </thead>
        <tbody class="bg-white divide-y divide-gray-200">
          <tr v-for="(game, index) in sortedGames" :key="index" class="hover:bg-gray-50">
            <td class="px-2 sm:px-4 py-2 sm:py-3 whitespace-nowrap text-xs sm:text-sm font-medium text-gray-900">
              {{ totalGames - index }}
            </td>
            <td class="px-2 sm:px-4 py-2 sm:py-3 whitespace-nowrap text-xs sm:text-sm text-gray-500">
              {{ formatDate(game.completedAt) }}
            </td>
            <td class="px-2 sm:px-4 py-2 sm:py-3 whitespace-nowrap">
              <span class="inline-flex px-1 sm:px-2 py-1 text-xs font-semibold rounded-full"
                    :class="getDifficultyClass(game.difficulty)">
                {{ getDifficultyName(game.difficulty) }}
              </span>
            </td>
            <td class="px-2 sm:px-4 py-2 sm:py-3 whitespace-nowrap text-xs sm:text-sm text-gray-900">
              {{ game.moves }}
            </td>
            <td class="px-2 sm:px-4 py-2 sm:py-3 whitespace-nowrap text-xs sm:text-sm text-gray-900">
              {{ game.gameTime }}s
            </td>
            <td class="px-2 sm:px-4 py-2 sm:py-3 whitespace-nowrap text-xs sm:text-sm text-gray-500 font-mono">
              {{ game.seed }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>

  <div v-if="games.length === 0" class="text-center py-12">
    <div class="text-gray-400 text-6xl mb-4">📊</div>
    <h3 class="text-lg font-medium text-gray-900 mb-2">Brak historii rozgrywek</h3>
    <p class="text-gray-500">Rozpocznij swoją pierwszą grę, aby zobaczyć statystyki!</p>
  </div>
  </div>
</template>

<style scoped>
.history-container {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 16px;
  backdrop-filter: blur(15px);
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.3);
  padding: 2rem;
  max-width: 1000px;
  width: 100%;
  margin: 0 auto;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.history-title {
  font-family: 'CS-Regular', 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  font-size: 2.5rem;
  font-weight: bold;
  color: #2c3e50;
  margin: 0;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.back-button {
  padding: 0.75rem 1.5rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: bold;
  font-size: 0.875rem;
  cursor: pointer;
  transition: all 0.2s;
}

.back-button:hover {
  background: #2563eb;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.back-button:active {
  transform: translateY(0);
}

@media (max-width: 640px) {
  .history-container {
    padding: 1rem;
    margin: 0 0.5rem;
  }
  
  .history-title {
    font-size: 2rem;
  }
  
  .history-header {
    flex-direction: column;
    text-align: center;
  }
}
</style>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const games = ref([])

const difficulties = {
  easy: { name: 'Łatwy', class: 'bg-green-100 text-green-800' },
  medium: { name: 'Średni', class: 'bg-yellow-100 text-yellow-800' },
  hard: { name: 'Trudny', class: 'bg-red-100 text-red-800' }
}

const loadHistory = () => {
  const stats = localStorage.getItem('memoryGame_statistics')
  games.value = stats ? JSON.parse(stats) : []
}

const sortedGames = computed(() => {
  return [...games.value].sort((a, b) => b.completedAt - a.completedAt)
})

const totalGames = computed(() => games.value.length)

const averageMoves = computed(() => {
  if (games.value.length === 0) return 0
  const total = games.value.reduce((sum, game) => sum + game.moves, 0)
  return Math.round(total / games.value.length)
})

const averageTime = computed(() => {
  if (games.value.length === 0) return 0
  const total = games.value.reduce((sum, game) => sum + game.gameTime, 0)
  return Math.round(total / games.value.length)
})

const bestTime = computed(() => {
  if (games.value.length === 0) return 0
  return Math.min(...games.value.map(game => game.gameTime))
})

const formatDate = (timestamp) => {
  const date = new Date(timestamp)
  return date.toLocaleDateString('pl-PL', {
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  })
}

const getDifficultyName = (difficulty) => {
  return difficulties[difficulty]?.name || difficulty
}

const getDifficultyClass = (difficulty) => {
  return difficulties[difficulty]?.class || 'bg-gray-100 text-gray-800'
}

const goToWelcome = () => {
  router.push('/')
}

onMounted(() => {
  loadHistory()
})
</script> 