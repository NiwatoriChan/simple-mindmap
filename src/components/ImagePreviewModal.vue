<template>
  <Modal :is-open="props.isOpen" :title="props.title" :width-class="props.widthClass" @close="emit('close')">
    <img :src="props.imageUrl" alt="Aperçu" class="max-w-full max-h-[80vh] block rounded mx-auto">
  </Modal>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue';
import Modal from './Modal.vue'; // Adjusted path assuming Modal.vue is in the same directory

const props = defineProps({
  isOpen: Boolean,
  imageUrl: String,
  title: {
    type: String,
    default: "Aperçu de l'image" // Default title
  },
  widthClass: {
    type: String,
    default: 'max-w-lg' // Default width class
  }
});

const emit = defineEmits(['close']);
</script>

<style scoped>
/* No specific styles needed if Tailwind is handling everything and Modal.vue has its own styles */
</style>
