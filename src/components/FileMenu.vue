<template>
  <div v-if="props.visible" id="fileContextMenu" ref="fileMenuElementRef"
       :style="{ position: 'absolute', top: props.top, left: props.left, zIndex: 1000 }">
    <div @click="emitResetApplication" class="context-menu-item">
      <i class="fas fa-file"></i> Nouveau
    </div>
    <div class="context-menu-separator"></div>
    <div @click="triggerMapInput" class="context-menu-item">
      <i class="fas fa-folder-open"></i> Ouvrir un .mapvue
    </div>
    <div @click="emitSaveMapToFile" class="context-menu-item">
      <i class="fas fa-file-download"></i> Télécharger (.mapvue)
    </div>
    <div class="context-menu-separator"></div>
    <div class="context-menu-item context-submenu">
      <i class="fas fa-cog"></i> Propriétés <i :class="fileMenuSubmenuOpenLeft ? 'fas fa-caret-left float-right' : 'fas fa-caret-right float-right'"></i>
      <div class="submenu-content" :class="{'submenu-left': fileMenuSubmenuOpenLeft}">
        <div class="submenu-item flex items-center justify-between px-3 py-2 hover:bg-slate-700" @click.stop>
          <label for="canvasBgColorPickerMenuFile" class="text-sm text-gray-300 mr-3 whitespace-nowrap cursor-pointer">Fond Canevas:</label>
          <input type="color" id="canvasBgColorPickerMenuFile"
                 class="color-picker-input h-6 w-10 p-0.5 rounded border-slate-600 bg-slate-700"
                 style="margin-left: 0;"
                 :value="props.initialCanvasBgColor"
                 @input="onCanvasBgColorChange">
        </div>
      </div>
    </div>
    <input type="file" ref="mapInputRef" @change="onMapFileSelected" accept=".mapvue,application/json" class="hidden">
  </div>
</template>

<script setup>
import { ref, computed, defineProps, defineEmits, nextTick, watch } from 'vue';

const props = defineProps({
  visible: Boolean,
  top: String,
  left: String,
  initialCanvasBgColor: String, // Renamed to avoid direct v-model conflict, will emit update
});

const emit = defineEmits([
  'reset-application',
  'save-map-to-file',
  'update-canvas-bg-color',
  'map-file-selected',
  'close-file-menu' // To tell App.vue to hide the menu
]);

const fileMenuElementRef = ref(null);
const mapInputRef = ref(null);

const fileMenuSubmenuOpenLeft = computed(() => {
  if (fileMenuElementRef.value) {
    const menuRect = fileMenuElementRef.value.getBoundingClientRect();
    const submenuWidthEstimate = 200; // As used in App.vue
    return menuRect.left + menuRect.width + submenuWidthEstimate > window.innerWidth;
  }
  return false;
});

const triggerMapInput = () => {
  mapInputRef.value?.click();
  // App.vue will handle closing the menu after the action
};

const onMapFileSelected = (event) => {
  const file = event.target.files[0];
  if (file) {
    emit('map-file-selected', file);
  }
  event.target.value = ''; // Reset file input
  emit('close-file-menu'); // Close menu after selection or cancellation
};

const onCanvasBgColorChange = (event) => {
  emit('update-canvas-bg-color', event.target.value);
  // Menu can remain open while changing color
};

const emitResetApplication = () => {
    emit('reset-application');
    emit('close-file-menu');
}

const emitSaveMapToFile = () => {
    emit('save-map-to-file');
    emit('close-file-menu');
}

// Watch for visibility to adjust position if needed (though parent controls position)
watch(() => props.visible, (isVisible) => {
  if (isVisible) {
    nextTick(() => {
      // Parent (App.vue) is responsible for initial positioning.
      // This component could re-adjust if it knew its own dimensions and props.x/y
      // but that logic is more complex and usually handled by the parent that controls visibility.
    });
  }
});

</script>

<style scoped>
/* Copied from App.vue, #contextMenu renamed to #fileContextMenu where appropriate */
#fileContextMenu {
    position: absolute;
    z-index: 1000;
    border: 1px solid #4a5568; /* Tailwind: border-slate-600 */
    border-radius: 0.375rem; /* Tailwind: rounded-md */
    box-shadow: 0 4px 6px rgba(0,0,0,0.3); /* Tailwind: shadow-lg (approx) */
    min-width: 220px;
    padding-top: 0.25rem; /* Tailwind: pt-1 */
    padding-bottom: 0.25rem; /* Tailwind: pb-1 */
    background-color: #1e293b; /* Tailwind: bg-slate-800 */
}
.context-menu-item { /* Shared class name, can be used by FileMenu as well */
    display: block;
    padding: 0.5rem 1rem; /* Tailwind: px-4 py-2 */
    font-size: 0.875rem; /* Tailwind: text-sm */
    cursor: pointer;
    white-space: nowrap;
    line-height: 1.5; /* Tailwind: leading-normal */
    color: #e2e8f0; /* Tailwind: text-slate-200 */
    transition: background-color 0.1s ease;
}
.context-menu-item:hover {
    background-color: #334155; /* Tailwind: hover:bg-slate-700 */
}
.context-menu-item > i:first-child:not(.fa-caret-right):not(.fa-caret-left) {
    margin-right: 0.6em;
    width: 1.25em;
    text-align: center;
    display: inline-block;
    vertical-align: middle;
}
.context-menu-separator {
    height: 1px;
    margin-top: 0.25rem; /* Tailwind: my-1 */
    margin-bottom: 0.25rem;
    background-color: #4a5568; /* Tailwind: bg-slate-600 */
}
.context-submenu {
    position: relative;
}
.submenu-content {
    display: none;
    position: absolute;
    left: 100%;
    top: -5px; /* Adjust as needed */
    border: 1px solid #4a5568;
    border-radius: 0.375rem;
    box-shadow: 0 4px 6px rgba(0,0,0,0.3);
    min-width: 180px;
    padding-top: 0.25rem;
    padding-bottom: 0.25rem;
    z-index: 1001; /* Ensure submenu is above parent */
    background-color: #1e293b;
}
.submenu-content.submenu-left {
    left: auto;
    right: 100%;
}
.context-submenu:hover > .submenu-content {
    display: block;
}
.submenu-item { /* Shared class name */
    display: block;
    padding: 0.5rem 1rem;
    font-size: 0.875rem;
    cursor: pointer;
    white-space: nowrap;
    line-height: 1.5;
    color: #e2e8f0;
    transition: background-color 0.1s ease;
}
.submenu-item:hover {
    background-color: #334155;
}
/* Copied from App.vue for color picker input in FileMenu's properties submenu */
.color-picker-input {
    width: 2.5rem; height: 1.5rem; padding: 0.1rem;
    border: 1px solid #4a5568;
    border-radius: 0.25rem; cursor: pointer; vertical-align: middle; margin-left: 0.25rem;
    background-color: #2d3748;
}
.hidden {
  display: none;
}
</style>
