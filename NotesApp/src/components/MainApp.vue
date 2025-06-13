<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import axios from 'axios'

interface Note {
  id: number
  title: string
  content: string
  category: string
  tags: string[]
  isFavorite: boolean
  createdAt: Date
  updatedAt: Date
}

interface ApiNote {
  id: number
  title: string
  content: string
  category: string
  tags: string[]
  isFavorite: boolean
  createdAt: string
  updatedAt: string
}

// API configuration
const API_BASE_URL = 'http://localhost:3000/api'
const api = axios.create({
  baseURL: API_BASE_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
})

// State
const notes = ref<Note[]>([])
const selectedNote = ref<Note | null>(null)
const isCreating = ref(false)
const newNoteTitle = ref('')
const newNoteContent = ref('')
const isLoading = ref(false)
const error = ref<string | null>(null)
const searchQuery = ref('')
const selectedCategory = ref('All Categories')
const sortBy = ref('Newest')
const viewMode = ref('grid') // 'grid' or 'list'
const showFavoritesOnly = ref(false)

// Categories
const categories = ['All Categories', 'Personal', 'Work', 'Ideas', 'Learning']
const sortOptions = ['Newest', 'Oldest', 'A-Z', 'Z-A']

// Computed properties
const totalNotes = computed(() => notes.value.length)
const favoriteNotes = computed(() => notes.value.filter(note => note.isFavorite).length)

const filteredNotes = computed(() => {
  let filtered = notes.value

  // Filter by search query
  if (searchQuery.value.trim()) {
    const query = searchQuery.value.toLowerCase()
    filtered = filtered.filter(note => 
      note.title.toLowerCase().includes(query) ||
      note.content.toLowerCase().includes(query) ||
      note.tags.some(tag => tag.toLowerCase().includes(query))
    )
  }

  // Filter by category
  if (selectedCategory.value !== 'All Categories') {
    filtered = filtered.filter(note => note.category === selectedCategory.value)
  }

  // Filter by favorites
  if (showFavoritesOnly.value) {
    filtered = filtered.filter(note => note.isFavorite)
  }

  // Sort
  switch (sortBy.value) {
    case 'Oldest':
      filtered.sort((a, b) => a.createdAt.getTime() - b.createdAt.getTime())
      break
    case 'A-Z':
      filtered.sort((a, b) => a.title.localeCompare(b.title))
      break
    case 'Z-A':
      filtered.sort((a, b) => b.title.localeCompare(a.title))
      break
    default: // Newest
      filtered.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
  }

  return filtered
})

// Utility functions
const convertApiNote = (apiNote: ApiNote): Note => ({
  ...apiNote,
  createdAt: new Date(apiNote.createdAt),
  updatedAt: new Date(apiNote.updatedAt)
})

const convertToApiNote = (note: Partial<Note>): Partial<ApiNote> => ({
  ...note,
  createdAt: note.createdAt?.toISOString(),
  updatedAt: note.updatedAt?.toISOString()
})

// API functions
const fetchNotes = async () => {
  try {
    isLoading.value = true
    error.value = null
    const response = await api.get<ApiNote[]>('/notes')
    notes.value = response.data.map(convertApiNote)
  } catch (err) {
    error.value = 'Failed to fetch notes'
    console.error('Error fetching notes:', err)
    // Fallback to sample data for demo
    notes.value = [
      {
        id: 1,
        title: 'Welcome to Your Enhanced Notes!',
        content: 'This is your first note with enhanced features. You can now search, filter, and sort your notes!',
        category: 'Personal',
        tags: ['welcome'],
        isFavorite: true,
        createdAt: new Date(Date.now() - 24 * 60 * 60 * 1000),
        updatedAt: new Date(Date.now() - 24 * 60 * 60 * 1000)
      },
      {
        id: 2,
        title: 'Project Ideas & Brainstorming',
        content: 'Collection of innovative project ideas and creative brainstorming sessions.',
        category: 'Work',
        tags: ['brainstorming'],
        isFavorite: false,
        createdAt: new Date(Date.now() - 2 * 24 * 60 * 60 * 1000),
        updatedAt: new Date(Date.now() - 60 * 60 * 1000)
      },
      {
        id: 3,
        title: 'Learning Notes',
        content: 'Important concepts and learnings from various courses and books.',
        category: 'Learning',
        tags: ['education', 'notes'],
        isFavorite: true,
        createdAt: new Date(Date.now() - 3 * 24 * 60 * 60 * 1000),
        updatedAt: new Date(Date.now() - 2 * 60 * 60 * 1000)
      }
    ]
  } finally {
    isLoading.value = false
  }
}

const createNote = async (title: string, content: string) => {
  try {
    isLoading.value = true
    error.value = null
    const newNote = {
      title,
      content,
      category: 'Personal',
      tags: [],
      isFavorite: false,
      createdAt: new Date(),
      updatedAt: new Date()
    }
    const response = await api.post<ApiNote>('/notes', convertToApiNote(newNote))
    const createdNote = convertApiNote(response.data)
    notes.value.unshift(createdNote)
    return createdNote
  } catch (err) {
    error.value = 'Failed to create note'
    console.error('Error creating note:', err)
    const fallbackNote: Note = {
      id: Date.now(),
      title,
      content,
      category: 'Personal',
      tags: [],
      isFavorite: false,
      createdAt: new Date(),
      updatedAt: new Date()
    }
    notes.value.unshift(fallbackNote)
    return fallbackNote
  } finally {
    isLoading.value = false
  }
}

const updateNote = async (note: Note) => {
  try {
    error.value = null
    note.updatedAt = new Date()
    await api.put<ApiNote>(`/notes/${note.id}`, convertToApiNote(note))
  } catch (err) {
    error.value = 'Failed to update note'
    console.error('Error updating note:', err)
  }
}

const deleteNote = async (noteId: number) => {
  try {
    isLoading.value = true
    error.value = null
    await api.delete(`/notes/${noteId}`)
    notes.value = notes.value.filter(note => note.id !== noteId)
    if (selectedNote.value?.id === noteId) {
      selectedNote.value = null
    }
  } catch (err) {
    error.value = 'Failed to delete note'
    console.error('Error deleting note:', err)
    await fetchNotes()
  } finally {
    isLoading.value = false
  }
}

const toggleFavorite = async (note: Note) => {
  note.isFavorite = !note.isFavorite
  await updateNote(note)
}

// Initialize
onMounted(() => {
  fetchNotes()
})

const formatDate = (date: Date) => {
  const now = new Date()
  const diff = now.getTime() - date.getTime()
  const days = Math.floor(diff / (1000 * 60 * 60 * 24))
  
  if (days === 0) {
    return 'Today'
  } else if (days === 1) {
    return 'Yesterday'
  } else if (days < 7) {
    return `${days} days ago`
  } else {
    return date.toLocaleDateString()
  }
}

const getWordCount = (content: string) => {
  return content.trim().split(/\s+/).filter(word => word.length > 0).length
}

// UI functions
const selectNote = (note: Note) => {
  selectedNote.value = note
  isCreating.value = false
}

const createNewNote = () => {
  isCreating.value = true
  selectedNote.value = null
  newNoteTitle.value = ''
  newNoteContent.value = ''
}

const saveNewNote = async () => {
  if (newNoteTitle.value.trim()) {
    const createdNote = await createNote(newNoteTitle.value, newNoteContent.value)
    selectedNote.value = createdNote
    isCreating.value = false
    newNoteTitle.value = ''
    newNoteContent.value = ''
  }
}

const cancelCreate = () => {
  isCreating.value = false
  newNoteTitle.value = ''
  newNoteContent.value = ''
}

const handleNoteUpdate = (note: Note) => {
  updateNote(note)
}

const handleDeleteNote = (noteId: number) => {
  deleteNote(noteId)
}

const dismissError = () => {
  error.value = null
}
</script>

<template>
  <div class="min-h-screen bg-gray-50 flex">
    <!-- Loading Overlay -->
    <div v-if="isLoading" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div class="bg-white rounded-lg p-6 flex items-center gap-3">
        <div class="animate-spin rounded-full h-6 w-6 border-b-2 border-purple-500"></div>
        <span class="text-gray-700">Loading...</span>
      </div>
    </div>

    <!-- Error Toast -->
    <div v-if="error" class="fixed top-4 right-4 z-40 bg-red-500 text-white px-6 py-3 rounded-lg shadow-lg flex items-center gap-3">
      <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
        <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7 4a1 1 0 11-2 0 1 1 0 012 0zm-1-9a1 1 0 00-1 1v4a1 1 0 102 0V6a1 1 0 00-1-1z" clip-rule="evenodd"/>
      </svg>
      <span>{{ error }}</span>
      <button @click="dismissError" class="ml-2 hover:bg-red-600 rounded p-1">
        <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
          <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd"/>
        </svg>
      </button>
    </div>

    <!-- Sidebar -->
    <div class="w-80 bg-white shadow-lg flex flex-col">
      <!-- Header -->
      <div class="bg-gradient-to-br from-blue-500 via-purple-500 to-purple-600 text-white p-4">
        <div class="flex items-center justify-between mb-4">
          <div class="flex items-center gap-2">
            <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 20 20">
              <path d="M4 4a2 2 0 00-2 2v8a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2H4zm0 2h12v8H4V6z"/>
              <path d="M6 8h8v2H6V8zm0 4h8v2H6v-2z"/>
            </svg>
            <h1 class="text-xl font-semibold">Notes</h1>
          </div>
          <button 
            @click="createNewNote"
            class="w-8 h-8 bg-white/20 hover:bg-white/30 rounded-full flex items-center justify-center transition-colors"
          >
            <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
              <path fill-rule="evenodd" d="M10 3a1 1 0 011 1v5h5a1 1 0 110 2h-5v5a1 1 0 11-2 0v-5H4a1 1 0 110-2h5V4a1 1 0 011-1z" clip-rule="evenodd"/>
            </svg>
          </button>
        </div>
        
        <!-- Stats -->
        <div class="grid grid-cols-2 gap-4">
          <div class="bg-white/10 rounded-lg p-3 text-center">
            <div class="text-2xl font-bold">{{ totalNotes }}</div>
            <div class="text-sm opacity-90">Total Notes</div>
          </div>
          <div class="bg-white/10 rounded-lg p-3 text-center">
            <div class="text-2xl font-bold">{{ favoriteNotes }}</div>
            <div class="text-sm opacity-90">Favorites</div>
          </div>
        </div>
      </div>

      <!-- Search -->
      <div class="p-4 border-b border-gray-200">
        <div class="relative">
          <svg class="absolute left-3 top-1/2 transform -translate-y-1/2 w-4 h-4 text-gray-400" fill="currentColor" viewBox="0 0 20 20">
            <path fill-rule="evenodd" d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z" clip-rule="evenodd"/>
          </svg>
          <input
            v-model="searchQuery"
            type="text"
            placeholder="Search notes..."
            class="w-full pl-10 pr-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-purple-500 focus:border-transparent"
          />
        </div>
      </div>

      <!-- Filters -->
      <div class="p-4 border-b border-gray-200 space-y-3">
        <div class="flex gap-2">
          <select v-model="selectedCategory" class="flex-1 px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-purple-500">
            <option v-for="category in categories" :key="category" :value="category">
              {{ category }}
            </option>
          </select>
          <select v-model="sortBy" class="flex-1 px-3 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-purple-500">
            <option v-for="option in sortOptions" :key="option" :value="option">
              {{ option }}
            </option>
          </select>
        </div>
        
        <div class="flex items-center justify-between">
          <button
            @click="showFavoritesOnly = !showFavoritesOnly"
            class="flex items-center gap-2 px-3 py-2 rounded-lg transition-colors"
            :class="showFavoritesOnly ? 'bg-yellow-100 text-yellow-800' : 'text-gray-600 hover:bg-gray-100'"
          >
            <svg class="w-4 h-4" :class="showFavoritesOnly ? 'text-yellow-500' : 'text-gray-400'" fill="currentColor" viewBox="0 0 20 20">
              <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z"/>
            </svg>
            <span class="text-sm font-medium">Favorites Only</span>
          </button>
          
          <div class="flex border border-gray-300 rounded-lg overflow-hidden">
            <button
              @click="viewMode = 'grid'"
              class="p-2 transition-colors"
              :class="viewMode === 'grid' ? 'bg-purple-500 text-white' : 'bg-white text-gray-600 hover:bg-gray-50'"
            >
              <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
                <path d="M5 3a2 2 0 00-2 2v2a2 2 0 002 2h2a2 2 0 002-2V5a2 2 0 00-2-2H5zM5 11a2 2 0 00-2 2v2a2 2 0 002 2h2a2 2 0 002-2v-2a2 2 0 00-2-2H5zM11 5a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2V5zM11 13a2 2 0 012-2h2a2 2 0 012 2v2a2 2 0 01-2 2h-2a2 2 0 01-2-2v-2z"/>
              </svg>
            </button>
            <button
              @click="viewMode = 'list'"
              class="p-2 transition-colors"
              :class="viewMode === 'list' ? 'bg-purple-500 text-white' : 'bg-white text-gray-600 hover:bg-gray-50'"
            >
              <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
                <path fill-rule="evenodd" d="M3 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1z" clip-rule="evenodd"/>
              </svg>
            </button>
          </div>
        </div>
      </div>

      <!-- Notes List -->
      <div class="flex-1 overflow-y-auto p-4">
        <div v-if="filteredNotes.length === 0" class="text-center py-8">
          <svg class="w-12 h-12 text-gray-300 mx-auto mb-3" fill="currentColor" viewBox="0 0 20 20">
            <path fill-rule="evenodd" d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z" clip-rule="evenodd"/>
          </svg>
          <p class="text-gray-500">No notes found</p>
        </div>
        
        <!-- Grid View -->
        <div v-else-if="viewMode === 'grid'" class="grid gap-3">
          <div 
            v-for="note in filteredNotes" 
            :key="note.id"
            @click="selectNote(note)"
            class="bg-white border border-gray-200 rounded-lg p-4 cursor-pointer hover:shadow-md transition-all duration-200 relative group"
            :class="{ 'ring-2 ring-purple-500 shadow-md': selectedNote?.id === note.id }"
          >
            <div class="flex items-start justify-between mb-2">
              <h3 class="font-semibold text-gray-900 text-sm line-clamp-1">{{ note.title }}</h3>
              <button
                @click.stop="toggleFavorite(note)"
                class="opacity-0 group-hover:opacity-100 transition-opacity p-1"
                :class="note.isFavorite ? 'opacity-100' : ''"
              >
                <svg class="w-4 h-4" :class="note.isFavorite ? 'text-yellow-500' : 'text-gray-400'" fill="currentColor" viewBox="0 0 20 20">
                  <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z"/>
                </svg>
              </button>
            </div>
            
            <div class="flex items-center gap-2 text-xs text-gray-500 mb-2">
              <span class="bg-gray-100 px-2 py-1 rounded">{{ note.category }}</span>
              <span>{{ getWordCount(note.content) }} words</span>
            </div>
            
            <div class="flex items-center justify-between text-xs text-gray-400">
              <span>{{ formatDate(note.createdAt) }}</span>
              <div v-if="note.tags.length > 0" class="flex gap-1">
                <span v-for="tag in note.tags.slice(0, 2)" :key="tag" class="text-purple-600">#{{ tag }}</span>
                <span v-if="note.tags.length > 2" class="text-gray-500">+{{ note.tags.length - 2 }}</span>
              </div>
            </div>
          </div>
        </div>
        
        <!-- List View -->
        <div v-else class="space-y-2">
          <div 
            v-for="note in filteredNotes" 
            :key="note.id"
            @click="selectNote(note)"
            class="bg-white border border-gray-200 rounded-lg p-3 cursor-pointer hover:shadow-md transition-all duration-200 group"
            :class="{ 'ring-2 ring-purple-500 shadow-md': selectedNote?.id === note.id }"
          >
            <div class="flex items-center justify-between">
              <div class="flex-1 min-w-0">
                <h3 class="font-semibold text-gray-900 text-sm truncate">{{ note.title }}</h3>
                <div class="flex items-center gap-2 text-xs text-gray-500 mt-1">
                  <span class="bg-gray-100 px-2 py-1 rounded">{{ note.category }}</span>
                  <span>{{ formatDate(note.createdAt) }}</span>
                  <span>{{ getWordCount(note.content) }} words</span>
                </div>
              </div>
              <button
                @click.stop="toggleFavorite(note)"
                class="opacity-0 group-hover:opacity-100 transition-opacity p-1 ml-2"
                :class="note.isFavorite ? 'opacity-100' : ''"
              >
                <svg class="w-4 h-4" :class="note.isFavorite ? 'text-yellow-500' : 'text-gray-400'" fill="currentColor" viewBox="0 0 20 20">
                  <path d="M9.049 2.927c.3-.921 1.603-.921 1.902 0l1.07 3.292a1 1 0 00.95.69h3.462c.969 0 1.371 1.24.588 1.81l-2.8 2.034a1 1 0 00-.364 1.118l1.07 3.292c.3.921-.755 1.688-1.54 1.118l-2.8-2.034a1 1 0 00-1.175 0l-2.8 2.034c-.784.57-1.838-.197-1.539-1.118l1.07-3.292a1 1 0 00-.364-1.118L2.98 8.72c-.783-.57-.38-1.81.588-1.81h3.461a1 1 0 00.951-.69l1.07-3.292z"/>
                </svg>
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Main Content -->
    <div class="flex-1 flex flex-col">
      <!-- Create New Note Form -->
      <div v-if="isCreating" class="h-full flex flex-col">
        <div class="bg-white border-b border-gray-200 p-4 flex items-center justify-between">
          <h2 class="text-xl font-semibold text-gray-900">Create New Note</h2>
          <div class="flex gap-2">
            <button 
              @click="cancelCreate"
              class="px-4 py-2 text-gray-600 hover:text-gray-800 transition-colors"
            >
              Cancel
            </button>
            <button 
              @click="saveNewNote"
              class="px-4 py-2 bg-purple-500 text-white rounded-lg hover:bg-purple-600 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
              :disabled="!newNoteTitle.trim() || isLoading"
            >
              {{ isLoading ? 'Saving...' : 'Save Note' }}
            </button>
          </div>
        </div>
        <div class="flex-1 p-6 bg-white">
          <input
            v-model="newNoteTitle"
            type="text"
            placeholder="Note title..."
            class="w-full text-2xl font-bold border-none outline-none mb-4 placeholder-gray-400"
          />
          <textarea
            v-model="newNoteContent"
            placeholder="Start writing your note..."
            class="w-full h-full resize-none border-none outline-none text-gray-700 placeholder-gray-400"
          ></textarea>
        </div>
      </div>

      <!-- Selected Note View -->
      <div v-else-if="selectedNote" class="h-full flex flex-col">
        <div class="bg-white border-b border-gray-200 p-4 flex items-center justify-between">
          <div>
            <h2 class="text-xl font-semibold text-gray-900">{{ selectedNote.title }}</h2>
            <p class="text-sm text-gray-500">
              Created {{ formatDate(selectedNote.createdAt) }}
              <span v-if="selectedNote.updatedAt > selectedNote.createdAt">
                • Updated {{ formatDate(selectedNote.updatedAt) }}
              </span>
            </p>
          </div>
          <button 
            @click="handleDeleteNote(selectedNote.id)"
            class="p-2 text-red-500 hover:text-red-700 hover:bg-red-50 rounded-lg transition-colors disabled:opacity-50"
            :disabled="isLoading"
          >
            <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
              <path fill-rule="evenodd" d="M9 2a1 1 0 000 2h2a1 1 0 100-2H9z" clip-rule="evenodd"/>
              <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd"/>
            </svg>
          </button>
        </div>
        <div class="flex-1 p-6 bg-white overflow-y-auto">
          <textarea
            v-model="selectedNote.content"
            @input="handleNoteUpdate(selectedNote)"
            class="w-full h-full resize-none border-none outline-none text-gray-700 text-lg leading-relaxed"
            placeholder="Start writing..."
            :disabled="isLoading"
          ></textarea>
        </div>
      </div>

      <!-- Empty State -->
      <div v-else class="flex-1 flex flex-col items-center justify-center bg-white">
        <div class="text-center max-w-md">
          <div class="w-24 h-24 bg-gradient-to-br from-purple-100 to-blue-100 rounded-full flex items-center justify-center mb-6 mx-auto">
            <svg class="w-12 h-12 text-purple-500" fill="currentColor" viewBox="0 0 20 20">
              <path d="M4 4a2 2 0 00-2 2v8a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2H4zm0 2h12v8H4V6z"/>
              <path d="M6 8h8v2H6V8zm0 4h8v2H6v-2z"/>
            </svg>
          </div>
          <h3 class="text-2xl font-bold text-gray-900 mb-3">Welcome to Your Notes</h3>
          <p class="text-gray-600 mb-8 leading-relaxed">
            Select a note from the sidebar to start reading, or create your first note to get started.
          </p>
          <div class="space-y-3">
            <button 
              @click="createNewNote"
              class="w-full inline-flex items-center justify-center gap-3 px-6 py-3 bg-gradient-to-r from-purple-500 to-blue-500 text-white rounded-lg hover:from-purple-600 hover:to-blue-600 transition-all duration-200 font-medium shadow-lg hover:shadow-xl transform hover:-translate-y-0.5"
            >
              <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
                <path fill-rule="evenodd" d="M10 3a1 1 0 011 1v5h5a1 1 0 110 2h-5v5a1 1 0 11-2 0v-5H4a1 1 0 110-2h5V4a1 1 0 011-1z" clip-rule="evenodd"/>
              </svg>
              Create Your First Note
            </button>
            <button 
              @click="selectedNote = filteredNotes[0] || null"
              v-if="filteredNotes.length > 0"
              class="w-full inline-flex items-center justify-center gap-3 px-6 py-3 bg-white text-gray-700 border-2 border-gray-200 rounded-lg hover:border-purple-300 hover:text-purple-600 transition-all duration-200 font-medium"
            >
              <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
                <path fill-rule="evenodd" d="M3 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zm0 4a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1z" clip-rule="evenodd"/>
              </svg>
              Browse Existing Notes
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Custom scrollbar */
::-webkit-scrollbar {
  width: 6px;
}

::-webkit-scrollbar-track {
  background: #f1f1f1;
}

::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}

/* Line clamp utility */
.line-clamp-1 {
  display: -webkit-box;
  --webkit-line-clamp: 1;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* Enhanced transitions */
.transition-all {
  transition-property: all;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  transition-duration: 150ms;
}

/* Gradient text */
.bg-gradient-to-r {
  background-image: linear-gradient(to right, var(--tw-gradient-stops));
}

.from-purple-500 {
  --tw-gradient-from: #8b5cf6;
  --tw-gradient-stops: var(--tw-gradient-from), var(--tw-gradient-to, rgba(139, 92, 246, 0));
}

.to-blue-500 {
  --tw-gradient-to: #3b82f6;
}

/* Card hover effects */
.group:hover .group-hover\:opacity-100 {
  opacity: 1;
}

/* Animation for loading */
@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

.animate-spin {
  animation: spin 1s linear infinite;
}
</style>