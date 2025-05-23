<template>
  <div v-if="props.visible" id="contextMenu" ref="contextMenuElementRef"
       :style="{ top: props.y + 'px', left: props.x + 'px' }">
    <template v-if="props.nodeId && props.node">
      <div @click="emit('add-child-node', props.nodeId)" class="context-menu-item"><i class="fas fa-plus"></i> Ajouter enfant</div>
      <div @click="emit('open-text-edit-modal', props.nodeId)" class="context-menu-item"><i class="fas fa-pencil-alt"></i> Modifier texte</div>
      <div @click="emit('trigger-image-upload', props.nodeId)" class="context-menu-item"><i class="fas fa-image"></i> {{ props.node.imageUrl ? 'Modifier' : 'Ajouter' }} image</div>
      <div @click="emit('open-hyperlink-modal', props.nodeId)" class="context-menu-item"><i class="fas fa-link"></i> {{ props.node.hyperlink ? 'Modifier' : 'Ajouter' }} lien</div>
      <div class="context-menu-separator"></div>
      <div class="context-menu-item context-submenu" @mouseleave="emit('clear-preview')">
        <i class="fas fa-icons"></i> Icône ({{ getNodeIconPreview(props.node.icon) }}) <i :class="contextSubmenuOpenLeft ? 'fas fa-caret-left float-right' : 'fas fa-caret-right float-right'"></i>
        <div class="submenu-content" :class="{'submenu-left': contextSubmenuOpenLeft}">
          <div @click="emit('set-node-property', { nodeId: props.nodeId, property: 'icon', value: null })"
               @mouseenter="emit('set-preview', { nodeId: props.nodeId, property: 'icon', value: null })"
               class="submenu-item"><i class="fas fa-ban"></i> Aucune <i v-if="!props.node.icon" class="fas fa-check ml-2 text-blue-500"></i></div>
          <div v-for="icon in props.availableIcons" :key="icon.name"
               @click="emit('set-node-property', { nodeId: props.nodeId, property: 'icon', value: icon.class })"
               @mouseenter="emit('set-preview', { nodeId: props.nodeId, property: 'icon', value: icon.class })"
               class="submenu-item">
            <i :class="icon.class"></i> {{ icon.name }} <i v-if="props.node.icon === icon.class" class="fas fa-check ml-2 text-blue-500"></i>
          </div>
        </div>
      </div>
      <div class="context-menu-separator"></div>
      <template v-if="props.node.parentId !== null">
        <div @click="emit('detach-node', props.nodeId)" class="context-menu-item"><i class="fas fa-unlink"></i> Détacher du parent</div>
        <div class="context-menu-item context-submenu" @mouseleave="emit('clear-preview')">
          <i class="fas fa-sign-in-alt"></i> Connecter à ({{ props.node.entryPoint || 'auto' }}) <i :class="contextSubmenuOpenLeft ? 'fas fa-caret-left float-right' : 'fas fa-caret-right float-right'"></i>
          <div class="submenu-content" :class="{'submenu-left': contextSubmenuOpenLeft}">
            <div v-for="point in props.CONNECTION_POINTS" :key="point"
                 @mouseenter="emit('set-preview', { nodeId: props.nodeId, property: 'entryPoint', value: point })"
                 @click="emit('set-node-property', { nodeId: props.nodeId, property: 'entryPoint', value: point })"
                 class="submenu-item">
              {{ capitalize(point) }} <i v-if="props.node.entryPoint === point" class="fas fa-check ml-2 text-blue-500"></i>
            </div>
          </div>
        </div>
        <div class="context-menu-item context-submenu" @mouseleave="emit('clear-preview')">
          <i class="fas fa-sign-out-alt"></i> Sortie parent ({{ props.node.parentExitPoint || 'auto' }}) <i :class="contextSubmenuOpenLeft ? 'fas fa-caret-left float-right' : 'fas fa-caret-right float-right'"></i>
          <div class="submenu-content" :class="{'submenu-left': contextSubmenuOpenLeft}">
            <div v-for="point in props.CONNECTION_POINTS" :key="point"
                 @mouseenter="emit('set-preview', { nodeId: props.nodeId, property: 'parentExitPoint', value: point })"
                 @click="emit('set-node-property', { nodeId: props.nodeId, property: 'parentExitPoint', value: point })"
                 class="submenu-item">
              {{ capitalize(point) }} <i v-if="props.node.parentExitPoint === point" class="fas fa-check ml-2 text-blue-500"></i>
            </div>
          </div>
        </div>
        <div class="context-menu-item context-submenu" @mouseleave="emit('clear-preview')">
          <i class="fas fa-link"></i> Type de lien ({{ props.node.connectionType || 'solid' }}) <i :class="contextSubmenuOpenLeft ? 'fas fa-caret-left float-right' : 'fas fa-caret-right float-right'"></i>
          <div class="submenu-content" :class="{'submenu-left': contextSubmenuOpenLeft}">
            <div v-for="type in props.CONNECTION_TYPES" :key="type"
                 @mouseenter="emit('set-preview', { nodeId: props.nodeId, property: 'connectionType', value: type })"
                 @click="emit('set-node-property', { nodeId: props.nodeId, property: 'connectionType', value: type })"
                 class="submenu-item">
              <span class="link-indicator">{{ getConnectionTypeIndicator(type) }}</span> {{ capitalize(type) }} <i v-if="props.node.connectionType === type" class="fas fa-check ml-2 text-blue-500"></i>
            </div>
          </div>
        </div>
      </template>
      <div class="context-menu-item context-submenu" @mouseleave="emit('clear-preview')">
        <i class="fas fa-shapes"></i> Forme ({{ props.node.shape || 'rectangle' }}) <i :class="contextSubmenuOpenLeft ? 'fas fa-caret-left float-right' : 'fas fa-caret-right float-right'"></i>
        <div class="submenu-content" :class="{'submenu-left': contextSubmenuOpenLeft}">
          <div v-for="shape in props.NODE_SHAPES" :key="shape"
               @mouseenter="emit('set-preview', { nodeId: props.nodeId, property: 'shape', value: shape })"
               @click="emit('set-node-property', { nodeId: props.nodeId, property: 'shape', value: shape })"
               class="submenu-item">
            <i :class="getNodeShapeIcon(shape)"></i> {{ capitalize(shape) }} <i v-if="props.node.shape === shape" class="fas fa-check ml-2 text-blue-500"></i>
          </div>
        </div>
      </div>
      <div @click="emit('start-linking', props.nodeId)" class="context-menu-item"><i class="fas fa-people-arrows"></i> Connecter à un autre...</div>
      <div class="context-menu-separator"></div>
      <div class="context-menu-item context-submenu" @mouseleave="emit('clear-preview')">
        <i class="fas fa-border-style"></i> Style bordure ({{ capitalize(props.node.borderStyle || 'solid') }}) <i :class="contextSubmenuOpenLeft ? 'fas fa-caret-left float-right' : 'fas fa-caret-right float-right'"></i>
        <div class="submenu-content" :class="{'submenu-left': contextSubmenuOpenLeft}">
          <div v-for="style in props.AVAILABLE_BORDER_STYLES" :key="style.value"
               @mouseenter="emit('set-preview', { nodeId: props.nodeId, property: 'borderStyle', value: style.value })"
               @click="emit('set-node-property', { nodeId: props.nodeId, property: 'borderStyle', value: style.value })"
               class="submenu-item">
            {{ style.name }} <i v-if="(props.node.borderStyle || 'solid') === style.value" class="fas fa-check ml-2 text-blue-500"></i>
          </div>
        </div>
      </div>
      <div class="context-menu-item context-submenu" @mouseleave="emit('clear-preview')">
        <i class="fas fa-ruler-horizontal"></i> Épaisseur bordure ({{ props.node.borderWidth || 1 }}px) <i :class="contextSubmenuOpenLeft ? 'fas fa-caret-left float-right' : 'fas fa-caret-right float-right'"></i>
        <div class="submenu-content" :class="{'submenu-left': contextSubmenuOpenLeft}">
          <div v-for="width in [1, 2, 3, 4, 5]" :key="width"
               @mouseenter="emit('set-preview', { nodeId: props.nodeId, property: 'borderWidth', value: width })"
               @click="emit('set-node-property', { nodeId: props.nodeId, property: 'borderWidth', value: width })"
               class="submenu-item">
            {{ width }}px <i v-if="(props.node.borderWidth || 1) === width" class="fas fa-check ml-2 text-blue-500"></i>
          </div>
        </div>
      </div>
      <div class="context-menu-separator"></div>
      <div @click="emit('request-delete-node', props.nodeId)" class="context-menu-item"><i class="fas fa-trash-alt"></i> Supprimer nœud</div>
      <div class="context-menu-separator"></div>
      <div @click="triggerNodeBgColorPicker" class="context-menu-item">
        <i class="fas fa-palette"></i> Changer fond
        <input type="color" ref="nodeBgColorPickerFromContextRef" :value="props.selectedNodeBgColor" @input="handleNodeBgColorChange" class="hidden" :disabled="!props.nodeId">
      </div>
      <div @click="triggerNodeTextColorPicker" class="context-menu-item">
        <i class="fas fa-font"></i> Changer texte
        <input type="color" ref="nodeTextColorPickerFromContextRef" :value="props.selectedNodeTextColor" @input="handleNodeTextColorChange" class="hidden" :disabled="!props.nodeId">
      </div>
    </template>
    <template v-else>
      <div v-if="!props.hasRootNode" @click="emit('add-root-node'); emit('hide-context-menu');" class="context-menu-item"><i class="fas fa-circle-notch"></i> Créer concept central</div>
      <div @click="emit('add-node-at-position', { x: props.rawX, y: props.rawY }); emit('hide-context-menu');" class="context-menu-item"><i class="fas fa-plus-circle"></i> Ajouter nœud ici</div>
      <div class="context-menu-separator"></div>
      <div @click="emit('reset-view'); emit('hide-context-menu');" class="context-menu-item"><i class="fas fa-crosshairs"></i> Centrer vue</div>
      <div @click="emit('zoom', 1.2); emit('hide-context-menu');" class="context-menu-item"><i class="fas fa-search-plus"></i> Zoom avant</div>
      <div @click="emit('zoom', 0.8); emit('hide-context-menu');" class="context-menu-item"><i class="fas fa-search-minus"></i> Zoom arrière</div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed, watch, nextTick, defineProps, defineEmits } from 'vue';

const props = defineProps({
  visible: Boolean,
  x: Number,
  y: Number,
  nodeId: [String, null],
  rawX: Number,
  rawY: Number,
  node: [Object, null],
  hasRootNode: Boolean,
  availableIcons: Array,
  CONNECTION_POINTS: Array,
  CONNECTION_TYPES: Array,
  NODE_SHAPES: Array,
  AVAILABLE_BORDER_STYLES: Array,
  selectedNodeBgColor: String,
  selectedNodeTextColor: String,
});

const emit = defineEmits([
  'hide-context-menu', 'add-child-node', 'open-text-edit-modal', 'trigger-image-upload',
  'open-hyperlink-modal', 'set-node-property', 'detach-node', 'start-linking',
  'request-delete-node', 'add-root-node', 'add-node-at-position', 'reset-view', 'zoom',
  'set-node-background-color', 'set-node-text-color', 'set-preview', 'clear-preview'
]);

const contextMenuElementRef = ref(null);
const nodeBgColorPickerFromContextRef = ref(null);
const nodeTextColorPickerFromContextRef = ref(null);

const contextSubmenuOpenLeft = computed(() => {
  if (contextMenuElementRef.value) {
    const menuWidth = contextMenuElementRef.value.offsetWidth || 200;
    const submenuWidthEstimate = 180; // Estimate, or measure if critical
    return props.x + menuWidth + submenuWidthEstimate > window.innerWidth;
  }
  return false;
});

const adjustContextMenuPosition = () => {
  if (contextMenuElementRef.value) {
    const menuEl = contextMenuElementRef.value;
    const menuWidth = menuEl.offsetWidth;
    const menuHeight = menuEl.offsetHeight;
    let newX = props.x;
    let newY = props.y;

    if (newX + menuWidth > window.innerWidth) newX = window.innerWidth - menuWidth - 5;
    if (newY + menuHeight > window.innerHeight) newY = window.innerHeight - menuHeight - 5;
    if (newX < 0) newX = 5;
    if (newY < 0) newY = 5;

    // This component can't directly modify props.x and props.y.
    // If adjustment is needed that changes x/y, it should emit an event to parent.
    // For now, assuming initial x,y from parent is sufficient or minor adjustments are internal.
    // Or, the parent handles this logic *before* setting the x,y props.
    // For this exercise, we'll assume the parent has positioned it adequately.
    // If the component itself *must* adjust its position, it's more complex with props.
  }
};

watch(() => props.visible, (newValue) => {
  if (newValue) {
    nextTick(() => {
      adjustContextMenuPosition();
    });
  }
});

const capitalize = (value) => {
  if (!value) return '';
  value = value.toString();
  return value.charAt(0).toUpperCase() + value.slice(1);
};

const getNodeIconPreview = (iconClass) => {
  if (!iconClass) return 'Aucune';
  const iconObj = props.availableIcons.find(i => i.class === iconClass);
  return iconObj ? iconObj.name : 'Inconnue';
};

const getNodeShapeIcon = (shape) => {
  if (shape === 'ellipse') return 'far fa-circle';
  if (shape === 'diamond') return 'far fa-gem';
  if (shape === 'triangle') return 'fas fa-play fa-rotate-270';
  return 'far fa-square';
};

const getConnectionTypeIndicator = (type) => {
  if (type === 'solid') return '⎯';
  if (type === 'dashed') return '- -';
  if (type === 'arrow') return '->';
  return '?';
};

const triggerNodeBgColorPicker = () => {
  nodeBgColorPickerFromContextRef.value?.click();
  emit('hide-context-menu');
};

const triggerNodeTextColorPicker = () => {
  nodeTextColorPickerFromContextRef.value?.click();
  emit('hide-context-menu');
};

const handleNodeBgColorChange = (event) => {
  if (props.nodeId) {
    emit('set-node-background-color', { nodeId: props.nodeId, color: event.target.value });
  }
};

const handleNodeTextColorChange = (event) => {
  if (props.nodeId) {
    emit('set-node-text-color', { nodeId: props.nodeId, color: event.target.value });
  }
};

</script>

<style scoped>
#contextMenu { /* Removed #fileContextMenu as it's not part of this component */
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
.context-menu-item {
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
.submenu-item { /* Identical to context-menu-item, could be combined if desired */
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
.submenu-item > i {
    margin-right: 0.6em;
    width: 1.25em;
    text-align: center;
    display: inline-block;
    vertical-align: middle;
}
.submenu-item .link-indicator { /* For connection type indicators */
    display: inline-block;
    width: 20px;
    margin-right: 0.5rem;
    font-family: monospace;
    text-align: center;
    vertical-align: middle;
}
.hidden {
  display: none;
}
</style>
