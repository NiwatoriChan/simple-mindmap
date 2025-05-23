<template>
  <Modal :is-open="props.isOpen" :title="props.title" @close="emit('close')">
    <input type="url" v-model="editableHyperlink" placeholder="https://example.com" class="w-full p-2 border border-slate-600 rounded mb-4 bg-slate-700 text-gray-900 dark:text-gray-200 focus:ring-2 focus:ring-blue-500 outline-none" ref="hyperlinkInputArea">
    <div class="flex justify-end space-x-2">
      <button @click="emit('close')" class="px-4 py-2 bg-slate-600 text-gray-200 hover:bg-slate-500 rounded transition-colors">Annuler</button>
      <button @click="save" class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 transition-colors">Sauvegarder</button>
    </div>
  </Modal>
</template>

<script setup>
import { ref, watch, defineProps, defineEmits, nextTick } from 'vue';
import Modal from './Modal.vue'; // Assuming Modal.vue is in the same directory

const props = defineProps({
  isOpen: Boolean,
  initialHyperlink: String,
  title: {
    type: String,
    default: "Ajouter/Modifier un lien hypertexte"
  }
});

const emit = defineEmits(['close', 'save-hyperlink']);

const editableHyperlink = ref('');
const hyperlinkInputArea = ref(null);

watch(() => props.initialHyperlink, (newVal) => {
  editableHyperlink.value = newVal || ''; // Ensure it's not undefined
});

watch(() => props.isOpen, (newVal) => {
  if (newVal) {
    editableHyperlink.value = props.initialHyperlink || ''; // Reset on open
    nextTick(() => {
      hyperlinkInputArea.value?.focus();
    });
  }
});

const save = () => {
  emit('save-hyperlink', editableHyperlink.value);
  // The parent (App.vue) will handle closing the modal via the @save-hyperlink handler
  // by setting isHyperlinkModalOpen = false. So, no need to emit('close') here.
};
</script>

<style scoped>
/* No specific styles needed if Tailwind is handling everything and Modal.vue has its own styles */
</style>
