<template>
  <Modal :is-open="props.isOpen" :title="props.title" @close="emit('close')">
    <p class="mb-4" :class="typeClass">{{ props.message }}</p>
    <div class="flex justify-end">
      <button @click="emit('close')" class="px-4 py-2 bg-slate-600 text-gray-200 hover:bg-slate-500 rounded transition-colors">Fermer</button>
    </div>
  </Modal>
</template>

<script setup>
import { computed, defineProps, defineEmits } from 'vue';
import Modal from './Modal.vue'; // Assuming Modal.vue is in the same components directory

const props = defineProps({
  isOpen: Boolean,
  title: String,
  message: String,
  type: { // For 'error', 'success', 'info'
    type: String,
    default: 'error' // Default to error type
  }
});

const emit = defineEmits(['close']);

const typeClass = computed(() => {
  if (props.type === 'success') return 'text-green-400';
  if (props.type === 'info') return 'text-blue-400';
  return 'text-red-400'; // Default to error
});
</script>

<style scoped>
/* No specific styles needed if Tailwind is handling everything and Modal.vue has its own styles */
</style>
