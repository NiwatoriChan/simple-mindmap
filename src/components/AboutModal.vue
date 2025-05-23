<template>
  <Modal :is-open="props.isOpen" :title="props.title" :width-class="props.widthClass" @close="emit('close')">
    <p class="text-gray-300 mb-2"><strong>Version :</strong> 1.1.0-alpha</p>
    <p class="text-gray-300 mb-4"><strong>Auteur :</strong> Charles-Antoine Huberdeau</p>
    <div class="border-t border-slate-700 pt-4 mt-4">
      <h4 class="text-md font-semibold mb-2 text-gray-200">Technologies :</h4>
      <ul class="list-disc list-inside text-sm text-gray-400">
        <li>Vue.js 3</li><li>Tailwind CSS</li><li>Font Awesome</li>
      </ul>
    </div>
    <div class="flex justify-end mt-6">
      <button @click="emit('close')" class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 transition-colors">Fermer</button>
    </div>
  </Modal>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue';
import Modal from './Modal.vue'; // Assuming Modal.vue is in the same components directory

const props = defineProps({
  isOpen: Boolean,
  title: {
    type: String,
    default: "À propos de Carte Mentale Interactive" // Default title
  },
  widthClass: {
    type: String,
    default: 'max-w-md' // Default width class
  }
});

const emit = defineEmits(['close']);
</script>

<style scoped>
/* No specific styles needed if Tailwind is handling everything and Modal.vue has its own styles */
</style>
