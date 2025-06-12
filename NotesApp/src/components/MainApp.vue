<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import axios from 'axios'

interface Note {
  id: number
  title: string
  content: string
  createdAt: Date
  updatedAt: Date
}

interface ApiNote {
  id: number
  title: string
  content: string
  createdAt: string
  updatedAt: string
}

// API configuration
const API_BASE_URL = 'http://localhost:3000/api' // Change to your backend URL
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

const notesCount = computed(() => notes.value.length)

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
        title: 'Welcome to Your Notes!',
        content: 'This is your first note. You can edit it or create new ones.',
        createdAt: new Date(Date.now() - 24 * 60 * 60 * 1000),
        updatedAt: new Date(Date.now() - 24 * 60 * 60 * 1000)
      },
      {
        id: 2,
        title: 'Ideas & Inspiration',
        content: 'Keep track of your brilliant ideas and inspirations here.',
        createdAt: new Date(Date.now() - 2 * 24 * 60 * 60 * 1000),
        updatedAt: new Date(Date.now() - 60 * 60 * 1000)
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
    // Fallback: create note locally
    const fallbackNote: Note = {
      id: Date.now(),
      title,
      content,
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
    // Note is already updated locally, so we don't need to do anything else
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
    // Revert the deletion if API call failed
    await fetchNotes()
  } finally {
    isLoading.value = false
  }
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
  } else {
    return `${days} days ago`
  }
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
      <div class="bg-gradient-to-r from-purple-500 to-purple-600 text-white p-4 flex items-center justify-between">
        <div class="flex items-center gap-2">
          <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 20 20">
            <path d="M4 4a2 2 0 00-2 2v8a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2H4zm0 2h12v8H4V6z"/>
            <path d="M6 8h8v2H6V8zm0 4h8v2H6v-2z"/>
          </svg>
          <h1 class="text-xl font-semibold">Notes ({{ notesCount }})</h1>
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

      <!-- Notes List -->
      <div class="flex-1 overflow-y-auto">
        <div v-for="note in notes" :key="note.id" class="border-b border-gray-200 last:border-b-0">
          <div 
            @click="selectNote(note)"
            class="p-4 cursor-pointer hover:bg-gray-50 transition-colors"
            :class="{ 'bg-purple-50 border-r-4 border-purple-500': selectedNote?.id === note.id }"
          >
            <h3 class="font-semibold text-gray-900 mb-1">{{ note.title }}</h3>
            <div class="flex items-center gap-2 text-sm text-gray-500">
              <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
                <path fill-rule="evenodd" d="M6 2a1 1 0 00-1 1v1H4a2 2 0 00-2 2v10a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2h-1V3a1 1 0 10-2 0v1H7V3a1 1 0 00-1-1zm0 5a1 1 0 000 2h8a1 1 0 100-2H6z" clip-rule="evenodd"/>
              </svg>
              <span>{{ formatDate(note.createdAt) }}</span>
              <span v-if="note.updatedAt > note.createdAt" class="text-blue-500 flex items-center gap-1">
                <svg class="w-3 h-3" fill="currentColor" viewBox="0 0 20 20">
                  <path fill-rule="evenodd" d="M4 2a1 1 0 011 1v2.101a7.002 7.002 0 0111.601 2.566 1 1 0 11-1.885.666A5.002 5.002 0 005.999 7H9a1 1 0 010 2H4a1 1 0 01-1-1V3a1 1 0 011-1zm.008 9.057a1 1 0 011.276.61A5.002 5.002 0 0014.001 13H11a1 1 0 110-2h5a1 1 0 011 1v5a1 1 0 11-2 0v-2.101a7.002 7.002 0 01-11.601-2.566 1 1 0 01.61-1.276z" clip-rule="evenodd"/>
                </svg>
                Updated
              </span>
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
              <path fill-rule="evenodd" d="M10 5a1 1 0 011 1v3l5-5a1 1 0 111.414 1.414L11.414 11l5 5a1 1 0 11-1.414 1.414l-5-5-5 5a1 1 0 01-1.414-1.414l5-5-5-5A1 1 0 015 5h5z" clip-rule="evenodd"/>
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
        <div class="text-center">
          <div class="w-24 h-24 bg-purple-100 rounded-full flex items-center justify-center mb-6 mx-auto">
            <svg class="w-12 h-12 text-purple-500" fill="currentColor" viewBox="0 0 20 20">
              <path d="M4 4a2 2 0 00-2 2v8a2 2 0 002 2h12a2 2 0 002-2V6a2 2 0 00-2-2H4zm0 2h12v8H4V6z"/>
              <path d="M6 8h8v2H6V8zm0 4h8v2H6v-2z"/>
            </svg>
          </div>
          <h3 class="text-xl font-semibold text-gray-900 mb-2">Select a note to view</h3>
          <p class="text-gray-600 mb-6">Choose a note from the list or create a new one</p>
          <button 
            @click="createNewNote"
            class="inline-flex items-center gap-2 px-6 py-3 bg-purple-500 text-white rounded-lg hover:bg-purple-600 transition-colors font-medium"
          >
            <svg class="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
              <path fill-rule="evenodd" d="M10 3a1 1 0 011 1v5h5a1 1 0 110 2h-5v5a1 1 0 11-2 0v-5H4a1 1 0 110-2h5V4a1 1 0 011-1z" clip-rule="evenodd"/>
            </svg>
            Create Your First Note
          </button>
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
</style>