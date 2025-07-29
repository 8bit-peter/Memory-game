<template>
  <div class="memory-game-container">
    <div class="game-header">
      <h1 class="game-title">Memory Game</h1>
      <button
        @click="goToWelcome"
        class="back-button"
      >
        ← Powrót do Menu
      </button>
    </div>

  <!-- Game info -->
  <div v-if="gameState.isGameStarted" class="text-center mb-3 sm:mb-5 text-xs sm:text-sm text-gray-600">
    <div class="grid grid-cols-2 sm:grid-cols-3 gap-1 sm:gap-2">
      <p>Poziom: {{ difficulties[gameState.difficulty].name }}</p>
      <p>Seed: {{ gameState.seed || 'Losowy' }}</p>
      <p>Kart: {{ difficulties[gameState.difficulty].pairs }} par</p>
      <p>Ruchy: {{ gameState.moves }}</p>
      <p>Czas: {{ gameState.gameTime }}s</p>
      <p v-if="gameState.isLoadedGame" class="text-blue-600 font-semibold">
        📋 Wczytano
      </p>
    </div>
  </div>
  
  <!-- New game control panel -->
  <div v-if="!gameState.isGameStarted" class="bg-gray-100 p-3 sm:p-5 rounded-lg mb-3 sm:mb-5">
    <button
      @click="startNewGame"
      class="w-full p-2 sm:p-3 bg-green-500 text-white rounded font-bold text-sm sm:text-base hover:bg-green-600"
    >
      Rozpocznij Nową Grę
    </button>
  </div>
  
  <!-- Game complete message -->
  <div v-if="gameState.isGameComplete" class="text-center mb-3 sm:mb-5 p-3 sm:p-4 bg-green-100 border border-green-400 rounded-lg">
    <h2 class="text-lg sm:text-xl font-bold text-green-800 mb-2">Gratulacje! 🎉</h2>
    <p class="text-green-700 text-sm sm:text-base">Ukończyłeś grę w {{ gameState.gameTime }} sekund</p>
    <p class="text-green-700 text-sm sm:text-base">Liczba ruchów: {{ gameState.moves }}</p>
    <button
      @click="goToWelcome"
      class="mt-3 px-3 sm:px-4 py-2 bg-green-500 text-white rounded hover:bg-green-600 text-sm sm:text-base"
    >
      Powrót do Menu
    </button>
  </div>
  
  <div class="flex justify-center">
    <canvas
      ref="canvasRef"
      @click="handleCanvasClick"
      @mousemove="handleMouseMove"
      @mouseleave="handleMouseLeave"
      @touchstart="handleTouchStart"
      @touchmove="handleTouchMove"
      @touchend="handleTouchEnd"
      class="border-2 border-gray-800 rounded-lg cursor-pointer max-w-full h-auto"
      style="touch-action: manipulation;"
    />
  </div>
      <div v-if="!gameState.isGameStarted" class="text-center mt-3 sm:mt-5 text-xs sm:text-sm text-gray-500">
      Wybierz poziom trudności i naciśnij "Start Gry"
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, watch, nextTick, onUnmounted } from 'vue'
import { useRouter } from 'vue-router'
import { Howl } from 'howler'

const router = useRouter()
const canvasRef = ref(null)
const ctx = ref(null)
const loadedImages = ref(new Map()) // Cache for loaded images

const difficulties = {
  easy: { grid: 4, pairs: 8, name: 'Łatwy' },
  medium: { grid: 6, pairs: 18, name: 'Średni' },
  hard: { grid: 8, pairs: 32, name: 'Trudny' }
}

const gameState = reactive({
  difficulty: 'easy',
  seed: '',
  cards: [],
  isGameStarted: false,
  canvasSize: 400,
  flippedCards: [],
  moves: 0,
  gameTime: 0,
  timer: null,
  isGameComplete: false,
  cardAnimations: [],
  animationSpeed: 0.15,
  isLoadedGame: false,
  hoveredCard: null,
  isProcessing: false
})

// Audio manager
const audioManager = {
  cardFlip: new Howl({
    src: ['/src/assets/cs_audio/card-flip.wav'],
    volume: 0.5
  }),
  cardMatch: new Howl({
    src: ['/src/assets/cs_audio/card-match.wav'],
    volume: 0.6
  })
}

// Calculate responsive canvas size
const getResponsiveCanvasSize = () => {
  const maxWidth = Math.min(window.innerWidth - 40, 600) // 40px for margins, max 600px
  const maxHeight = Math.min(window.innerHeight - 300, 600) // 300px for other elements
  return Math.min(maxWidth, maxHeight)
}

// Pseudo-random generator based on seed (Linear Congruential Generator)
class SeededRandom {
  constructor(seed) {
    this.seed = seed % 2147483647
    if (this.seed <= 0) this.seed += 2147483646
  }
  next() {
    this.seed = (this.seed * 16807) % 2147483647
    return this.seed
  }
  nextFloat() {
    return (this.next() - 1) / 2147483646
  }
}

// Function to hash string to number (for text seed)
const hashString = (str) => {
  let hash = 0
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i)
    hash = ((hash << 5) - hash) + char
    hash = hash & hash // Convert to 32-bit integer
  }
  return Math.abs(hash)
}

// Shuffle array using seed
const shuffleWithSeed = (array, seed) => {
  const rng = new SeededRandom(typeof seed === 'string' ? hashString(seed) : seed)
  const shuffled = [...array]
  for (let i = shuffled.length - 1; i > 0; i--) {
    const j = Math.floor(rng.nextFloat() * (i + 1))
    ;[shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]]
  }
  return shuffled
}

// Generate cards based on difficulty and seed
const generateCards = () => {
  const difficulty = difficulties[gameState.difficulty]
  
  // List of available weapon images
  const weaponImages = [
    'cs_ar_1.webp', 'cs_ar_2.webp', 'cs_ar_3.webp', 'cs_ar_4.webp',
    'cs_gun_1.webp', 'cs_gun_2.webp', 'cs_gun_3.webp', 'cs_gun_4.webp', 'cs_gun_5.webp', 'cs_gun_6.webp',
    'cs_knife_1.webp', 'cs_knife_2.webp', 'cs_knife_3.webp', 'cs_knife_4.webp', 'cs_knife_5.webp', 'cs_knife_6.webp',
    'cs_mg_1.webp', 'cs_mg_2.webp',
    'cs_shotgun_1.webp', 'cs_shotgun_2.webp', 'cs_shotgun_3.webp', 'cs_shotgun_4.webp',
    'cs_smg_1.webp', 'cs_smg_2.webp', 'cs_smg_3.webp', 'cs_smg_4.webp', 'cs_smg_5.webp', 'cs_smg_6.webp',
    'cs_sniper_1.webp', 'cs_sniper_2.webp', 'cs_sniper_3.webp', 'cs_sniper_4.webp'
  ]
  
  const selectedImages = weaponImages.slice(0, difficulty.pairs)
  const cardPairs = []

  selectedImages.forEach((image, index) => {
    // Determine background color based on file name number
    const numberMatch = image.match(/(\d+)\.webp$/)
    const number = numberMatch ? parseInt(numberMatch[1]) : 1
    let backgroundColor
    
    if (number === 1) {
      backgroundColor = '#808080' // Gray background
    } else if (number === 2) {
      backgroundColor = '#2196f3' // Blue background
    } else if (number === 3) {
      backgroundColor = '#9c27b0' // Purple background
    } else {
      backgroundColor = '#ffd700' // Gold background
    }
    
    cardPairs.push(
      { 
        id: index * 2, 
        image, 
        backgroundColor,
        cardClass: `cardClass${index}`, 
        isFlipped: false, 
        isMatched: false 
      },
      { 
        id: index * 2 + 1, 
        image, 
        backgroundColor,
        cardClass: `cardClass${index}`, 
        isFlipped: false, 
        isMatched: false 
      }
    )
  })
  
  const seedValue = gameState.seed || Date.now()
  return shuffleWithSeed(cardPairs, seedValue)
}

const drawRoundedRect = (context, x, y, width, height, radius) => {
  context.beginPath()
  context.moveTo(x + radius, y)
  context.lineTo(x + width - radius, y)
  context.quadraticCurveTo(x + width, y, x + width, y + radius)
  context.lineTo(x + width, y + height - radius)
  context.quadraticCurveTo(x + width, y + height, x + width - radius, y + height)
  context.lineTo(x + radius, y + height)
  context.quadraticCurveTo(x, y + height, x, y + height - radius)
  context.lineTo(x, y + radius)
  context.quadraticCurveTo(x, y, x + radius, y)
  context.closePath()
}

const renderBoard = () => {
  if (!ctx.value || !gameState.cards.length) return
  const canvas = canvasRef.value
  const difficulty = difficulties[gameState.difficulty]
  const gridSize = difficulty.grid
  const cardSize = gameState.canvasSize / gridSize
  const padding = 2
  ctx.value.clearRect(0, 0, canvas.width, canvas.height)
  ctx.value.fillStyle = '#1a1a2e'
  ctx.value.fillRect(0, 0, canvas.width, canvas.height)
  
  gameState.cards.forEach((card, index) => {
    const row = Math.floor(index / gridSize)
    const col = index % gridSize
    const x = col * cardSize + padding
    const y = row * cardSize + padding
    const width = cardSize - padding * 2
    const height = cardSize - padding * 2
    
    const animation = gameState.cardAnimations[index]
    
    if (animation) {
      // Render animation
      const scaleX = animation.progress < 0.5 ? 
        1 - animation.progress * 2 : 
        (animation.progress - 0.5) * 2
      
      // Add bounce effect at the end of the animation
      const bounce = animation.progress > 0.8 ? 
        Math.sin((animation.progress - 0.8) * 10) * 0.05 : 0
      
      // Scale card horizontally (3D rotation effect)
      ctx.value.save()
      ctx.value.translate(x + width/2, y + height/2)
      ctx.value.scale(scaleX + bounce, 1)
      ctx.value.translate(-(x + width/2), -(y + height/2))
      
      ctx.value.shadowColor = 'rgba(0,0,0,0.3)'
      ctx.value.shadowBlur = 10
      ctx.value.shadowOffsetX = 2
      ctx.value.shadowOffsetY = 2
      
      drawCard(card, x, y, width, height, index)
      
      ctx.value.restore()
    } else {
      // Normal rendering without animation
      drawCard(card, x, y, width, height, index)
    }
  })
}

const drawCard = (card, x, y, width, height, cardIndex) => {
  let cardColor
  if (card.isMatched) {
    cardColor = card.backgroundColor || '#2196f3'
  } else if (card.isFlipped) {
    cardColor = card.backgroundColor || '#2196f3'
  } else {
    cardColor = '#424242'
  }
  
  const isHovered = gameState.hoveredCard === cardIndex && !card.isMatched
  
  // Create radial gradient
  const centerX = x + width / 2
  const centerY = y + height / 2
  const radius = Math.min(width, height) / 2
  
  // Adjust gradient intensity based on hover
  const gradientRadius = isHovered ? radius * 0.8 : radius * 0.6
  const gradient = ctx.value.createRadialGradient(
    centerX, centerY, 0,
    centerX, centerY, gradientRadius
  )
  
  // Add colors to gradient
  gradient.addColorStop(0, cardColor + '80') // 50% transparency in the center
  gradient.addColorStop(0.7, cardColor + '40') // 25% transparency
  gradient.addColorStop(1, cardColor) // Full color on the edge
  
  ctx.value.fillStyle = gradient
  
  drawRoundedRect(ctx.value, x, y, width, height, 8)
  ctx.value.fill()
  
  // Add border
  ctx.value.strokeStyle = '#616161'
  ctx.value.lineWidth = 2
  ctx.value.stroke()
  
  // Draw image or text
  if (card.isFlipped || card.isMatched) {
    // Check if image is already loaded
    const cachedImg = loadedImages.value.get(card.image)
    if (cachedImg) {
      drawImageOnCard(cachedImg, x, y, width, height)
    } else {
      // Load image and save to cache
      const img = new Image()
      img.onload = () => {
        loadedImages.value.set(card.image, img)
        drawImageOnCard(img, x, y, width, height)
      }
      img.src = new URL(`../assets/cs_weapons/${card.image}`, import.meta.url).href
    }
  } else {
    // Draw question mark on the card
    ctx.value.fillStyle = '#757575'
    ctx.value.font = `${height * 0.2}px Arial`
    ctx.value.textAlign = 'center'
    ctx.value.textBaseline = 'middle'
    ctx.value.fillText('?', x + width / 2, y + height / 2)
  }
}

// Helper function to draw image on the card
const drawImageOnCard = (img, x, y, width, height) => {
  // Calculate image size to fit in the card with margin
  const margin = width * 0.1
  const imgWidth = width - margin * 2
  const imgHeight = height - margin * 2
  
  const aspectRatio = img.width / img.height
  let drawWidth = imgWidth
  let drawHeight = imgHeight
  
  if (aspectRatio > 1) {
    // Image is wider than higher
    drawHeight = imgWidth / aspectRatio
  } else {
    // Image is higher than wider
    drawWidth = imgHeight * aspectRatio
  }
  
  // Center image in the card
  const drawX = x + (width - drawWidth) / 2
  const drawY = y + (height - drawHeight) / 2
  
  ctx.value.drawImage(img, drawX, drawY, drawWidth, drawHeight)
}

// Handle click on canvas
const handleCanvasClick = (event) => {
  if (!gameState.isGameStarted || gameState.isGameComplete || gameState.isProcessing) return
  
  const canvas = canvasRef.value
  const rect = canvas.getBoundingClientRect()
  
  // Handle both mouse and touch events
  let clientX, clientY
  if (event.touches && event.touches[0]) {
    clientX = event.touches[0].clientX
    clientY = event.touches[0].clientY
  } else if (event.changedTouches && event.changedTouches[0]) {
    clientX = event.changedTouches[0].clientX
    clientY = event.changedTouches[0].clientY
  } else {
    clientX = event.clientX
    clientY = event.clientY
  }
  
  const x = clientX - rect.left
  const y = clientY - rect.top
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  const canvasX = x * scaleX
  const canvasY = y * scaleY
  const difficulty = difficulties[gameState.difficulty]
  const gridSize = difficulty.grid
  const cardSize = gameState.canvasSize / gridSize
  const col = Math.floor(canvasX / cardSize)
  const row = Math.floor(canvasY / cardSize)
  const cardIndex = row * gridSize + col
  
  if (cardIndex >= 0 && cardIndex < gameState.cards.length) {
    const card = gameState.cards[cardIndex]
    
    // Check if card can be flipped
    if (!card.isFlipped && !card.isMatched && gameState.flippedCards.length < 2 && !gameState.flippedCards.includes(cardIndex)) {
      // Set processing flag
      gameState.isProcessing = true
      
      audioManager.cardFlip.play()
      
      // Animate card flip
      animateCardFlip(cardIndex, true)
      gameState.flippedCards.push(cardIndex)
      
      // Check if we have two flipped cards
      if (gameState.flippedCards.length === 2) {
        gameState.moves++
        checkForMatch()
      }
      
      // Save game state after each action
      saveGameState()
      
      // Clear processing flag after short delay
      setTimeout(() => {
        gameState.isProcessing = false
      }, 300) // 300ms delay
    }
  }
}

const checkForMatch = () => {
  const [index1, index2] = gameState.flippedCards
  const card1 = gameState.cards[index1]
  const card2 = gameState.cards[index2]
  
  // Check if cards have the same class (or image as backup)
  const isMatch = card1.cardClass === card2.cardClass || card1.image === card2.image
  
  setTimeout(() => {
    if (isMatch) {
      audioManager.cardMatch.play()
      
      // Mark cards as matched
      gameState.cards[index1].isMatched = true
      gameState.cards[index2].isMatched = true
      
      // Check if game is complete
      const matchedCards = gameState.cards.filter(card => card.isMatched).length
      if (matchedCards === gameState.cards.length) {
        endGame()
      }
    } else {
      // Animate card flip back
      animateCardFlip(index1, false)
      animateCardFlip(index2, false)
    }
    
    // Clear flipped cards list
    gameState.flippedCards = []
    
    // Save game state after checking pairs
    saveGameState()
    
    gameState.isProcessing = false
  }, 1000)
}

const endGame = () => {
  gameState.isGameComplete = true
  if (gameState.timer) {
    clearInterval(gameState.timer)
    gameState.timer = null
  }
  saveGameState()
  saveStatistics()
}

const startNewGame = () => {
  // Check if game configuration exists
  const config = localStorage.getItem('memoryGame_config')
  if (config) {
    try {
      const gameConfig = JSON.parse(config)
      gameState.difficulty = gameConfig.difficulty
      gameState.seed = gameConfig.seed
    } catch {
      // Use default values if configuration is invalid
      gameState.difficulty = 'easy'
      gameState.seed = Date.now().toString()
    }
  }
  
  // Reset game state
  gameState.cards = generateCards()
  gameState.isGameStarted = true
  gameState.isGameComplete = false
  gameState.flippedCards = []
  gameState.moves = 0
  gameState.gameTime = 0
  gameState.cardAnimations = []
  gameState.isLoadedGame = false
  gameState.isProcessing = false
  
  startTimer()
  saveGameState()
}

const goToWelcome = () => {
  router.push('/')
}

const animateCardFlip = (cardIndex, isFlippingTo) => {
  const card = gameState.cards[cardIndex]
  const animation = {
    progress: 0,
    target: isFlippingTo,
    startTime: Date.now(),
    duration: 300 // 300ms
  }
  
  gameState.cardAnimations[cardIndex] = animation
  
  const animate = () => {
    const elapsed = Date.now() - animation.startTime
    animation.progress = Math.min(elapsed / animation.duration, 1)
    
    renderBoard()
    
    if (animation.progress < 1) {
      requestAnimationFrame(animate)
    } else {
      // Animation finished
      card.isFlipped = isFlippingTo
      gameState.cardAnimations[cardIndex] = null
      renderBoard()
    }
  }
  
  requestAnimationFrame(animate)
}

// Preload weapon images
const preloadImages = () => {
  const weaponImages = [
    'cs_ar_1.webp', 'cs_ar_2.webp', 'cs_ar_3.webp', 'cs_ar_4.webp',
    'cs_gun_1.webp', 'cs_gun_2.webp', 'cs_gun_3.webp', 'cs_gun_4.webp', 'cs_gun_5.webp', 'cs_gun_6.webp',
    'cs_knife_1.webp', 'cs_knife_2.webp', 'cs_knife_3.webp', 'cs_knife_4.webp', 'cs_knife_5.webp', 'cs_knife_6.webp',
    'cs_mg_1.webp', 'cs_mg_2.webp',
    'cs_shotgun_1.webp', 'cs_shotgun_2.webp', 'cs_shotgun_3.webp', 'cs_shotgun_4.webp',
    'cs_smg_1.webp', 'cs_smg_2.webp', 'cs_smg_3.webp', 'cs_smg_4.webp', 'cs_smg_5.webp', 'cs_smg_6.webp',
    'cs_sniper_1.webp', 'cs_sniper_2.webp', 'cs_sniper_3.webp', 'cs_sniper_4.webp'
  ]
  
  weaponImages.forEach(imageName => {
    const img = new Image()
    img.onload = () => {
      loadedImages.value.set(imageName, img)
    }
    img.src = new URL(`../assets/cs_weapons/${imageName}`, import.meta.url).href
  })
}

// Initialize Canvas
onMounted(() => {
  preloadImages()
  
  if (canvasRef.value) {
    const canvas = canvasRef.value
    const context = canvas.getContext('2d')
    
    gameState.canvasSize = getResponsiveCanvasSize()
    canvas.width = gameState.canvasSize
    canvas.height = gameState.canvasSize
    
    // Set CSS style for responsiveness
    canvas.style.maxWidth = '100%'
    canvas.style.height = 'auto'
    
    context.imageSmoothingEnabled = true
    ctx.value = context
    
    const loadedGame = loadGameState()
    if (loadedGame) {
      gameState.isLoadedGame = true
      nextTick(() => renderBoard())
    } else {
      // Check if new game configuration exists
      const config = localStorage.getItem('memoryGame_config')
      if (config) {
        try {
          const gameConfig = JSON.parse(config)
          gameState.difficulty = gameConfig.difficulty
          gameState.seed = gameConfig.seed
          // Clear configuration after loading
          localStorage.removeItem('memoryGame_config')
          // Automatically start game with loaded parameters
          startNewGame()
        } catch {
          // If configuration is invalid, go to welcome screen
          router.push('/')
          return
        }
      } else {
        // No saved game and configuration - go to welcome screen
        router.push('/')
        return
      }
    }
    
    renderBoard()
  }
  
  window.addEventListener('beforeunload', handleBeforeUnload)
  window.addEventListener('pagehide', handleBeforeUnload)
  window.addEventListener('resize', handleResize)
  
  onUnmounted(() => {
    window.removeEventListener('beforeunload', handleBeforeUnload)
    window.removeEventListener('pagehide', handleBeforeUnload)
    window.removeEventListener('resize', handleResize)
  })
})

// Save game state before closing the page
const handleBeforeUnload = () => {
  if (gameState.isGameStarted && !gameState.isGameComplete) {
    saveGameState()
  }
}

const handleResize = () => {
  if (canvasRef.value && gameState.isGameStarted) {
    const newSize = getResponsiveCanvasSize()
    if (Math.abs(newSize - gameState.canvasSize) > 20) {
      gameState.canvasSize = newSize
      const canvas = canvasRef.value
      canvas.width = gameState.canvasSize
      canvas.height = gameState.canvasSize
      nextTick(() => renderBoard())
    }
  }
}

// Handle hover for desktop
const handleMouseMove = (event) => {
  if (!gameState.isGameStarted || gameState.isGameComplete) return
  
  const canvas = canvasRef.value
  const rect = canvas.getBoundingClientRect()
  const x = event.clientX - rect.left
  const y = event.clientY - rect.top
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  const canvasX = x * scaleX
  const canvasY = y * scaleY
  const difficulty = difficulties[gameState.difficulty]
  const gridSize = difficulty.grid
  const cardSize = gameState.canvasSize / gridSize
  const col = Math.floor(canvasX / cardSize)
  const row = Math.floor(canvasY / cardSize)
  const cardIndex = row * gridSize + col
  
  if (cardIndex >= 0 && cardIndex < gameState.cards.length) {
    const card = gameState.cards[cardIndex]
    if (!card.isMatched && !card.isFlipped) {
      gameState.hoveredCard = cardIndex
      renderBoard()
    }
  } else {
    gameState.hoveredCard = null
    renderBoard()
  }
}

const handleMouseLeave = () => {
  gameState.hoveredCard = null
  renderBoard()
}

// Handle touch for mobile devices
const touchStartPos = ref(null)

const handleTouchStart = (event) => {
  event.preventDefault()
  const touch = event.touches[0]
  touchStartPos.value = { x: touch.clientX, y: touch.clientY }
}

const handleTouchMove = (event) => {
  event.preventDefault()
  if (!gameState.isGameStarted || gameState.isGameComplete) return
  
  const touch = event.touches[0]
  const canvas = canvasRef.value
  const rect = canvas.getBoundingClientRect()
  const x = touch.clientX - rect.left
  const y = touch.clientY - rect.top
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  const canvasX = x * scaleX
  const canvasY = y * scaleY
  const difficulty = difficulties[gameState.difficulty]
  const gridSize = difficulty.grid
  const cardSize = gameState.canvasSize / gridSize
  const col = Math.floor(canvasX / cardSize)
  const row = Math.floor(canvasY / cardSize)
  const cardIndex = row * gridSize + col
  
  if (cardIndex >= 0 && cardIndex < gameState.cards.length) {
    const card = gameState.cards[cardIndex]
    if (!card.isMatched && !card.isFlipped) {
      gameState.hoveredCard = cardIndex
      renderBoard()
    }
  } else {
    gameState.hoveredCard = null
    renderBoard()
  }
}

const handleTouchEnd = (event) => {
  event.preventDefault()
  if (!touchStartPos.value) return
  
  const touch = event.changedTouches[0]
  const deltaX = Math.abs(touch.clientX - touchStartPos.value.x)
  const deltaY = Math.abs(touch.clientY - touchStartPos.value.y)
  
  // If the displacement is small, treat it as a click
  if (deltaX < 10 && deltaY < 10) {
    handleCanvasClick(event)
  }
  
  touchStartPos.value = null
  gameState.hoveredCard = null
  renderBoard()
}

// Clear timer when component is unmounted
onUnmounted(() => {
  if (gameState.timer) {
    clearInterval(gameState.timer)
    gameState.timer = null
  }
  // Save game state before closing
  if (gameState.isGameStarted && !gameState.isGameComplete) {
    saveGameState()
  }
})

// Render board when game state changes
watch(
  () => [gameState.cards, gameState.difficulty, gameState.canvasSize],
  () => {
    nextTick(() => renderBoard())
  },
  { deep: true }
)

// Regenerate cards when difficulty or seed changes
watch(
  () => [gameState.difficulty, gameState.seed],
  () => {
    if (gameState.isGameStarted && !gameState.isLoadedGame) {
      // Stop timer if game was active
      if (gameState.timer) {
        clearInterval(gameState.timer)
        gameState.timer = null
      }
      // Reset game state
      gameState.cards = generateCards()
      gameState.isGameComplete = false
      gameState.flippedCards = []
      gameState.moves = 0
      gameState.gameTime = 0
      gameState.cardAnimations = []
      gameState.isLoadedGame = false
      // Restart timer
      startTimer()
      saveGameState()
    }
  }
)

// Keys for localStorage
const STORAGE_KEYS = {
  ACTIVE_GAME: 'memoryGame_activeGame',
  GAME_STATE: 'memoryGame_gameState',
  STATISTICS: 'memoryGame_statistics'
}

// Function to save game state
const saveGameState = () => {
  const gameData = {
    isGameStarted: gameState.isGameStarted,
    isGameComplete: gameState.isGameComplete,
    difficulty: gameState.difficulty,
    seed: gameState.seed,
    cards: gameState.cards,
    flippedCards: gameState.flippedCards,
    moves: gameState.moves,
    gameTime: gameState.gameTime,
    lastSaveTime: Date.now()
  }
  
  try {
    localStorage.setItem(STORAGE_KEYS.GAME_STATE, JSON.stringify(gameData))
    localStorage.setItem(STORAGE_KEYS.ACTIVE_GAME, 'true')
  } catch (error) {
    console.error('Błąd podczas zapisywania stanu gry:', error)
  }
}

// Function to load game state
const loadGameState = () => {
  const activeGame = localStorage.getItem(STORAGE_KEYS.ACTIVE_GAME)
  
  if (!activeGame) {
    return false
  }
  
  const gameData = localStorage.getItem(STORAGE_KEYS.GAME_STATE)
  
  if (!gameData) {
    return false
  }
  
  try {
    const savedState = JSON.parse(gameData)
    
    // Check if game is older than 24 hours
    const hoursSinceSave = (Date.now() - savedState.lastSaveTime) / (1000 * 60 * 60)
    
    if (hoursSinceSave > 24) {
      clearGameState()
      return false
    }
    
    // Restore game state
    gameState.isGameStarted = savedState.isGameStarted
    gameState.isGameComplete = savedState.isGameComplete
    gameState.difficulty = savedState.difficulty
    gameState.seed = savedState.seed
    gameState.cards = savedState.cards
    gameState.flippedCards = savedState.flippedCards
    gameState.moves = savedState.moves
    gameState.gameTime = savedState.gameTime
    gameState.isLoadedGame = true
    gameState.isProcessing = false
    
    // Start timer if game is active and not complete
    if (gameState.isGameStarted && !gameState.isGameComplete) {
      startTimer()
    }
    
    return true
  } catch (error) {
    console.error('Błąd podczas wczytywania stanu gry:', error)
    clearGameState()
    return false
  }
}

// Function to clear game state
const clearGameState = () => {
  localStorage.removeItem(STORAGE_KEYS.GAME_STATE)
  localStorage.removeItem(STORAGE_KEYS.ACTIVE_GAME)
}

// Function to save statistics
const saveStatistics = () => {
  const stats = {
    seed: gameState.seed,
    moves: gameState.moves,
    gameTime: gameState.gameTime,
    difficulty: gameState.difficulty,
    completedAt: Date.now()
  }
  
  // Load existing statistics
  const existingStats = localStorage.getItem(STORAGE_KEYS.STATISTICS)
  let statistics = existingStats ? JSON.parse(existingStats) : []
  
  // Add new statistic
  statistics.push(stats)
  
  // Keep only last 50 games
  if (statistics.length > 50) {
    statistics = statistics.slice(-50)
  }
  
  localStorage.setItem(STORAGE_KEYS.STATISTICS, JSON.stringify(statistics))
}

const startTimer = () => {
  if (gameState.timer) {
    clearInterval(gameState.timer)
  }
  gameState.timer = setInterval(() => {
    gameState.gameTime++
    // Save state every 5 seconds
    if (gameState.gameTime % 5 === 0) {
      saveGameState()
    }
  }, 1000)
}
</script>

<style scoped lang="scss">
.memory-game-container {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 16px;
  backdrop-filter: blur(15px);
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.3);
  padding: 2rem;
  max-width: 800px;
  width: 100%;
  margin: 0 auto;
}

.game {
  &-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
    flex-wrap: wrap;
    gap: 1rem;
  }

  &-title {
    font-family: 'CS-Regular', 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    font-size: 2.5rem;
    font-weight: bold;
  }
}

.back-button {
  padding: 0.75rem 1.5rem;
  background: #6b7280;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: bold;
  font-size: 0.875rem;
  cursor: pointer;

  &:hover {
    background: #4b5563;
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  }

  &:active {
    transform: translateY(0);
  }
}

@media (max-width: 640px) {
  .game {
    &-title {
      font-size: 2rem;
    }
  }
}

@media (max-width: 640px) {
  .memory-game-container {
    padding: 1rem;
    margin: 0 0.5rem;
  }

  .game {
    &-title {
      font-size: 2rem;
    }

    &-header {
      flex-direction: column;
      text-align: center;
    }
  }
}
</style> 