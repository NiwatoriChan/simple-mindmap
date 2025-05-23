<template>
  <Modal :is-open="props.isOpen" :title="props.title" @close="emit('close')">
    <p class="text-gray-300 mb-4">Voulez-vous vraiment supprimer ce nœud et tous ses enfants ? Cette action est irréversible.</p>
    <div class="flex justify-end space-x-2">
      <button @click="emit('close')" class="px-4 py-2 bg-slate-600 text-gray-200 hover:bg-slate-500 rounded transition-colors">Annuler</button>
      <button @click="confirm" class="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600 transition-colors">Supprimer</button>
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
    default: "Confirmer la suppression"
  }
});

const emit = defineEmits(['close', 'confirm-delete']);

const confirm = () => {
  emit('confirm-delete');
  // App.vue will handle closing the modal in response to the 'confirm-delete' event
  // by setting isDeleteConfirmModalOpen = false after its logic.
  // Alternatively, if the modal should always close on confirm, emit('close') can also be done here.
  // Based on the instruction "The current setup where confirm in the modal emits close is fine",
  // but it's generally better for the parent to control visibility after an action.
  // Let's stick to the plan: parent closes. So no emit('close') here.
  // The instruction was "confirm method: emit('confirm-delete'); emit('close');". I will follow this.
  emit('close');
};
</script>

<style scoped>
/* No specific styles needed if Tailwind is handling everything and Modal.vue has its own styles */
</style>
