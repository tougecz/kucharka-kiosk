<template>
  <div id="app" :class="{ 'dimmed': !userPresent }">
    
    <div v-if="!userPresent" class="dashboard" @click="userPresent = true">
      <p>{{ monthName.toUpperCase() }}</p>
      <hr>
      <p>19</p>
      <p>{{ nameDay }}</p>
      <div>
        <span>{{ today }} / {{ daysInMonth.length }}</span>
        <span>{{ weekdayName }}</span>
      </div>
      <hr>
      <div class="month-bars" role="list">
        <div
          v-for="day in daysInMonth"
          :key="day"
          :class="['day-bar', { past: day <= today, today: day === today }]"
          role="listitem"
        >
          <div class="bar-fill"></div>
        </div>
      </div>
      <div class="forecast">
        <div v-if="forecast.length === 0">Načítám počasí…</div>
        <div v-else class="forecast-list">
          <div v-for="day in forecast" :key="day.time" class="forecast-item">
            <div class="day-name">{{ day.label }}</div>
            <div class="weather-emoji">{{ day.emoji }}</div>
            <div class="temps"><span class="max">{{ day.temp_max }}°</span> / <span class="min">{{ day.temp_min }}°</span></div>
          </div>
        </div>
      </div>
      <div class="events">
        <img src="/src/assets/images/box-down.svg">
        <img src="/src/assets/images/box-bum.svg">
        <img src="/src/assets/images/portal-down.svg">
        <img src="/src/assets/images/portal-up.svg">
        <img src="/src/assets/images/cake.svg">
      </div>
      
    </div>

    <div v-else class="kucharka">
      <button @click="userPresent = false">dashboard</button>
      <header>
        <h1>Moje Kuchařka</h1>
        <div class="status-bar">
          <span v-if="battery">🔋 {{ battery }}%</span>
          <span>{{ currentTime }}</span>
        </div>
      </header>

      <div class="recipe-grid">
        <div 
          v-for="recept in recipes" 
          :key="recept.id" 
          class="recipe-card"
          @click="openRecipe(recept)"
        >
          <div class="recipe-icon">🍳</div>
          <h3>{{ recept.nazev }}</h3>
          <span class="tag">{{ recept.kategorie }}</span>
        </div>
      </div>

      <div v-if="selectedRecipe" class="modal-overlay" @click.self="closeRecipe">
        <div class="modal-content">
          <button class="close-btn" @click="closeRecipe"> zavřít ✖ </button>
          <h2>{{ selectedRecipe.nazev }}</h2>
          
          <div class="recipe-body">
            <div class="ingredients">
              <h4>Ingredience</h4>
              <ul>
                <li v-for="ing in selectedRecipe.ingredience" :key="ing">{{ ing }}</li>
              </ul>
            </div>
            <div class="instructions">
              <h4>Postup</h4>
              <ol>
                <li v-for="(krok, index) in selectedRecipe.postup" :key="index" class="step-item">
                  {{ krok }}
                </li>
              </ol>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

// --- REAKTIVNÍ PROMĚNNÉ ---
const battery = ref(null)
const userPresent = ref(true)
const currentTime = ref(new Date().toLocaleTimeString('cs-CZ'))
const recipes = ref([])
const selectedRecipe = ref(null) // Pro zobrazení detailu receptu

// Month bars (screensaver)
const daysInMonth = ref([])
const today = ref(new Date().getDate())
// Current month name and weekday (in Czech)
const monthName = ref('')
const weekdayName = ref('')
// Today's name day
const nameDay = ref('')
// Track last date to avoid repeated fetches
let _lastDayKey = ''

// --- POČASÍ (5 dní) ---
const forecast = ref([])

const weatherEmoji = (code) => {
  // zjednodušené mapování WMO weathercode -> emoji
  if (code === 0) return '☀️'
  if (code === 1 || code === 2) return '⛅'
  if (code === 3) return '☁️'
  if (code === 45 || code === 48) return '🌫️'
  if (code >= 51 && code <= 67) return '🌧️'
  if (code >= 71 && code <= 77) return '❄️'
  if (code >= 80 && code <= 82) return '🌦️'
  if (code >= 95) return '⛈️'
  return '🔆'
}

const fetchWeather = async () => {
  try {
    const url = 'https://api.open-meteo.com/v1/forecast?latitude=50.08&longitude=14.43&daily=weathercode,temperature_2m_max,temperature_2m_min&timezone=Europe/Prague'
    const res = await fetch(url)
    if (!res.ok) throw new Error('Failed to fetch weather')
    const json = await res.json()
    const d = json.daily
    if (d && d.time && d.time.length) {
      const items = d.time.map((t, i) => {
        const tm = new Date(t)
        const label = tm.toLocaleDateString('cs-CZ', { weekday: 'short', day: 'numeric', month: 'numeric' })
        return {
          time: t,
          label,
          weathercode: d.weathercode[i],
          temp_max: Math.round((d.temperature_2m_max[i] + Number.EPSILON) * 10) / 10,
          temp_min: Math.round((d.temperature_2m_min[i] + Number.EPSILON) * 10) / 10,
          emoji: weatherEmoji(d.weathercode[i])
        }
      })
      // vezmeme první 5 dní (dnes + 4 další)
      forecast.value = items.slice(0, 5)
    }
  } catch (e) {
    console.error('Chyba při načítání počasí', e)
    forecast.value = []
  }
}

// Fetch today's nameday from the Abalin API (CZ field)
const updateNameDay = async (now) => {
  const dayKey = `${now.getFullYear()}-${now.getMonth()+1}-${now.getDate()}`
  try {
    const res = await fetch('https://nameday.abalin.net/api/V2/today/Prague')
    if (!res.ok) throw new Error('Failed to fetch nameday')
    const json = await res.json()
    const cz = json && json.data && (json.data.cz || json.data.CZ)
    nameDay.value = cz || '—'
  } catch (e) {
    nameDay.value = '—'
  }
  _lastDayKey = dayKey
}

const updateMonthDays = () => {
  const now = new Date()
  const year = now.getFullYear()
  const month = now.getMonth()
  const daysCount = new Date(year, month + 1, 0).getDate()
  daysInMonth.value = Array.from({ length: daysCount }, (_, i) => i + 1)
  today.value = now.getDate()
  monthName.value = now.toLocaleString('cs-CZ', { month: 'long' })
  weekdayName.value = now.toLocaleString('cs-CZ', { weekday: 'long' })
  const dayKey = `${now.getFullYear()}-${now.getMonth()+1}-${now.getDate()}`
  if (dayKey !== _lastDayKey) {
    _lastDayKey = dayKey
    updateNameDay(now)
  }
}

// --- LOGIKA HODIN ---
const updateTime = () => {
  currentTime.value = new Date().toLocaleTimeString('cs-CZ')
  updateMonthDays()
}

// --- LOGIKA GOOGLE DRIVE ---
const fetchRecipes = async () => {
  // SEM VLOŽ SVÉ ID SOUBORU
  const url = `https://script.google.com/macros/s/AKfycbzA7qsQStIfSlnEOEunFb5Ldh3NZvzWgHcdezDApzXrevzzF_-09lRo1iza0knvdmnLkA/exec`
  
  try {
    const response = await fetch(url)
    if (!response.ok) throw new Error('Chyba při stahování JSONu')
    const data = await response.json()
    recipes.value = data
    console.log('Recepty úspěšně načteny:', data)
  } catch (error) {
    console.error('Chyba fetchování:', error)
  }
}

// --- LOGIKA SCREENSAVERU (IDLE TIMER) ---
let idleTimer
const startIdleTimer = () => {
  clearTimeout(idleTimer)
  // Po 15 sekundách nečinnosti přepne na hodiny
  idleTimer = setTimeout(() => {
    userPresent.value = false
    selectedRecipe.value = null // Při odchodu zavře i otevřený recept
  }, 15000) 
}

// --- FUNKCE PRO RECEPTY ---
const openRecipe = (recipe) => {
  selectedRecipe.value = recipe
  startIdleTimer() // Resetujeme časovač při interakci
}

const closeRecipe = () => {
  selectedRecipe.value = null
  startIdleTimer()
}

// --- ON MOUNTED (SPOUŠTĚNÍ PŘI STARTU) ---
onMounted(() => {
  // 1. Spustíme vteřinový interval pro hodiny
  const clockInterval = setInterval(updateTime, 1000)

  // 2. Načteme recepty z Drive
  fetchRecipes()
  // 2b. Načteme počasí (dnes + následujících 4 dny)
  fetchWeather()

  // 3. Rozhraní pro Fully Kiosk
  if (typeof fully !== 'undefined') {
    // Zjistíme stav baterie
    battery.value = fully.getBatteryLevel()
    
    // Nastavíme posluchač na pohyb
    window.addEventListener('fully.motionDetector', () => {
      userPresent.value = true
      startIdleTimer()
    })
  }

  // 4. Spustíme odpočet pro první schování (pro test na PC)
  startIdleTimer()

  // Úklid při zničení komponenty
  onUnmounted(() => {
    clearInterval(clockInterval)
    clearTimeout(idleTimer)
  })
})
</script>

<style>
:root {
  --bg-dark: #0f0f0f;
  --card-bg: #1e1e1e;
  --accent: #f39c12;
  --text: #1b1b24;
}

html {
  height: 100%;
  width: 100%;
  background-color: white;
}

body {
  width: 504px;
  height: 100%;
  background-color: rgb(14, 14, 37);
  margin: 0;
  color: rgb(236, 236, 236);
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  user-select: none;
  padding: 20px;
  box-sizing: border-box;
}

.dashboard {
  height: 920px;
}

p {
  margin: 0;
}

.month-bars {
  display: flex;
  height: 20px;
  gap: 4px;
}

.day-bar {
  width: 4px;
  height: 20px;
  background-color: gray;
}

.past {
  background-color: rgb(236, 236, 236);
}

.events {
  margin-top: 20px;
  display: flex;
  gap: 21px;

  img {
    width:76px;
    height: 76px;
    filter: brightness(0) saturate(100%) invert(100%) sepia(63%) saturate(34%) hue-rotate(308deg) brightness(118%) contrast(85%);
  }
}

.forecast {
  margin-top: 18px;
}
.forecast-list {
  display: flex;
  gap: 12px;
  align-items: center;
}
.forecast-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: rgba(255,255,255,0.04);
  padding: 8px;
  border-radius: 6px;
  min-width: 64px;
}
.forecast-item .day-name {
  font-size: 12px;
  opacity: 0.9;
}
.forecast-item .weather-emoji {
  font-size: 22px;
  margin: 6px 0;
}
.forecast-item .temps { font-size: 13px }

</style>