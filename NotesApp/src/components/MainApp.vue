<script setup lang="ts">
import { ref, onMounted } from 'vue'

// Props
interface Props {
  user: {
    email: string
    fullName?: string
    token: string
  }
}

const props = defineProps<Props>()

// Emits
const emit = defineEmits<{
  logout: []
}>()

// Reactive state
const notes = ref<Array<{
  id: string
  title: string
  content: string
  createdAt: string
  category: string
}>>([])

const loading = ref<boolean>(false)
const showCreateNote = ref<boolean>(false)

// Sample data
const sampleNotes = [
  {
    id: '1',
    title: 'Welcome to Your Enhanced Notes!',
    content: 'This is your first note. You can create, edit, and organize your thoughts here.',
    createdAt: new Date().toISOString(),
    category: 'Personal'
  },
  {
    id: '2',
    title: 'Project Ideas & Brainstorming',
    content: 'Great ideas start with great notes. Keep track of your inspirations here.',
    createdAt: new Date(Date.now() - 86400000).toISOString(), // Yesterday
    category: 'Work'
  },
  {
    id: '3',
    title: 'Learning Notes',
    content: 'Document your learning journey and key insights.',
    createdAt: new Date(Date.now() - 172800000).toISOString(), // 2 days ago
    category: 'Learning'
  }
]

// Methods
const handleLogout = (): void => {
  if (confirm('Are you sure you want to logout?')) {
    emit('logout')
  }
}

const loadNotes = async (): Promise<void> => {
  loading.value = true
  try {
    // Here you would fetch notes from your API
    // For now, we'll use sample data
    await new Promise(resolve => setTimeout(resolve, 1000)) // Simulate API call
    notes.value = sampleNotes
  } catch (error) {
    console.error('Error loading notes:', error)
  } finally {
    loading.value = false
  }
}

const createNote = (): void => {
  showCreateNote.value = true
}

const formatDate = (dateString: string): string => {
  const date = new Date(dateString)
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'short',
    day: 'numeric'
  })
}

const getCategoryColor = (category: string): string => {
  const colors: Record<string, string> = {
    'Personal': 'bg-blue-100 text-blue-800',
    'Work': 'bg-green-100 text-green-800',
    'Learning': 'bg-purple-100 text-purple-800',
    'Ideas': 'bg-yellow-100 text-yellow-800'
  }
  return colors[category] || 'bg-gray-100 text-gray-800'
}

// Lifecycle
onMounted(() => {
  loadNotes()
})
</script>

<template>
  <div class="min-h-screen bg-gray-50">
    <!-- Header -->
    <header class="bg-white shadow-sm border-b">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex justify-between items-center h-16">
          <div class="flex items-center">
            <h1 class="text-2xl font-bold text-gray-900">Notes App</h1>
          </div>
          
          <div class="flex items-center space-x-4">
            <span class="text-sm text-gray-700">
              Welcome, {{ props.user.fullName || props.user.email }}!
            </span>
            <button
              @click="handleLogout"
              class="bg-red-600 hover:bg-red-700 text-white px-4 py-2 rounded-lg text-sm font-medium transition-colors"
            >
              Logout
            </button>
          </div>
        </div>
      </div>
    </header>

    <!-- Main Content -->
    <main class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
      <!-- Action Bar -->
      <div class="flex justify-between items-center mb-8">
        <div>
          <h2 class="text-3xl font-bold text-gray-900">Your Notes</h2>
          <p class="text-gray-600 mt-1">Organize your thoughts, simplify your life</p>
        </div>
        
        <button
          @click="createNote"
          class="bg-purple-600 hover:bg-purple-700 text-white px-6 py-3 rounded-xl font-medium transition-all duration-200 transform hover:scale-105 shadow-lg"
        >
          + Create Note
        </button>
      </div>

      <!-- Loading State -->
      <div v-if="loading" class="flex justify-center items-center py-16">
        <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-purple-600"></div>
      </div>

      <!-- Notes Grid -->
      <div v-else-if="notes.length > 0" class="grid gap-6 md:grid-cols-2 lg:grid-cols-3">
        <div
          v-for="note in notes"
          :key="note.id"
          class="bg-white rounded-xl shadow-sm border border-gray-200 p-6 hover:shadow-md transition-shadow cursor-pointer"
        >
          <div class="flex justify-between items-start mb-4">
            <h3 class="text-lg font-semibold text-gray-900 line-clamp-2">
              {{ note.title }}
            </h3>
            <span
              :class="[
                'px-2 py-1 rounded-full text-xs font-medium',
                getCategoryColor(note.category)
              ]"
            >
              {{ note.category }}
            </span>
          </div>
          
          <p class="text-gray-600 text-sm mb-4 line-clamp-3">
            {{ note.content }}
          </p>
          
          <div class="flex justify-between items-center text-xs text-gray-500">
            <span>{{ formatDate(note.createdAt) }}</span>
            <div class="flex space-x-2">
              <button class="hover:text-purple-600 transition-colors">Edit</button>
              <button class="hover:text-red-600 transition-colors">Delete</button>
            </div>
          </div>
        </div>
      </div>

      <!-- Empty State -->
      <div v-else class="text-center py-16">
        <div class="max-w-md mx-auto">
          <svg class="mx-auto h-24 w-24 text-gray-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"></path>
          </svg>
          <h3 class="mt-4 text-lg font-medium text-gray-900">No notes yet</h3>
          <p class="mt-2 text-gray-500">Get started by creating your first note.</p>
          <button
            @click="createNote"
            class="mt-6 bg-purple-600 hover:bg-purple-700 text-white px-6 py-3 rounded-xl font-medium transition-colors"
          >
            Create Your First Note
          </button>
        </div>
      </div>
    </main>

    <!-- Create Note Modal (placeholder) -->
    <div
      v-if="showCreateNote"
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50"
      @click="showCreateNote = false"
    >
      <div
        class="bg-white rounded-2xl p-8 max-w-2xl w-full max-h-[90vh] overflow-y-auto"
        @click.stop
      >
        <div class="flex justify-between items-center mb-6">
          <h3 class="text-2xl font-bold text-gray-900">Create New Note</h3>
          <button
            @click="showCreateNote = false"
            class="text-gray-400 hover:text-gray-600 transition-colors"
          >
            <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
            </svg>
          </button>
        </div>
        
        <div class="space-y-4">
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-2">Title</label>
            <input
              type="text"
              placeholder="Enter note title..."
              class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-purple-500 focus:border-transparent"
            >
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-2">Category</label>
            <select class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-purple-500 focus:border-transparent">
              <option>Personal</option>
              <option>Work</option>
              <option>Learning</option>
              <option>Ideas</option>
            </select>
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-2">Content</label>
            <textarea
              rows="8"
              placeholder="Write your note content here..."
              class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-purple-500 focus:border-transparent resize-none"
            ></textarea>
          </div>
        </div>

        <div class="flex justify-end space-x-4 mt-8">
          <button
            @click="showCreateNote = false"
            class="px-6 py-3 border border-gray-300 rounded-xl text-gray-700 hover:bg-gray-50 transition-colors"
          >
            Cancel
          </button>
          <button
            class="px-6 py-3 bg-purple-600 hover:bg-purple-700 text-white rounded-xl transition-colors"
          >
            Create Note
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.line-clamp-2 {
  display: -webkit-box;
  --webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.line-clamp-3 {
  display: -webkit-box;
  --webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
</style>