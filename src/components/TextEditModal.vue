<template>
  <Modal :is-open="props.isOpen" :title="title" @close="emit('close')">
    <textarea v-model="editableText" class="w-full p-2 border border-slate-600 rounded mb-4 bg-slate-700 text-gray-200 focus:ring-2 focus:ring-blue-500 outline-none" rows="3" ref="nodeTextInputAreaRef"></textarea>
    <div class="flex justify-end space-x-2">
      <button @click="emit('close')" class="px-4 py-2 bg-slate-600 text-gray-200 hover:bg-slate-500 rounded transition-colors">Annuler</button>
      <button @click="save" class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 transition-colors">Sauvegarder</button>
    </div>
  </Modal>
</template>

<script setup>
import { ref, watch, defineProps, defineEmits, nextTick } from 'vue';
import Modal from './Modal.vue'; // Assuming Modal.vue is in the same directory for now. Adjust if needed.

const props = defineProps({
  isOpen: Boolean,
  initialText: String,
  title: { // Added title prop
    type: String,
    default: "Modifier le texte du nœud"
  }
});

const emit = defineEmits(['close', 'save-text']);

const editableText = ref('');
const nodeTextInputAreaRef = ref(null);

watch(() => props.initialText, (newVal) => {
  editableText.value = newVal;
});

watch(() => props.isOpen, (newVal) => {
  if (newVal) {
    // Update editableText when modal opens with the current initialText
    editableText.value = props.initialText; 
    nextTick(() => {
      nodeTextInputAreaRef.value?.focus();
    });
  }
});

const save = () => {
  emit('save-text', editableText.value);
  // emit('close'); // The parent (App.vue) will handle closing after save logic.
};
</script>

<style scoped>
/* No specific styles needed if Tailwind is handling everything */
</style>
