<template>
  <div id="toolbar" class="p-2 m-2 flex items-center space-x-1 sm:space-x-2 flex-wrap bg-slate-800 border border-slate-700 rounded-lg shadow">
    <button @click="handleToggleFileMenu" ref="fileMenuButtonRef" title="Fichier" class="tool-button bg-sky-600 text-white hover:bg-sky-700 rounded mr-1">
      <i class="fas fa-bars"></i>
    </button>
    <span class="border-r h-6 border-slate-600"></span>

    <button @click="emit('undo')" title="Annuler (Ctrl+Z)" :disabled="!props.canUndo" class="tool-button bg-slate-600 text-gray-200 hover:bg-slate-500 rounded disabled:opacity-50 disabled:cursor-not-allowed">
      <i class="fas fa-undo"></i>
    </button>
    <button @click="emit('redo')" title="Rétablir (Ctrl+Y)" :disabled="!props.canRedo" class="tool-button bg-slate-600 text-gray-200 hover:bg-slate-500 rounded disabled:opacity-50 disabled:cursor-not-allowed">
      <i class="fas fa-redo"></i>
    </button>
    <span class="border-r h-6 mx-1 border-slate-600"></span>

    <span class="text-xs text-gray-400 ml-2 mr-1 hidden sm:inline">Alt+Glisser pour lier/créer</span>
    <div class="flex-grow"></div>

    <!-- Removed "Ouvrir .mapvue" button, as this is now in FileMenu.vue -->
    <button @click="triggerImageInput" title="Importer image pour noeud" class="tool-button bg-purple-500 text-white rounded hover:bg-purple-600 mr-1"><i class="fas fa-image"></i></button>

    <button @click="emit('export-png')" title="Exporter en PNG" class="tool-button bg-indigo-500 text-white rounded hover:bg-indigo-600">
      <i class="fas fa-file-export"></i>
    </button>
    <button @click="emit('open-about-modal')" title="À propos" class="tool-button bg-slate-600 text-gray-200 hover:bg-slate-500 rounded">
      <i class="fas fa-question-circle"></i>
    </button>

    <input type="file" ref="imageInputRef" @change="onImageSelected" accept="image/*" class="hidden">
    <!-- Removed mapInputRef and its input element -->
  </div>
</template>

<script setup>
import { ref, defineProps, defineEmits } from 'vue';

const props = defineProps({
  canUndo: Boolean,
  canRedo: Boolean
});

const emit = defineEmits([
  'toggle-file-menu',
  'undo',
  'redo',
  'export-png',
  'open-about-modal',
  'image-selected' 
  // Removed 'map-file-selected' from emits as it's handled by FileMenu.vue
]);

const imageInputRef = ref(null);
// Removed mapInputRef
const fileMenuButtonRef = ref(null); // Ref for the file menu button itself

const triggerImageInput = () => {
  imageInputRef.value?.click();
};

// Removed triggerMapInput

const onImageSelected = (event) => {
  const file = event.target.files[0];
  if (file) {
    emit('image-selected', file);
  }
  event.target.value = ''; // Reset file input
};

// Removed onMapFileSelected

const handleToggleFileMenu = (event) => {
    // Pass the button element itself for positioning calculations if needed by parent
    emit('toggle-file-menu', event.currentTarget);
}

</script>

<style scoped>
/* Styles from App.vue specific to toolbar */
/* #toolbar { border-radius: 0.5rem; box-shadow: 0 2px 5px rgba(0,0,0,0.1); } - This ID is on the root, so it's fine */
.tool-button, .toolbar-select { /* .toolbar-select is not used in the current template, but kept for consistency if added later */
    transition: background-color 0.2s ease-in-out, transform 0.1s ease, color 0.2s ease-in-out;
    display: inline-flex; align-items: center; justify-content: center;
    height: 2.5rem; padding-left: 0.75rem; padding-right: 0.75rem;
}
.tool-button { width: 2.5rem; }
.tool-button:active, .toolbar-select:active { transform: scale(0.95); }

/* Specific styles for toolbar items if they were not covered by Tailwind utility classes */
/* For example, if .toolbar-select was used and had specific styles:
.toolbar-select {
    background-color: #37474f;
    border-color: #546e7a;
    color: #eceff1;
    min-width: 80px;
}
*/
</style>
