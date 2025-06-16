<script setup lang="ts">
import { ref } from 'vue'

// Types
interface LoginForm {
  email: string
  password: string
  rememberMe: boolean
}

interface SignUpForm {
  fullName: string
  email: string
  password: string
}

type ActiveTab = 'signin' | 'signup'
type SocialProvider = 'google' | 'twitter'

// Reactive state
const activeTab = ref<ActiveTab>('signin')
const showPassword = ref<boolean>(false)
const loading = ref<boolean>(false)
const error = ref<string>('')

// Form data
const loginForm = ref<LoginForm>({
  email: '',
  password: '',
  rememberMe: false
})

const signUpForm = ref<SignUpForm>({
  fullName: '',
  email: '',
  password: ''
})

// API Base URL - Update this to match your backend
const API_BASE_URL = 'http://localhost:5077/api'

// Utility function for API calls
const apiCall = async (endpoint: string, options: RequestInit = {}) => {
  try {
    const response = await fetch(`${API_BASE_URL}${endpoint}`, {
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        ...options.headers
      },
      ...options
    })

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`)
    }

    return await response.json()
  } catch (err) {
    console.error('API call failed:', err)
    throw err
  }
}

// Define emits
const emit = defineEmits<{
  'login-success': [userData: any]
}>()

// Methods
const signIn = async (): Promise<void> => {
  if (!loginForm.value.email || !loginForm.value.password) {
    error.value = 'Please fill in all required fields'
    return
  }

  loading.value = true
  error.value = ''

  try {
    const response = await apiCall('/Auth/login', {
      method: 'POST',
      body: JSON.stringify({
        email: loginForm.value.email,
        password: loginForm.value.password,
        rememberMe: loginForm.value.rememberMe
      })
    })

    console.log('Sign in successful:', response)
    
    // Store token if provided
    if (response.token) {
      localStorage.setItem('authToken', response.token)
    }
    
    // Emit login success event to parent component
    emit('login-success', {
      email: loginForm.value.email,
      token: response.token,
      ...response.user // Include any other user data from response
    })
    
  } catch (err) {
    error.value = 'Invalid email or password. Please try again.'
    console.error('Sign in error:', err)
  } finally {
    loading.value = false
  }
}

const signUp = async (): Promise<void> => {
  if (!signUpForm.value.fullName || !signUpForm.value.email || !signUpForm.value.password) {
    error.value = 'Please fill in all required fields'
    return
  }

  if (signUpForm.value.password.length < 6) {
    error.value = 'Password must be at least 6 characters long'
    return
  }

  loading.value = true
  error.value = ''

  try {
    const response = await apiCall('/Auth/register', {
      method: 'POST',
      body: JSON.stringify({
        fullName: signUpForm.value.fullName,
        email: signUpForm.value.email,
        password: signUpForm.value.password
      })
    })

    console.log('Sign up successful:', response)
    
    // Option 1: Auto-login after registration
    if (response.token) {
      localStorage.setItem('authToken', response.token)
      emit('login-success', {
        email: signUpForm.value.email,
        fullName: signUpForm.value.fullName,
        token: response.token,
        ...response.user
      })
    } else {
      // Option 2: Show success message and switch to sign in
      alert('Account created successfully! Please sign in.')
      switchTab('signin')
    }
    
  } catch (err) {
    error.value = 'Registration failed. Please try again.'
    console.error('Sign up error:', err)
  } finally {
    loading.value = false
  }
}

const socialLogin = (provider: SocialProvider): void => {
  loading.value = true
  error.value = ''

  try {
    // Redirect to social auth endpoint
    window.location.href = `${API_BASE_URL}/Auth/${provider}`
  } catch (err) {
    error.value = `${provider} login failed. Please try again.`
    console.error('Social login error:', err)
    loading.value = false
  }
}

const togglePasswordVisibility = (): void => {
  showPassword.value = !showPassword.value
}

const switchTab = (tab: ActiveTab): void => {
  activeTab.value = tab
  showPassword.value = false
  error.value = ''
}

// Clear error when user starts typing
const clearError = (): void => {
  if (error.value) {
    error.value = ''
  }
}
</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-purple-600 via-purple-700 to-purple-800">
    <div class="min-h-screen flex flex-col items-center justify-center px-4">
      <!-- Header -->
      <div class="text-center mb-8">
        <h1 class="text-4xl md:text-5xl font-bold text-white mb-2">Notes App</h1>
        <p class="text-purple-200 text-lg">Organize your thoughts, simplify your life</p>
      </div>

      <!-- Login Card -->
      <div class="w-full max-w-md bg-white/10 backdrop-blur-lg rounded-3xl p-8 shadow-2xl border border-white/20">
        <!-- Error Message -->
        <div v-if="error" class="mb-6 p-4 bg-red-500/20 border border-red-500/30 rounded-2xl">
          <p class="text-red-200 text-sm">{{ error }}</p>
        </div>

        <!-- Tab Navigation -->
        <div class="flex mb-8">
          <button 
            @click="switchTab('signin')"
            :class="[
              'flex-1 py-3 px-6 rounded-full font-medium transition-all duration-200 mr-2',
              activeTab === 'signin' ? 'bg-white text-purple-700' : 'text-white hover:bg-white/10'
            ]"
            :disabled="loading"
          >
            Sign In
          </button>
          <button 
            @click="switchTab('signup')"
            :class="[
              'flex-1 py-3 px-6 rounded-full font-medium transition-all duration-200',
              activeTab === 'signup' ? 'bg-white text-purple-700' : 'text-white hover:bg-white/10'
            ]"
            :disabled="loading"
          >
            Sign Up
          </button>
        </div>

        <!-- Welcome Message -->
        <div class="text-center mb-8">
          <h2 class="text-2xl font-bold text-white">
            {{ activeTab === 'signin' ? 'Welcome Back!' : 'Create Account' }}
          </h2>
        </div>

        <!-- Sign In Form -->
        <form v-if="activeTab === 'signin'" @submit.prevent="signIn" class="space-y-6">
          <div>
            <input 
              v-model="loginForm.email"
              type="email" 
              placeholder="Email address"
              class="w-full px-4 py-4 bg-white/10 border border-white/20 rounded-2xl text-white placeholder-white/70 focus:outline-none focus:ring-2 focus:ring-white/50 focus:border-transparent transition-all duration-200"
              required
              :disabled="loading"
              @input="clearError"
            >
          </div>
          
          <div class="relative">
            <input 
              v-model="loginForm.password"
              :type="showPassword ? 'text' : 'password'"
              placeholder="Password"
              class="w-full px-4 py-4 bg-white/10 border border-white/20 rounded-2xl text-white placeholder-white/70 focus:outline-none focus:ring-2 focus:ring-white/50 focus:border-transparent transition-all duration-200 pr-12"
              required
              :disabled="loading"
              @input="clearError"
            >
            <button 
              type="button"
              @click="togglePasswordVisibility"
              class="absolute right-4 top-1/2 transform -translate-y-1/2 text-white/70 hover:text-white transition-colors"
              :disabled="loading"
            >
              <!-- Eye Open Icon -->
              <svg v-if="!showPassword" class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path>
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"></path>
              </svg>
              <!-- Eye Closed Icon -->
              <svg v-else class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13.875 18.825A10.05 10.05 0 0112 19c-4.478 0-8.268-2.943-9.543-7a9.97 9.97 0 011.563-3.029m5.858.908a3 3 0 114.243 4.243M9.878 9.878l4.242 4.242M9.878 9.878L3 3m6.878 6.878L21 21"></path>
              </svg>
            </button>
          </div>

          <div class="flex items-center justify-between">
            <label class="flex items-center text-white/80 cursor-pointer">
              <input 
                v-model="loginForm.rememberMe"
                type="checkbox" 
                class="mr-2 rounded border-white/30 bg-white/10 text-purple-600 focus:ring-purple-500 focus:ring-offset-0"
                :disabled="loading"
              >
              Remember me
            </label>
            <a href="#" class="text-white/80 hover:text-white transition-colors text-sm">
              Forgot password?
            </a>
          </div>

          <button 
            type="submit"
            class="w-full bg-white text-purple-700 py-4 rounded-2xl font-semibold hover:bg-white/90 transition-all duration-200 transform hover:scale-[1.02] active:scale-[0.98] disabled:opacity-50 disabled:cursor-not-allowed disabled:transform-none"
            :disabled="loading"
          >
            {{ loading ? 'Signing In...' : 'Sign In' }}
          </button>
        </form>

        <!-- Sign Up Form -->
        <form v-else @submit.prevent="signUp" class="space-y-6">
          <div>
            <input 
              v-model="signUpForm.fullName"
              type="text" 
              placeholder="Full name"
              class="w-full px-4 py-4 bg-white/10 border border-white/20 rounded-2xl text-white placeholder-white/70 focus:outline-none focus:ring-2 focus:ring-white/50 focus:border-transparent transition-all duration-200"
              required
              :disabled="loading"
              @input="clearError"
            >
          </div>
          
          <div>
            <input 
              v-model="signUpForm.email"
              type="email" 
              placeholder="Email address"
              class="w-full px-4 py-4 bg-white/10 border border-white/20 rounded-2xl text-white placeholder-white/70 focus:outline-none focus:ring-2 focus:ring-white/50 focus:border-transparent transition-all duration-200"
              required
              :disabled="loading"
              @input="clearError"
            >
          </div>
          
          <div class="relative">
            <input 
              v-model="signUpForm.password"
              :type="showPassword ? 'text' : 'password'"
              placeholder="Password (min 6 characters)"
              class="w-full px-4 py-4 bg-white/10 border border-white/20 rounded-2xl text-white placeholder-white/70 focus:outline-none focus:ring-2 focus:ring-white/50 focus:border-transparent transition-all duration-200 pr-12"
              required
              minlength="6"
              :disabled="loading"
              @input="clearError"
            >
            <button 
              type="button"
              @click="togglePasswordVisibility"
              class="absolute right-4 top-1/2 transform -translate-y-1/2 text-white/70 hover:text-white transition-colors"
              :disabled="loading"
            >
              <!-- Eye Open Icon -->
              <svg v-if="!showPassword" class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"></path>
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"></path>
              </svg>
              <!-- Eye Closed Icon -->
              <svg v-else class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13.875 18.825A10.05 10.05 0 0112 19c-4.478 0-8.268-2.943-9.543-7a9.97 9.97 0 011.563-3.029m5.858.908a3 3 0 114.243 4.243M9.878 9.878l4.242 4.242M9.878 9.878L3 3m6.878 6.878L21 21"></path>
              </svg>
            </button>
          </div>

          <button 
            type="submit"
            class="w-full bg-white text-purple-700 py-4 rounded-2xl font-semibold hover:bg-white/90 transition-all duration-200 transform hover:scale-[1.02] active:scale-[0.98] disabled:opacity-50 disabled:cursor-not-allowed disabled:transform-none"
            :disabled="loading"
          >
            {{ loading ? 'Creating Account...' : 'Create Account' }}
          </button>
        </form>

        <!-- Social Login -->
        <div class="mt-8">
          <div class="relative">
            <div class="absolute inset-0 flex items-center">
              <div class="w-full border-t border-white/20"></div>
            </div>
            <div class="relative flex justify-center text-sm">
              <span class="px-2 bg-transparent text-white/70">Or continue with</span>
            </div>
          </div>

          <div class="mt-6 grid grid-cols-2 gap-3">
            <button 
              @click="socialLogin('google')"
              type="button"
              class="flex items-center justify-center px-4 py-3 bg-white/10 border border-white/20 rounded-2xl text-white hover:bg-white/20 transition-all duration-200 transform hover:scale-[1.02] active:scale-[0.98] disabled:opacity-50 disabled:cursor-not-allowed disabled:transform-none"
              :disabled="loading"
            >
              <!-- Google Icon -->
              <svg class="w-5 h-5 mr-2" viewBox="0 0 24 24">
                <path fill="currentColor" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/>
                <path fill="currentColor" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/>
                <path fill="currentColor" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/>
                <path fill="currentColor" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/>
              </svg>
              Google
            </button>
            
            <button 
              @click="socialLogin('twitter')"
              type="button"
              class="flex items-center justify-center px-4 py-3 bg-white/10 border border-white/20 rounded-2xl text-white hover:bg-white/20 transition-all duration-200 transform hover:scale-[1.02] active:scale-[0.98] disabled:opacity-50 disabled:cursor-not-allowed disabled:transform-none"
              :disabled="loading"
            >
              <!-- Twitter Icon -->
              <svg class="w-5 h-5 mr-2" fill="currentColor" viewBox="0 0 24 24">
                <path d="M23.953 4.57a10 10 0 01-2.825.775 4.958 4.958 0 002.163-2.723c-.951.555-2.005.959-3.127 1.184a4.92 4.92 0 00-8.384 4.482C7.69 8.095 4.067 6.13 1.64 3.162a4.822 4.822 0 00-.666 2.475c0 1.71.87 3.213 2.188 4.096a4.904 4.904 0 01-2.228-.616v.06a4.923 4.923 0 003.946 4.827 4.996 4.996 0 01-2.212.085 4.936 4.936 0 004.604 3.417 9.867 9.867 0 01-6.102 2.105c-.39 0-.779-.023-1.17-.067a13.995 13.995 0 007.557 2.209c9.053 0 13.998-7.496 13.998-13.985 0-.21 0-.42-.015-.63A9.935 9.935 0 0024 4.59z"/>
              </svg>
              Twitter
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Custom styles for form inputs */
input[type="checkbox"] {
  appearance: none;
  width: 1rem;
  height: 1rem;
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 0.25rem;
  background-color: rgba(255, 255, 255, 0.1);
  position: relative;
  cursor: pointer;
}

input[type="checkbox"]:checked {
  background-color: rgb(147, 51, 234);
  border-color: rgb(147, 51, 234);
}

input[type="checkbox"]:checked::after {
  content: '✓';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  font-size: 0.75rem;
  font-weight: bold;
}

input[type="checkbox"]:focus {
  outline: none;
  box-shadow: 0 0 0 2px rgba(147, 51, 234, 0.5);
}

input[type="checkbox"]:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Custom scrollbar for the container if needed */
::-webkit-scrollbar {
  width: 6px;
}

::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
}

::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}
</style>