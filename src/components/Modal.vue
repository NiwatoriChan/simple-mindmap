<template>
  <div v-if="isOpen" class="fixed inset-0 modal-backdrop flex items-center justify-center z-50 p-4" @click="$emit('close')">
    <div class="modal-content bg-slate-800 p-5 sm:p-6 rounded-lg shadow-xl w-full" :class="widthClass" @click.stop>
      <div class="flex justify-between items-center mb-4">
        <h3 class="text-lg font-semibold text-white">{{ title }}</h3>
        <button @click="$emit('close')" class="text-gray-500 hover:text-gray-300 transition-colors">
          <i class="fas fa-times"></i>
        </button>
      </div>
      <slot></slot>
    </div>
  </div>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue';

defineProps({
  isOpen: Boolean,
  title: String,
  widthClass: {
    type: String,
    default: 'max-w-sm'
  }
});

defineEmits(['close']);
</script>

<style scoped>
/* Styles from App.vue that were specific to ModalComponent can be moved here if necessary */
/* For now, keeping global styles in App.vue as they might be used by other components or elements */
/* .modal-backdrop { background-color: rgba(0, 0, 0, 0.6); } from App.vue global style */
/* .modal-content { transform: scale(0.95); opacity: 0; animation: modal-appear 0.3s forwards; } from App.vue global style */
/* @keyframes modal-appear { to { opacity: 1; transform: scale(1); } } from App.vue global style */
</style>
