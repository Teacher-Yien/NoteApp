<script setup lang="ts">
import { ref, onMounted } from 'vue'
import Login from './components/Login.vue'
import MainApp from './components/MainApp.vue'

// Authentication state
const isAuthenticated = ref<boolean>(false)
const user = ref<any>(null)
const loading = ref<boolean>(true)

// Check if user is already authenticated on app load
onMounted(() => {
  checkAuthStatus()
})

const checkAuthStatus = (): void => {
  const token = localStorage.getItem('authToken')
  const userData = localStorage.getItem('userData')
  
  if (token && userData) {
    try {
      user.value = JSON.parse(userData)
      isAuthenticated.value = true
    } catch (error) {
      console.error('Error parsing user data:', error)
      logout()
    }
  }
  
  loading.value = false
}

const handleLoginSuccess = (userData: any): void => {
  user.value = userData
  isAuthenticated.value = true
  
  // Store user data for persistence
  localStorage.setItem('userData', JSON.stringify(userData))
  
  console.log('Login successful, user data:', userData)
}

const logout = (): void => {
  isAuthenticated.value = false
  user.value = null
  
  // Clear stored data
  localStorage.removeItem('authToken')
  localStorage.removeItem('userData')
  
  console.log('User logged out')
}
</script>

<template>
  <div>
    <!-- Loading state -->
    <div v-if="loading" class="min-h-screen bg-gradient-to-br from-purple-600 via-purple-700 to-purple-800 flex items-center justify-center">
      <div class="text-white text-xl">Loading...</div>
    </div>
    
    <!-- Show Login component if not authenticated -->
    <Login 
      v-else-if="!isAuthenticated" 
      @login-success="handleLoginSuccess"
    />
    
    <!-- Show MainApp component if authenticated -->
    <MainApp 
      v-else 
      :user="user"
      @logout="logout"
    />
  </div>
</template>

<style scoped>
</style>