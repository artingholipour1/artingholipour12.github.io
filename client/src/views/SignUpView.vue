<template>
  <div class="auth-screen">
    <div class="utility-buttons-top">
      <button class="source-button" @click="openSourceCode">Source</button>
      <button class="guide-button" @click="openGuide">Guide</button>
    </div>
    <div class="auth-wrapper">
      <router-link to="/" class="back-link">← Back to Home</router-link>
      <div class="auth-container">
        <h1>Sign Up</h1>
        
        <form @submit.prevent="handleSubmit" class="auth-form">
          <div class="form-group">
            <label for="username">Email Address</label>
            <input 
              type="text" 
              id="username" 
              v-model="username" 
              required
              autocomplete="username"
            />
          </div>
          
          <div class="form-group">
            <label for="password">Password</label>
            <input 
              type="password" 
              id="password" 
              v-model="password" 
              required
              autocomplete="new-password"
              @input="validatePassword"
            />
            <div class="password-requirements">
              <div class="requirement" :class="{ met: passwordValidation.length }">
                ✓ At least 8 characters
              </div>
              <div class="requirement" :class="{ met: passwordValidation.lowercase }">
                ✓ Contains lowercase letter
              </div>
              <div class="requirement" :class="{ met: passwordValidation.uppercase }">
                ✓ Contains uppercase letter
              </div>
              <div class="requirement" :class="{ met: passwordValidation.number }">
                ✓ Contains number
              </div>
              <div class="requirement" :class="{ met: passwordValidation.symbol }">
                ✓ Contains symbol
              </div>
            </div>
          </div>

          <div v-if="error" class="error-message">
            {{ error }}
          </div>

          <button 
            type="submit" 
            class="submit-button"
            :disabled="!isFormValid"
            :class="{ 'disabled': !isFormValid }"
          >
            Sign Up
          </button>

          <router-link to="/signin" class="toggle-button">
            Already have an account? Login
          </router-link>
        </form>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onUnmounted, onMounted, computed } from 'vue'
import { useGameStore } from '../stores/game'
import { useRouter } from 'vue-router'

const gameStore = useGameStore()
const router = useRouter()
const username = ref('')
const password = ref('')
const error = ref('')

// Store listener IDs
let successListenerId: string | null = null;
let failureListenerId: string | null = null;

// Setup event listeners
const setupListeners = () => {
  // Remove any existing listeners first
  removeListeners();
  
  successListenerId = gameStore.addEventListener('signup_success', (data) => {
    if (data && data.userId) {
      gameStore.userId = data.userId
    }
    router.push('/play')
    removeListeners();
  })
  
  failureListenerId = gameStore.addEventListener('signup_failure', (data) => {
    error.value = data || 'Authentication failed'
    removeListeners();
  })
}

// Cleanup event listeners
const removeListeners = () => {
  if (successListenerId) {
    gameStore.removeEventListener('signup_success', successListenerId)
    successListenerId = null;
  }
  if (failureListenerId) {
    gameStore.removeEventListener('signup_failure', failureListenerId)
    failureListenerId = null;
  }
}

const passwordValidation = ref({
  length: false,
  lowercase: false,
  uppercase: false,
  number: false,
  symbol: false
})

const openSourceCode = () => {
  window.open('https://github.com/kirodotdev/spirit-of-kiro/', '_blank')
}

const openGuide = () => {
  window.open('https://kiro.dev/docs/guides/learn-by-playing/', '_blank')
}

const validatePassword = () => {
  const pass = password.value
  passwordValidation.value = {
    length: pass.length >= 8,
    lowercase: /[a-z]/.test(pass),
    uppercase: /[A-Z]/.test(pass),
    number: /[0-9]/.test(pass),
    symbol: /[!@#$%^&*(),.?":{}|<>]/.test(pass)
  }
}

const handleSubmit = async () => {
  error.value = ''
  try {
    // Check if password meets all requirements
    if (!Object.values(passwordValidation.value).every(Boolean)) {
      error.value = 'Password does not meet all requirements'
      return
    }

    if (!gameStore.ws) {
      // Try to reconnect if WebSocket is not available
      gameStore.reconnect()
      throw new Error('No WebSocket connection available. Attempting to reconnect...')
    }

    // Setup listeners before sending message
    setupListeners();

    const message = {
      type: 'signup',
      body: {
        username: username.value,
        password: password.value
      }
    }

    // Send authentication message
    gameStore.ws.send(JSON.stringify(message))
  } catch (e: any) {
    error.value = e.message || 'An error occurred'
  }
}

// Setup connection status listeners
let connectionListenerId: string | null = null;
let reconnectFailedId: string | null = null;

onMounted(() => {
  // Listen for connection status changes
  connectionListenerId = gameStore.addEventListener('reconnect-attempt', (data) => {
    error.value = `Connection lost. Reconnecting... (${data.attempt}/${data.maxAttempts})`;
  });
  
  reconnectFailedId = gameStore.addEventListener('reconnect-failed', () => {
    error.value = 'Failed to reconnect. Please try again later.';
  });
});

// Cleanup listeners when component is unmounted
onUnmounted(() => {
  removeListeners();
  
  // Clean up connection status listeners
  if (connectionListenerId) {
    gameStore.removeEventListener('reconnect-attempt', connectionListenerId);
  }
  
  if (reconnectFailedId) {
    gameStore.removeEventListener('reconnect-failed', reconnectFailedId);
  }
})

const isFormValid = computed(() => {
  return username.value.length > 0 && 
         Object.values(passwordValidation.value).every(Boolean)
})
</script>

<style scoped>
.auth-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.9);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.auth-wrapper {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 400px;
}

.auth-container {
  position: relative;
  background: #1a1a1a;
  padding: 2rem;
  border-radius: 8px;
  width: 100%;
  max-width: 400px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.5);
}

h1 {
  color: #fff;
  text-align: center;
  margin-bottom: 2rem;
}

.auth-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

label {
  color: #fff;
  font-size: 0.9rem;
}

input {
  padding: 0.75rem;
  border-radius: 4px;
  border: 1px solid #333;
  background: #222;
  color: #fff;
  font-size: 1rem;
}

input:focus {
  outline: none;
  border-color: #4CAF50;
}

.submit-button {
  padding: 0.75rem;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.3s;
}

.submit-button:hover {
  background: #45a049;
}

.submit-button.disabled {
  background: #666;
  cursor: not-allowed;
  opacity: 0.7;
}

.submit-button.disabled:hover {
  background: #666;
}

.toggle-button {
  background: none;
  border: none;
  color: #4CAF50;
  cursor: pointer;
  padding: 0.5rem;
  font-size: 0.9rem;
  text-decoration: none;
  text-align: center;
}

.toggle-button:hover {
  text-decoration: underline;
}

.error-message {
  color: #ff4444;
  text-align: center;
  font-size: 0.9rem;
  margin-top: 0.5rem;
}

.back-link {
  color: #4CAF50;
  text-decoration: none;
  font-size: 0.9rem;
  transition: color 0.3s;
  margin-bottom: 1rem;
  align-self: flex-start;
}

.back-link:hover {
  color: #45a049;
}

.password-requirements {
  margin-top: 0.5rem;
  font-size: 0.8rem;
  color: #666;
}

.requirement {
  margin: 0.2rem 0;
  color: #666;
}

.requirement.met {
  color: #4CAF50;
}

.utility-buttons-top {
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  gap: 10px;
  z-index: 1001;
}

.source-button {
  padding: 8px 16px;
  background-color: #2196F3;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background-color 0.3s;
}

.source-button:hover {
  background-color: #1976D2;
}

.guide-button {
  padding: 8px 16px;
  background-color: #FF9800;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background-color 0.3s;
}

.guide-button:hover {
  background-color: #F57C00;
}
</style> 