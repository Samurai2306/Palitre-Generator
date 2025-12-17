<template>
  <div class="app-container">
    <header class="header">
      <h1>Генератор палитр</h1>
    </header>

    <main class="main-content">
      <!-- Панель управления -->
      <section class="controls">
        <div class="control-group">
          <label for="color-count">Количество цветов:</label>
          <select id="color-count" v-model="colorCount" @change="generatePalette">
            <option :value="3">3</option>
            <option :value="5">5</option>
            <option :value="7">7</option>
          </select>
        </div>

        <div class="control-group">
          <label for="color-format">Формат:</label>
          <select id="color-format" v-model="colorFormat">
            <option value="hex">HEX</option>
            <option value="rgb">RGB</option>
          </select>
        </div>

        <button class="btn btn-primary" @click="generatePalette">
          Случайная палитра
        </button>
      </section>

      <!-- Заготовленные палитры -->
      <section class="preset-palettes-section">
        <div class="preset-palettes">
          <div
            v-for="(preset, index) in presetPalettes"
            :key="index"
            class="preset-palette"
            @click="loadPresetPalette(preset.colors)"
          >
            <div class="preset-name">{{ preset.name }}</div>
            <div class="preset-colors">
              <div
                v-for="(color, colorIndex) in preset.colors"
                :key="colorIndex"
                class="preset-color"
                :style="{ backgroundColor: color }"
                :title="color"
              ></div>
            </div>
          </div>
        </div>
      </section>

      <!-- Палитра цветов -->
      <section class="palette-section">
        <div class="palette" v-if="palette.length > 0">
          <div
            v-for="(color, index) in palette"
            :key="index"
            class="color-card"
            :class="{ 'pinned': pinnedColors.includes(index) }"
            :style="{ backgroundColor: color }"
            @click="copyToClipboard(color, index)"
          >
            <div class="color-info">
              <div class="color-text-info">
                <span class="color-value">{{ formatColor(color) }}</span>
                <span v-if="colorNames[index]" class="color-name">{{ colorNames[index] }}</span>
                <span v-else-if="loadingNames[index]" class="color-name loading">Загрузка...</span>
              </div>
              <div class="color-actions">
                <button
                  class="edit-btn"
                  @click.stop="openColorEditor(index)"
                  title="Редактировать цвет"
                >
                  ✏️
                </button>
                <button
                  class="pin-btn"
                  @click.stop="togglePin(index)"
                  :title="pinnedColors.includes(index) ? 'Открепить' : 'Закрепить'"
                >
                  {{ pinnedColors.includes(index) ? '📌' : '📍' }}
                </button>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- Уведомление о копировании -->
      <transition name="fade">
        <div v-if="showNotification" class="notification">
          ✓ Скопировано в буфер обмена!
        </div>
      </transition>

      <!-- Модальное окно редактора цвета -->
      <transition name="modal">
        <div v-if="editingColorIndex !== null" class="modal-overlay" @click="closeColorEditor">
          <div class="modal-content" @click.stop>
            <div class="modal-header">
              <h3>Редактирование цвета</h3>
              <button class="close-btn" @click="closeColorEditor">✕</button>
            </div>
            <div class="color-preview-large" :style="{ backgroundColor: editingColor }">
              <div class="color-preview-info">
                <div class="preview-hex">{{ editingColor }}</div>
                <div class="preview-rgb">{{ hexToRgb(editingColor) }}</div>
              </div>
            </div>
            <div class="color-editor">
              <div class="editor-section">
                <label>Оттенок (Hue): {{ editingHsl.h }}°</label>
                <input
                  type="range"
                  min="0"
                  max="360"
                  v-model.number="editingHsl.h"
                  @input="updateColorFromHsl"
                  class="slider"
                />
              </div>
              <div class="editor-section">
                <label>Насыщенность (Saturation): {{ editingHsl.s }}%</label>
                <input
                  type="range"
                  min="0"
                  max="100"
                  v-model.number="editingHsl.s"
                  @input="updateColorFromHsl"
                  class="slider"
                />
              </div>
              <div class="editor-section">
                <label>Яркость (Lightness): {{ editingHsl.l }}%</label>
                <input
                  type="range"
                  min="0"
                  max="100"
                  v-model.number="editingHsl.l"
                  @input="updateColorFromHsl"
                  class="slider"
                />
              </div>
              <div class="editor-actions">
                <button class="btn btn-secondary" @click="closeColorEditor">Отмена</button>
                <button class="btn btn-primary" @click="applyColorEdit">Применить</button>
              </div>
            </div>
          </div>
        </div>
      </transition>

      <!-- Кнопка портфолио -->
      <a href="https://samurai2306.github.io/portfolio_project" 
         target="_blank" 
         rel="noopener noreferrer" 
         class="portfolio-link" 
         aria-label="Портфолио">
      </a>

      <!-- Просмотрщик с mockup -->
      <section class="preview-section">
        <div class="preview-controls">
          <h2>Превью палитры</h2>
          <div class="theme-toggle">
            <label>
              <input type="checkbox" v-model="isDarkTheme" />
              <span>Тёмный фон</span>
            </label>
          </div>
        </div>

        <div class="mockup" :class="{ 'dark': isDarkTheme }">
          <div class="mockup-header" :style="{ backgroundColor: palette[0] || '#667eea' }">
            <h3>Заголовок</h3>
          </div>
          <div class="mockup-content">
            <div class="mockup-card" :style="{ backgroundColor: palette[1] || '#f0f0f0' }">
              <p>Карточка контента</p>
            </div>
            <button class="mockup-button" :style="{ backgroundColor: palette[2] || '#4CAF50' }">
              Кнопка действия
            </button>
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'

// Состояние
const palette = ref([])
const colorCount = ref(5)
const colorFormat = ref('hex')
const pinnedColors = ref([])
const showNotification = ref(false)
const isDarkTheme = ref(false)
const editingColorIndex = ref(null)
const editingColor = ref('#000000')
const editingHsl = ref({ h: 0, s: 0, l: 0 })
const colorNames = ref({})
const loadingNames = ref({})

// Заготовленные палитры
const presetPalettes = ref([
  {
    name: 'Океан',
    colors: ['#1E3A5F', '#2E5984', '#4A90A4', '#6BB6B8', '#8FD3D1']
  },
  {
    name: 'Закат',
    colors: ['#FF6B6B', '#FF8E53', '#FFA07A', '#FFB347', '#FFD700']
  },
  {
    name: 'Лес',
    colors: ['#2D5016', '#3D7C2E', '#5A9F4F', '#7BC77E', '#9DD89F']
  },
  {
    name: 'Лаванда',
    colors: ['#6B4C93', '#8B6FA8', '#A892BD', '#C5B5D2', '#E2D8E7']
  },
  {
    name: 'Коралл',
    colors: ['#FF6F61', '#FF8C69', '#FFA07A', '#FFB4A0', '#FFC8C6']
  },
  {
    name: 'Небо',
    colors: ['#4A90E2', '#5BA3F5', '#6CB6FF', '#7DC9FF', '#8EDCFF']
  },
  {
    name: 'Земля',
    colors: ['#8B4513', '#A0522D', '#CD853F', '#DEB887', '#F5DEB3']
  },
  {
    name: 'Розовый',
    colors: ['#FF1493', '#FF69B4', '#FFB6C1', '#FFC0CB', '#FFE4E1']
  },
  {
    name: 'Изумруд',
    colors: ['#006400', '#228B22', '#32CD32', '#90EE90', '#98FB98']
  },
  {
    name: 'Фиолет',
    colors: ['#4B0082', '#6A0DAD', '#9370DB', '#BA55D3', '#DA70D6']
  },
  {
    name: 'Золото',
    colors: ['#B8860B', '#DAA520', '#FFD700', '#FFE55C', '#FFF8DC']
  },
  {
    name: 'Мятный',
    colors: ['#00CED1', '#20B2AA', '#48D1CC', '#7FFFD4', '#B0E0E6']
  }
])

// Генерация случайного цвета в HEX
const generateRandomColor = () => {
  return '#' + Math.floor(Math.random() * 16777215).toString(16).padStart(6, '0').toUpperCase()
}

// Генерация гармоничной палитры
const generatePalette = () => {
  const newPalette = []
  const baseHue = Math.random() * 360
  
  for (let i = 0; i < colorCount.value; i++) {
    // Сохраняем закреплённые цвета
    if (pinnedColors.value.includes(i) && palette.value[i]) {
      newPalette[i] = palette.value[i]
    } else {
      // Генерируем гармоничные цвета на основе базового оттенка
      const hue = (baseHue + (i * 60)) % 360
      const saturation = 50 + Math.random() * 30
      const lightness = 40 + Math.random() * 30
      
      // Конвертируем HSL в HEX
      const color = hslToHex(hue, saturation, lightness)
      newPalette[i] = color
    }
  }
  
  palette.value = newPalette
  saveToLocalStorage()
  // Загружаем названия цветов для новой палитры
  loadColorNames()
}

// Конвертация HSL в HEX
const hslToHex = (h, s, l) => {
  l /= 100
  const a = s * Math.min(l, 1 - l) / 100
  const f = n => {
    const k = (n + h / 30) % 12
    const color = l - a * Math.max(Math.min(k - 3, 9 - k, 1), -1)
    return Math.round(255 * color).toString(16).padStart(2, '0')
  }
  return `#${f(0)}${f(8)}${f(4)}`.toUpperCase()
}

// Форматирование цвета для отображения
const formatColor = (hex) => {
  if (colorFormat.value === 'rgb') {
    return hexToRgb(hex)
  }
  return hex
}

// Конвертация HEX в RGB
const hexToRgb = (hex) => {
  const r = parseInt(hex.slice(1, 3), 16)
  const g = parseInt(hex.slice(3, 5), 16)
  const b = parseInt(hex.slice(5, 7), 16)
  return `rgb(${r}, ${g}, ${b})`
}

// Копирование в буфер обмена
const copyToClipboard = async (color, index) => {
  const textToCopy = formatColor(color)
  try {
    await navigator.clipboard.writeText(textToCopy)
    showNotification.value = true
    setTimeout(() => {
      showNotification.value = false
    }, 2000)
  } catch (err) {
    console.error('Ошибка копирования:', err)
  }
}

// Закрепление/открепление цвета
const togglePin = (index) => {
  const pinIndex = pinnedColors.value.indexOf(index)
  if (pinIndex > -1) {
    pinnedColors.value.splice(pinIndex, 1)
  } else {
    pinnedColors.value.push(index)
  }
  saveToLocalStorage()
}

// Сохранение в localStorage
const saveToLocalStorage = () => {
  localStorage.setItem('palette', JSON.stringify(palette.value))
  localStorage.setItem('pinnedColors', JSON.stringify(pinnedColors.value))
  localStorage.setItem('colorCount', JSON.stringify(colorCount.value))
  localStorage.setItem('colorFormat', colorFormat.value)
}

// Загрузка из localStorage
const loadFromLocalStorage = () => {
  const savedPalette = localStorage.getItem('palette')
  const savedPinned = localStorage.getItem('pinnedColors')
  const savedCount = localStorage.getItem('colorCount')
  const savedFormat = localStorage.getItem('colorFormat')
  
  if (savedPalette) {
    palette.value = JSON.parse(savedPalette)
  }
  if (savedPinned) {
    pinnedColors.value = JSON.parse(savedPinned)
  }
  if (savedCount) {
    colorCount.value = parseInt(savedCount)
  }
  if (savedFormat) {
    colorFormat.value = savedFormat
  }
}

// Конвертация HEX в HSL
const hexToHsl = (hex) => {
  const r = parseInt(hex.slice(1, 3), 16) / 255
  const g = parseInt(hex.slice(3, 5), 16) / 255
  const b = parseInt(hex.slice(5, 7), 16) / 255

  const max = Math.max(r, g, b)
  const min = Math.min(r, g, b)
  let h, s, l = (max + min) / 2

  if (max === min) {
    h = s = 0
  } else {
    const d = max - min
    s = l > 0.5 ? d / (2 - max - min) : d / (max + min)
    switch (max) {
      case r: h = ((g - b) / d + (g < b ? 6 : 0)) / 6; break
      case g: h = ((b - r) / d + 2) / 6; break
      case b: h = ((r - g) / d + 4) / 6; break
    }
  }

  return {
    h: Math.round(h * 360),
    s: Math.round(s * 100),
    l: Math.round(l * 100)
  }
}

// Открытие редактора цвета
const openColorEditor = (index) => {
  editingColorIndex.value = index
  editingColor.value = palette.value[index]
  editingHsl.value = hexToHsl(palette.value[index])
}

// Закрытие редактора цвета
const closeColorEditor = () => {
  editingColorIndex.value = null
}

// Обновление цвета из HSL
const updateColorFromHsl = () => {
  editingColor.value = hslToHex(editingHsl.value.h, editingHsl.value.s, editingHsl.value.l)
}

// Применение изменений цвета
const applyColorEdit = async () => {
  if (editingColorIndex.value !== null) {
    palette.value[editingColorIndex.value] = editingColor.value
    saveToLocalStorage()
    closeColorEditor()
    // Загружаем новое название для измененного цвета
    await getColorName(editingColor.value, editingColorIndex.value)
  }
}

// Получение названия цвета через API
const getColorName = async (hexColor, index) => {
  try {
    loadingNames.value[index] = true
    // Убираем # из начала HEX кода
    const hexWithoutHash = hexColor.replace('#', '')
    
    const response = await fetch(`https://www.thecolorapi.com/id?hex=${hexWithoutHash}`)
    
    if (!response.ok) {
      throw new Error('Ошибка при получении названия цвета')
    }
    
    const data = await response.json()
    const colorName = data.name?.value || 'Неизвестный цвет'
    
    // Сохраняем название цвета
    colorNames.value[index] = colorName
    loadingNames.value[index] = false
    
    return colorName
  } catch (error) {
    console.error('Ошибка при получении названия цвета:', error)
    loadingNames.value[index] = false
    colorNames.value[index] = null
    return null
  }
}

// Загрузка названий для всех цветов в палитре
const loadColorNames = async () => {
  // Очищаем предыдущие названия
  colorNames.value = {}
  loadingNames.value = {}
  
  // Загружаем названия для каждого цвета параллельно
  const promises = palette.value.map((color, index) => 
    getColorName(color, index)
  )
  
  await Promise.all(promises)
}

// Загрузка заготовленной палитры
const loadPresetPalette = (colors) => {
  palette.value = [...colors]
  colorCount.value = colors.length
  pinnedColors.value = []
  saveToLocalStorage()
  // Загружаем названия цветов для заготовленной палитры
  loadColorNames()
}

// Отслеживание изменений формата
watch(colorFormat, () => {
  saveToLocalStorage()
})

// Инициализация
onMounted(async () => {
  loadFromLocalStorage()
  if (palette.value.length === 0) {
    generatePalette()
  } else {
    // Загружаем названия для существующей палитры
    await loadColorNames()
  }
})
</script>

<style scoped>
.app-container {
  background: rgba(45, 27, 61, 0.4);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 24px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 
    0 8px 32px rgba(0, 0, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  overflow: hidden;
  min-height: calc(100vh - 40px);
}

.header {
  background: rgba(147, 112, 219, 0.2);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.95);
  padding: 40px 30px;
  text-align: center;
  position: relative;
  overflow: hidden;
}

.header::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, rgba(186, 85, 211, 0.1) 0%, rgba(147, 112, 219, 0.1) 100%);
  pointer-events: none;
}

.header h1 {
  font-size: 2.8em;
  font-weight: 700;
  letter-spacing: -0.02em;
  text-shadow: 0 2px 20px rgba(147, 112, 219, 0.3);
  position: relative;
  z-index: 1;
}

.main-content {
  padding: 40px;
}

.controls {
  display: flex;
  gap: 20px;
  align-items: center;
  flex-wrap: wrap;
  margin-bottom: 40px;
  padding: 24px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(15px);
  -webkit-backdrop-filter: blur(15px);
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 
    0 4px 20px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
}

.control-group {
  display: flex;
  align-items: center;
  gap: 12px;
}

.control-group label {
  font-weight: 600;
  color: rgba(255, 255, 255, 0.85);
  font-size: 15px;
}

.control-group select {
  padding: 12px 18px;
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 12px;
  font-size: 15px;
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  color: rgba(255, 255, 255, 0.9);
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.control-group select:hover {
  border-color: rgba(186, 85, 211, 0.4);
  background: rgba(255, 255, 255, 0.12);
  transform: translateY(-1px);
}

.control-group select:focus {
  outline: none;
  border-color: rgba(186, 85, 211, 0.6);
  box-shadow: 0 0 0 3px rgba(186, 85, 211, 0.2);
}

.control-group select option {
  background: #2d1b3d;
  color: rgba(255, 255, 255, 0.9);
}

.btn {
  padding: 14px 28px;
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 12px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
}

.btn-primary {
  background: linear-gradient(135deg, rgba(186, 85, 211, 0.3) 0%, rgba(147, 112, 219, 0.3) 100%);
  color: rgba(255, 255, 255, 0.95);
  margin-left: auto;
  border: 1px solid rgba(186, 85, 211, 0.3);
  box-shadow: 
    0 4px 15px rgba(186, 85, 211, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 
    0 6px 20px rgba(186, 85, 211, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.15);
  background: linear-gradient(135deg, rgba(186, 85, 211, 0.4) 0%, rgba(147, 112, 219, 0.4) 100%);
}

.btn-primary:active {
  transform: translateY(0);
}

.palette-section {
  margin-bottom: 40px;
}

.palette {
  display: flex;
  gap: 20px;
  flex-wrap: wrap;
  justify-content: center;
}

.color-card {
  flex: 1;
  min-width: 160px;
  min-height: 220px;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 
    0 8px 32px rgba(0, 0, 0, 0.3),
    0 2px 8px rgba(0, 0, 0, 0.2);
  display: flex;
  align-items: flex-end;
  padding: 20px;
  position: relative;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.1);
  background-clip: padding-box;
}

.color-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(180deg, transparent 0%, rgba(0, 0, 0, 0.1) 100%);
  pointer-events: none;
}

.color-card:hover {
  transform: translateY(-8px) scale(1.02);
  box-shadow: 
    0 16px 48px rgba(0, 0, 0, 0.4),
    0 4px 16px rgba(0, 0, 0, 0.3);
  border-color: rgba(255, 255, 255, 0.2);
}

.color-card.pinned {
  border: 2px solid rgba(255, 215, 0, 0.6);
  box-shadow: 
    0 8px 32px rgba(255, 215, 0, 0.3),
    0 0 20px rgba(255, 215, 0, 0.2),
    inset 0 0 20px rgba(255, 215, 0, 0.1);
}

.color-info {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  padding: 12px 16px;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 
    0 4px 16px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.2);
  position: relative;
  z-index: 1;
}

.color-text-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
  flex: 1;
  min-width: 0;
}

.color-actions {
  display: flex;
  gap: 8px;
  align-items: center;
}

.edit-btn {
  background: rgba(255, 255, 255, 0.12);
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 10px;
  font-size: 16px;
  cursor: pointer;
  padding: 8px 10px;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: 
    0 2px 8px rgba(0, 0, 0, 0.15),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 36px;
  min-height: 36px;
}

.edit-btn:hover {
  transform: scale(1.1);
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.35);
  box-shadow: 
    0 4px 12px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.15);
}

.color-value {
  font-weight: 600;
  font-size: 14px;
  color: rgba(255, 255, 255, 0.95);
  font-family: 'Courier New', monospace;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
  letter-spacing: 0.5px;
}

.color-name {
  font-weight: 500;
  font-size: 12px;
  color: rgba(255, 255, 255, 0.85);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
  font-style: italic;
  text-transform: capitalize;
  opacity: 0.9;
}

.color-name.loading {
  font-style: normal;
  opacity: 0.7;
  animation: pulse 1.5s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% {
    opacity: 0.7;
  }
  50% {
    opacity: 0.4;
  }
}

.pin-btn {
  background: rgba(255, 255, 255, 0.12);
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 10px;
  font-size: 18px;
  cursor: pointer;
  padding: 8px 10px;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: 
    0 2px 8px rgba(0, 0, 0, 0.15),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 36px;
  min-height: 36px;
}

.pin-btn:hover {
  transform: scale(1.1) rotate(5deg);
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.35);
  box-shadow: 
    0 4px 12px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.15);
}

.color-card.pinned .pin-btn {
  background: rgba(255, 215, 0, 0.2);
  border-color: rgba(255, 215, 0, 0.4);
  box-shadow: 
    0 2px 8px rgba(255, 215, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.2);
}

.notification {
  position: fixed;
  top: 30px;
  right: 30px;
  background: rgba(76, 175, 80, 0.25);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  color: rgba(255, 255, 255, 0.95);
  padding: 16px 28px;
  border-radius: 16px;
  border: 1px solid rgba(76, 175, 80, 0.3);
  box-shadow: 
    0 8px 32px rgba(76, 175, 80, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  z-index: 1000;
  font-weight: 600;
  font-size: 15px;
}

.fade-enter-active, .fade-leave-active {
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
  transform: translateY(-20px) scale(0.95);
}

.preset-palettes-section {
  margin-bottom: 40px;
}

.preset-palettes {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 20px;
}

.preset-palette {
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(15px);
  -webkit-backdrop-filter: blur(15px);
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  padding: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 
    0 4px 20px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
}

.preset-palette:hover {
  transform: translateY(-4px);
  background: rgba(255, 255, 255, 0.08);
  border-color: rgba(255, 255, 255, 0.2);
  box-shadow: 
    0 8px 30px rgba(0, 0, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.15);
}

.preset-name {
  color: rgba(255, 255, 255, 0.9);
  font-weight: 600;
  font-size: 16px;
  margin-bottom: 12px;
  text-align: center;
}

.preset-colors {
  display: flex;
  gap: 8px;
  height: 60px;
}

.preset-color {
  flex: 1;
  border-radius: 8px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  transition: transform 0.2s;
}

.preset-color:hover {
  transform: scale(1.1);
  z-index: 1;
}

.preview-section {
  margin-top: 40px;
  padding: 32px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(15px);
  -webkit-backdrop-filter: blur(15px);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 
    0 4px 20px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
}

.preview-controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
  flex-wrap: wrap;
  gap: 20px;
}

.preview-controls h2 {
  color: rgba(255, 255, 255, 0.9);
  font-size: 1.8em;
  font-weight: 700;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.theme-toggle label {
  display: flex;
  align-items: center;
  gap: 12px;
  cursor: pointer;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.85);
  font-size: 15px;
  padding: 10px 16px;
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.15);
  transition: all 0.3s ease;
}

.theme-toggle label:hover {
  background: rgba(255, 255, 255, 0.12);
  border-color: rgba(255, 255, 255, 0.2);
}

.theme-toggle input[type="checkbox"] {
  width: 22px;
  height: 22px;
  cursor: pointer;
  accent-color: rgba(186, 85, 211, 0.6);
}

.mockup {
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 16px;
  padding: 32px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 
    0 4px 20px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  transition: all 0.3s ease;
}

.mockup.dark {
  background: rgba(26, 26, 26, 0.4);
  border-color: rgba(255, 255, 255, 0.15);
}

.mockup-header {
  padding: 32px;
  border-radius: 12px;
  margin-bottom: 24px;
  color: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
}

.mockup-header h3 {
  font-size: 2.2em;
  font-weight: 700;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
}

.mockup-content {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.mockup-card {
  padding: 28px;
  border-radius: 12px;
  color: rgba(255, 255, 255, 0.95);
  min-height: 120px;
  display: flex;
  align-items: center;
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
}

.mockup-card p {
  font-size: 1.3em;
  font-weight: 500;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
}

.mockup-button {
  padding: 16px 32px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 12px;
  color: rgba(255, 255, 255, 0.95);
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  align-self: flex-start;
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: 
    0 4px 16px rgba(0, 0, 0, 0.2),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
}

.mockup-button:hover {
  transform: translateY(-2px) scale(1.02);
  box-shadow: 
    0 6px 20px rgba(0, 0, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.3);
}

.mockup-button:active {
  transform: translateY(0) scale(1);
}

/* Планшеты */
@media (max-width: 1024px) {
  .preset-palettes {
    grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  }
  
  .main-content {
    padding: 30px;
  }
}

/* Мобильные устройства */
@media (max-width: 768px) {
  .main-content {
    padding: 20px;
  }
  
  .header {
    padding: 24px 20px;
  }
  
  .header h1 {
    font-size: 1.8em;
  }
  
  .palette {
    flex-direction: column;
    gap: 15px;
  }
  
  .color-card {
    min-width: 100%;
    min-height: 180px;
  }
  
  .controls {
    flex-direction: column;
    align-items: stretch;
    gap: 15px;
    padding: 20px;
  }
  
  .control-group {
    flex-direction: column;
    align-items: flex-start;
    width: 100%;
  }
  
  .control-group select {
    width: 100%;
  }
  
  .btn-primary {
    margin-left: 0;
    width: 100%;
  }
  
  .preview-controls {
    flex-direction: column;
    align-items: flex-start;
    gap: 15px;
  }
  
  .preview-controls h2 {
    font-size: 1.5em;
  }
  
  .preview-section {
    padding: 24px;
  }
  
  .preset-palettes {
    grid-template-columns: 1fr;
    gap: 15px;
  }
  
  .preset-palette {
    padding: 16px;
  }
  
  .preset-name {
    font-size: 15px;
    margin-bottom: 10px;
  }
  
  .preset-colors {
    height: 50px;
    gap: 6px;
  }
  
  .mockup {
    padding: 20px;
  }
  
  .mockup-header {
    padding: 20px;
  }
  
  .mockup-header h3 {
    font-size: 1.6em;
  }
  
  .mockup-card {
    padding: 20px;
    min-height: 100px;
  }
  
  .mockup-card p {
    font-size: 1.1em;
  }
  
  .color-info {
    padding: 10px 12px;
    flex-wrap: wrap;
    gap: 8px;
  }
  
  .color-value {
    font-size: 13px;
    flex: 1;
    min-width: 100px;
  }
  
  .color-actions {
    gap: 6px;
  }
  
  .edit-btn,
  .pin-btn {
    min-width: 32px;
    min-height: 32px;
    padding: 6px 8px;
    font-size: 16px;
  }
}

/* Очень маленькие экраны */
@media (max-width: 480px) {
  body {
    padding: 10px;
  }
  
  .app-container {
    min-height: calc(100vh - 20px);
    border-radius: 16px;
  }
  
  .main-content {
    padding: 16px;
  }
  
  .header {
    padding: 20px 16px;
  }
  
  .header h1 {
    font-size: 1.5em;
  }
  
  .controls {
    padding: 16px;
  }
  
  .control-group label {
    font-size: 14px;
  }
  
  .control-group select {
    padding: 10px 14px;
    font-size: 14px;
  }
  
  .btn {
    padding: 12px 20px;
    font-size: 15px;
  }
  
  .color-card {
    min-height: 160px;
    padding: 16px;
  }
  
  .color-value {
    font-size: 12px;
  }
  
  .preview-section {
    padding: 20px;
  }
  
  .preview-controls h2 {
    font-size: 1.3em;
  }
  
  .mockup {
    padding: 16px;
  }
  
  .mockup-header {
    padding: 16px;
  }
  
  .mockup-header h3 {
    font-size: 1.4em;
  }
  
  .notification {
    top: 15px;
    right: 15px;
    left: 15px;
    padding: 12px 20px;
    font-size: 14px;
  }
  
  .modal-overlay {
    padding: 10px;
    align-items: flex-start;
    padding-top: 20px;
  }
  
  .modal-content {
    max-width: 100%;
    border-radius: 20px;
    max-height: calc(100vh - 40px);
  }
  
  .modal-header {
    padding: 20px;
  }
  
  .modal-header h3 {
    font-size: 1.3em;
  }
  
  .color-preview-large {
    height: 150px;
    margin: 20px;
    padding: 16px;
  }
  
  .preview-hex {
    font-size: 16px;
  }
  
  .preview-rgb {
    font-size: 12px;
  }
  
  .color-editor {
    padding: 20px;
  }
  
  .editor-section {
    margin-bottom: 20px;
  }
  
  .editor-section label {
    font-size: 13px;
    margin-bottom: 10px;
  }
  
  .editor-actions {
    flex-direction: column;
    gap: 10px;
  }
  
  .editor-actions .btn {
    width: 100%;
  }
}

/* Модальное окно редактора */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 20px;
  overflow-y: auto;
}

.modal-content {
  background: rgba(45, 27, 61, 0.95);
  backdrop-filter: blur(30px);
  -webkit-backdrop-filter: blur(30px);
  border-radius: 24px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 
    0 20px 60px rgba(0, 0, 0, 0.5),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  max-width: 500px;
  width: 100%;
  max-height: 90vh;
  overflow-y: auto;
  margin: auto;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.modal-header h3 {
  color: rgba(255, 255, 255, 0.95);
  font-size: 1.5em;
  font-weight: 700;
  margin: 0;
}

.close-btn {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  color: rgba(255, 255, 255, 0.9);
  font-size: 20px;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.3s ease;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: rotate(90deg);
}

.color-preview-large {
  height: 200px;
  border-radius: 16px;
  margin: 24px;
  display: flex;
  align-items: flex-end;
  padding: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}

.color-preview-info {
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  padding: 12px 16px;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  width: 100%;
}

.preview-hex {
  color: rgba(255, 255, 255, 0.95);
  font-weight: 700;
  font-size: 18px;
  font-family: 'Courier New', monospace;
  margin-bottom: 4px;
}

.preview-rgb {
  color: rgba(255, 255, 255, 0.8);
  font-size: 14px;
  font-family: 'Courier New', monospace;
}

.color-editor {
  padding: 24px;
}

.editor-section {
  margin-bottom: 24px;
}

.editor-section label {
  display: block;
  color: rgba(255, 255, 255, 0.9);
  font-weight: 600;
  margin-bottom: 12px;
  font-size: 14px;
}

.slider {
  width: 100%;
  height: 8px;
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.1);
  outline: none;
  -webkit-appearance: none;
  appearance: none;
}

.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: linear-gradient(135deg, rgba(186, 85, 211, 0.8) 0%, rgba(147, 112, 219, 0.8) 100%);
  border: 2px solid rgba(255, 255, 255, 0.3);
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  transition: all 0.2s;
}

.slider::-webkit-slider-thumb:hover {
  transform: scale(1.2);
  box-shadow: 0 4px 12px rgba(186, 85, 211, 0.5);
}

.slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: linear-gradient(135deg, rgba(186, 85, 211, 0.8) 0%, rgba(147, 112, 219, 0.8) 100%);
  border: 2px solid rgba(255, 255, 255, 0.3);
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  transition: all 0.2s;
}

.slider::-moz-range-thumb:hover {
  transform: scale(1.2);
  box-shadow: 0 4px 12px rgba(186, 85, 211, 0.5);
}

.editor-actions {
  display: flex;
  gap: 12px;
  margin-top: 32px;
}

.btn-secondary {
  background: rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.9);
  border: 1px solid rgba(255, 255, 255, 0.2);
  flex: 1;
}

.btn-secondary:hover {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.3);
}

.modal-enter-active, .modal-leave-active {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.modal-enter-from, .modal-leave-to {
  opacity: 0;
}

.modal-enter-from .modal-content, .modal-leave-to .modal-content {
  transform: scale(0.9) translateY(-20px);
}

.modal-enter-to .modal-content, .modal-leave-from .modal-content {
  transform: scale(1) translateY(0);
}

/* Кнопка портфолио */
.portfolio-link {
  position: fixed;
  bottom: 30px;
  right: 30px;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  display: block;
  cursor: pointer;
  transition: all 0.3s ease;
  text-decoration: none;
  overflow: hidden;
  border: 2px solid rgba(184, 165, 255, 0.3);
  background-image: url('https://drive.google.com/thumbnail?id=1e-zV7dZKgUXv96QRAvwxcUrQ7ADRSGei&sz=w200');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  z-index: 1000;
  box-shadow: 
    0 4px 20px rgba(0, 0, 0, 0.3),
    inset 0 1px 0 rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
}

.portfolio-link:hover {
  transform: scale(1.1);
  border-color: rgba(184, 165, 255, 0.6);
  filter: brightness(1.2);
  box-shadow: 
    0 6px 30px rgba(184, 165, 255, 0.4),
    inset 0 1px 0 rgba(255, 255, 255, 0.15);
}

.portfolio-link:active {
  transform: scale(0.95);
}

@media (max-width: 768px) {
  .portfolio-link {
    width: 45px;
    height: 45px;
    bottom: 20px;
    right: 20px;
  }
}

@media (max-width: 480px) {
  .portfolio-link {
    width: 40px;
    height: 40px;
    bottom: 15px;
    right: 15px;
  }
}

@media (max-width: 375px) {
  .portfolio-link {
    width: 36px;
    height: 36px;
    bottom: 15px;
    right: 15px;
  }
}
</style>

