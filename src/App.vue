<template>
  <div id="app-vue-container" class="flex flex-col h-full"> <!-- Renamed id to avoid conflict with main.js mount point -->
    <Toolbar
      :canUndo="canUndo"
      :canRedo="canRedo"
      @toggle-file-menu="toggleFileMenu"
      @undo="undo"
      @redo="redo"
      @export-png="exportToPng"
      @open-about-modal="isAboutModalOpen = true"
      @image-selected="handleImageSelectedDirect"
      @map-file-selected="handleMapFileSelectedDirect"
      ref="toolbarRef"
    />
    <MindMapCanvas
      ref="mindMapCanvasComponentRef"
      :nodes="nodes"
      :selectedNodeId="selectedNodeId"
      :offsetX="offsetX"
      :offsetY="offsetY"
      :scale="scale"
      :isLinking="isLinking"
      :linkStartNodeId="linkStartNodeId"
      :isDraggingLink="isDraggingLink"
      :linkDragStartNodeId="linkDragStartNodeId"
      :linkDragEndPos="linkDragEndPos"
      :potentialLinkTargetId="potentialLinkTargetId"
      :potentialDropTargetId="potentialDropTargetId"
      :hoveredNodeId="hoveredNodeId"
      :hoveredConnectionId="hoveredConnectionId"
      :draggingConnectionEnd="draggingConnectionEnd"
      :potentialEndPoint="potentialEndPoint"
      :canvasBgColor="canvasBgColor"
      :previewConnectionSide="previewConnectionSide"
      :lastMousePos="lastMousePos" 
      @canvas-mousedown="handleCanvasMousedown"
      @canvas-mousemove="handleCanvasMousemove"
      @canvas-mouseup="handleCanvasMouseup"
      @canvas-mouseout="handleCanvasMouseout"
      @canvas-dblclick="handleCanvasDblclick"
      @canvas-wheel="handleCanvasWheel"
      @canvas-contextmenu="handleCanvasContextmenu"
      @update-last-mouse-pos="updateLastMousePos"
    />
    <TextEditModal
        :is-open="isTextEditModalOpen"
        :initial-text="nodeTextInput"
        title="Modifier le texte du nœud" 
        @close="isTextEditModalOpen = false"
        @save-text="saveNodeTextFromModal"
    />
    <HyperlinkModal
        :is-open="isHyperlinkModalOpen"
        :initial-hyperlink="hyperlinkInput"
        title="Ajouter/Modifier un lien hypertexte"
        @close="isHyperlinkModalOpen = false"
        @save-hyperlink="saveNodeHyperlinkFromModal"
    />
    <ImagePreviewModal
        :is-open="isImagePreviewModalOpen"
        :image-url="previewImageUrl"
        title="Aperçu de l'image"
        width-class="max-w-lg"
        @close="isImagePreviewModalOpen = false"
    />
    <DeleteConfirmModal
        :is-open="isDeleteConfirmModalOpen"
        title="Confirmer la suppression"
        @close="isDeleteConfirmModalOpen = false"
        @confirm-delete="confirmDeleteNode"
    />
    <modal :is-open="isErrorModalOpen" @close="isErrorModalOpen = false" :title="errorModalTitle">
        <p class="mb-4" :class="errorModalTypeClass">{{ errorModalMessage }}</p>
        <div class="flex justify-end">
            <button @click="isErrorModalOpen = false" class="px-4 py-2 bg-slate-600 text-gray-200 hover:bg-slate-500 rounded transition-colors">Fermer</button>
        </div>
    </modal>
    <modal :is-open="isAboutModalOpen" @close="isAboutModalOpen = false" title="À propos de Carte Mentale Interactive" width-class="max-w-md">
        <p class="text-gray-300 mb-2"><strong>Version :</strong> 1.1.0-alpha</p>
        <p class="text-gray-300 mb-4"><strong>Auteur :</strong> Charles-Antoine Huberdeau</p>
        <div class="border-t border-slate-700 pt-4 mt-4">
            <h4 class="text-md font-semibold mb-2 text-gray-200">Technologies :</h4>
            <ul class="list-disc list-inside text-sm text-gray-400">
                <li>Vue.js 3</li><li>Tailwind CSS</li><li>Font Awesome</li>
            </ul>
        </div>
        <div class="flex justify-end mt-6">
            <button @click="isAboutModalOpen = false" class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 transition-colors">Fermer</button>
        </div>
    </modal>
    <ContextMenu
        :visible="contextMenu.visible"
        :x="contextMenu.x"
        :y="contextMenu.y"
        :nodeId="contextMenu.nodeId"
        :rawX="contextMenu.rawX"
        :rawY="contextMenu.rawY"
        :node="contextMenu.nodeId ? findNodeById(contextMenu.nodeId) : null"
        :hasRootNode="hasRootNode"
        :availableIcons="availableIcons"
        :CONNECTION_POINTS="CONNECTION_POINTS"
        :CONNECTION_TYPES="CONNECTION_TYPES"
        :NODE_SHAPES="NODE_SHAPES"
        :AVAILABLE_BORDER_STYLES="availableBorderStyles"
        :selectedNodeBgColor="selectedNodeBgColor"
        :selectedNodeTextColor="selectedNodeTextColor"
        @hide-context-menu="hideContextMenu"
        @add-child-node="addNode"
        @open-text-edit-modal="openTextEditModal"
        @trigger-image-upload="triggerImageUpload"
        @open-hyperlink-modal="openHyperlinkModal"
        @set-node-property="args => setNodeProperty(args.nodeId, args.property, args.value)"
        @detach-node="detachNode"
        @start-linking="startLinking"
        @request-delete-node="requestDeleteNode"
        @add-root-node="addRootNode"
        @add-node-at-position="args => addNodeAtPosition(args.x, args.y)"
        @reset-view="resetView"
        @zoom="zoom"
        @set-node-background-color="args => { selectedNodeId = args.nodeId; handleNodeBgColorChangeDirect(args.color); }"
        @set-node-text-color="args => { selectedNodeId = args.nodeId; handleNodeTextColorChangeDirect(args.color); }"
        @set-preview="args => setPreviewConnectionSide(args.nodeId, args.property, args.value)"
        @clear-preview="clearPreviewConnectionSide"
        ref="contextMenuComponentRef"
    />
    <FileMenu
        :visible="fileMenu.visible"
        :top="fileMenu.top"
        :left="fileMenu.left"
        :initialCanvasBgColor="canvasBgColor"
        @reset-application="resetApplication"
        @save-map-to-file="saveMapToFile"
        @update-canvas-bg-color="canvasBgColor = $event"
        @map-file-selected="handleMapFileSelectedDirect"
        @close-file-menu="fileMenu.visible = false"
        ref="fileMenuComponentRef"
    />
  </div>
</template>

<script>
// generateUUID is still needed for creating new nodes.
const generateUUID = () => (typeof crypto !== 'undefined' && crypto.randomUUID) ? crypto.randomUUID() : (Date.now().toString(36) + Math.random().toString(36).substring(2));

// Constants that might be shared or specific to App.vue logic (not drawing)
const MAX_HISTORY_STATES = 50;
const AUTOSAVE_INTERVAL = 50000;
const DEFAULT_CANVAS_BG_COLOR = '#2d3748'; // Used as initial value for canvasBgColor
const NODE_DEFAULT_WIDTH = 120; // Needed for addNodeAtPosition, addInitialNode etc.
const NODE_DEFAULT_HEIGHT = 40; // Needed for addNodeAtPosition, addInitialNode etc.
const CHILD_NODE_SPACING = 40;
// NODE_DEFAULT_BG_COLOR, NODE_DEFAULT_TEXT_COLOR, NODE_DEFAULT_SHAPE might be needed for new nodes
const NODE_DEFAULT_BG_COLOR = '#ffffff';
const NODE_DEFAULT_TEXT_COLOR = '#1f2937';
const NODE_DEFAULT_SHAPE = 'rectangle';
const ICON_SIZE = 16; // For text width calculation if it remains in App.vue

// Constants that are used by context menus, and also by canvas drawing logic
// These are candidates for a shared constants.js file.
// For now, they are duplicated in MindMapCanvas.vue and will be needed here for context menu logic.
const AVAILABLE_ICONS = [
    { name: 'Ampoule', class: 'fas fa-lightbulb', unicode: '\uf0eb' }, { name: 'Étoile', class: 'fas fa-star', unicode: '\uf005' },
    { name: 'Coche', class: 'fas fa-check', unicode: '\uf00c' }, { name: 'Question', class: 'fas fa-question-circle', unicode: '\uf059' },
    { name: 'Attention', class: 'fas fa-exclamation-triangle', unicode: '\uf071' }, { name: 'Info', class: 'fas fa-info-circle', unicode: '\uf05a' },
    { name: 'Lien', class: 'fas fa-link', unicode: '\uf0c1' }, { name: 'Dossier', class: 'fas fa-folder', unicode: '\uf07b' },
    { name: 'Personne', class: 'fas fa-user', unicode: '\uf007' },
];
const CONNECTION_POINTS = ['top', 'bottom', 'left', 'right'];
const CONNECTION_TYPES = ['solid', 'dashed', 'arrow'];
const NODE_SHAPES = ['rectangle', 'ellipse', 'diamond', 'triangle'];
const AVAILABLE_BORDER_STYLES = [
    { name: 'Solide', value: 'solid', dash: [] }, { name: 'Tiretée', value: 'dashed', dash: [7, 4] },
    { name: 'Pointillée', value: 'dotted', dash: [2, 3] },
];


import Modal from './components/Modal.vue';
import Toolbar from './components/Toolbar.vue';
import MindMapCanvas from './components/MindMapCanvas.vue';
import ContextMenu from './components/ContextMenu.vue';
import FileMenu from './components/FileMenu.vue';
import TextEditModal from './components/TextEditModal.vue';
import HyperlinkModal from './components/HyperlinkModal.vue';
import ImagePreviewModal from './components/ImagePreviewModal.vue';
import DeleteConfirmModal from './components/DeleteConfirmModal.vue';

export default {
  name: 'App',
  components: { 'modal': Modal, Toolbar, MindMapCanvas, ContextMenu, FileMenu, TextEditModal, HyperlinkModal, ImagePreviewModal, DeleteConfirmModal },
  data() {
      return {
          nodes: [], selectedNodeId: null, offsetX: 0, offsetY: 0, scale: 1,
          isTextEditModalOpen: false,
          nodeToEditTextId: null,
          nodeTextInput: '',
          isImagePreviewModalOpen: false,
          previewImageUrl: '',
          isDeleteConfirmModalOpen: false,
          nodeToDeleteId: null,
          isErrorModalOpen: false,
          errorModalTitle: 'Erreur',
          errorModalMessage: '',
          errorModalType: 'error', // 'error', 'success', 'info'
          isAboutModalOpen: false,
          isHyperlinkModalOpen: false,
          hyperlinkInput: '',
          nodeToEditHyperlinkId: null,
          contextMenu: { visible: false, x: 0, y: 0, nodeId: null, rawX: 0, rawY: 0 },
          fileMenu: { visible: false, top: '0px', left: '0px' },
          draggedNodeId: null, isDragging: false, isPanning: false, panStart: { x: 0, y: 0 }, dragStart: { x: 0, y: 0 },
          isLinking: false, // For context menu "Connect to another..."
          linkStartNodeId: null, // For context menu "Connect to another..."
          isDraggingLink: false, // For Alt+Drag to create/link node
          linkDragStartNodeId: null, // For Alt+Drag
          linkDragEndPos: null, // For Alt+Drag
          potentialLinkTargetId: null, // For Alt+Drag, highlights node under cursor
          potentialDropTargetId: null, // For node drag reparenting, highlights node under cursor
          hoveredNodeId: null, isAnimating: false, lastMousePos: { clientX: 0, clientY: 0 }, hoveredConnectionId: null, draggingConnectionEnd: null, potentialEndPoint: null,
          canvasBgColor: DEFAULT_CANVAS_BG_COLOR,
          history: [], currentHistoryIndex: -1,
          availableIcons: AVAILABLE_ICONS, availableBorderStyles: AVAILABLE_BORDER_STYLES,
          CONNECTION_POINTS: CONNECTION_POINTS, CONNECTION_TYPES: CONNECTION_TYPES, NODE_SHAPES: NODE_SHAPES,
          previewConnectionSide: null,
          autosaveIntervalId: null,
      }
  },
  computed: {
      dynamicStyles() { return getDynamicDrawingStyles(); },
      selectedNode() { return this.findNodeById(this.selectedNodeId); },
      selectedNodeBgColor: { get() { return this.selectedNode ? this.selectedNode.color : NODE_DEFAULT_BG_COLOR; }, set(value) {} },
      selectedNodeTextColor: { get() { return this.selectedNode ? this.selectedNode.textColor : NODE_DEFAULT_TEXT_COLOR; }, set(value) {} },
      errorModalTypeClass() { if (this.errorModalType === 'success') return 'text-green-400'; if (this.errorModalType === 'info') return 'text-blue-400'; return 'text-red-400'; },
      contextSubmenuOpenLeft() { if (this.$refs.contextMenuElement) { const menuWidth = this.$refs.contextMenuElement.offsetWidth || 200; const submenuWidthEstimate = 180; return this.contextMenu.x + menuWidth + submenuWidthEstimate > window.innerWidth; } return false; },
      fileMenuSubmenuOpenLeft() {
          if (this.$refs.fileMenuElement) {
              const menuRect = this.$refs.fileMenuElement.getBoundingClientRect();
              const submenuWidthEstimate = 200;
              return menuRect.left + menuRect.width + submenuWidthEstimate > window.innerWidth;
          }
          return false;
      },
      canUndo() { return this.currentHistoryIndex > 0; },
      canRedo() { return this.currentHistoryIndex < this.history.length - 1; },
      hasRootNode() { return this.nodes.some(n => n.parentId === null || n.parentId === undefined); },
  },
  watch: {
      canvasBgColor(newValue, oldValue) {
          if(newValue !== oldValue) {
              this.recordHistory("Change Canvas Bg");
              this.redrawCanvas();
          }
      },
      nodes: { handler() { this.requestCanvasRedraw(); }, deep: true },
      offsetX() { this.requestCanvasRedraw(); }, offsetY() { this.requestCanvasRedraw(); }, scale() { this.requestCanvasRedraw(); },
      selectedNodeId() { this.requestCanvasRedraw(); },
      potentialDropTargetId() { this.requestCanvasRedraw(); }, potentialLinkTargetId() { this.requestCanvasRedraw(); },
      hoveredNodeId() { this.requestCanvasRedraw(); }, hoveredConnectionId() { this.requestCanvasRedraw(); },
      draggingConnectionEnd() { this.requestCanvasRedraw(); },
      isDraggingLink() { this.requestCanvasRedraw(); },
      isLinking() { this.requestCanvasRedraw(); },
      previewConnectionSide() { this.requestCanvasRedraw(); },
      canvasBgColor() { this.requestCanvasRedraw(); } // Watch canvasBgColor too
  },
  mounted() {
      // Resize listener for App.vue might not be needed if MindMapCanvas handles its own resize.
      // window.addEventListener('resize', this.resizeApp); // Example if App needed to react to resize
      let didLoad = this.loadMapFromLocalStorage();

      if (!didLoad && !this.hasRootNode) {
          this.$nextTick(() => {
               this.addInitialNode();
          });
      } else if (this.nodes.length > 0 && this.history.length === 0) {
           this.recordHistory("Load Initial Map");
      } else if (this.history.length > 0 && this.currentHistoryIndex === -1 && this.nodes.length > 0){
           this.currentHistoryIndex = this.history.length - 1;
      }

      document.addEventListener('click', this.handleGlobalClick);
      document.addEventListener('keydown', this.handleGlobalKeyDown);

      this.autosaveIntervalId = setInterval(() => {
          this.saveMapToLocalStorage(true);
      }, AUTOSAVE_INTERVAL);

      this.$nextTick(() => {
          this.isTextEditModalOpen = false;
          this.isHyperlinkModalOpen = false;
          this.isImagePreviewModalOpen = false;
          this.isDeleteConfirmModalOpen = false;
          this.isAboutModalOpen = false;
          this.isErrorModalOpen = false;
          this.contextMenu.visible = false;
          this.fileMenu.visible = false;
          this.requestCanvasRedraw(); // Initial redraw after mount
      });
  },
  beforeUnmount() {
      // window.removeEventListener('resize', this.resizeApp);
      document.removeEventListener('click', this.handleGlobalClick);
      document.removeEventListener('keydown', this.handleGlobalKeyDown);
      if (this.autosaveIntervalId) {
          clearInterval(this.autosaveIntervalId);
      }
  },
  methods: {
      toggleFileMenu(buttonElement) { // buttonElement is now passed from Toolbar
          if (this.fileMenu.visible) {
              this.fileMenu.visible = false;
          } else {
              if (buttonElement) {
                const buttonRect = buttonElement.getBoundingClientRect();
                this.fileMenu.top = `${buttonRect.bottom + window.scrollY}px`;
                this.fileMenu.left = `${buttonRect.left + window.scrollX}px`;
                this.fileMenu.visible = true;
                this.hideContextMenu();
              } else {
                console.error("File menu button element not provided to toggleFileMenu");
              }
          }
      },
      // Direct handlers for file inputs, as the file object is emitted directly
      handleImageSelectedDirect(file) {
        if (file) {
            // We need nodeId to associate the image. This logic might need adjustment.
            // For now, let's assume the selectedNodeId is the target.
            // This might require more sophisticated state management if image upload can be triggered
            // for a node that isn't currently selected.
            const nodeIdToUpdate = this.selectedNodeId;
            if (!nodeIdToUpdate) {
                this.showError("Veuillez sélectionner un nœud avant de charger une image.");
                return;
            }
            const reader = new FileReader();
            reader.onload = (e_reader) => {
                const img = new Image();
                img.onload = () => {
                    const node = this.findNodeById(nodeIdToUpdate);
                    if (node) {
                        if (node.imageUrl && node.imageUrl.startsWith('blob:')) {
                            URL.revokeObjectURL(node.imageUrl);
                        }
                        node.imageUrl = img.src;
                        node.imageObj = img;
                        this.recordHistory("Add/Change Node Image");
                    }
                };
                img.onerror = () => {
                    this.showError("Erreur lors du chargement de l'image.");
                    const node = this.findNodeById(nodeIdToUpdate);
                    if (node) {
                        node.imageUrl = null;
                        node.imageObj = null;
                    }
                };
                img.src = e_reader.target.result;
            };
            reader.onerror = () => this.showError("Erreur lors de la lecture du fichier image.");
            reader.readAsDataURL(file);
        }
      },
      handleMapFileSelectedDirect(file) {
        if (!file) return;
        if (!file.type.match('application/json') && !file.name.endsWith('.mapvue')) {
            this.showError("Veuillez sélectionner un fichier .mapvue ou .json valide.");
            return;
        }
        const reader = new FileReader();
        reader.onload = async (e_reader) => {
            try {
                const loadedData = JSON.parse(e_reader.target.result);
                await this.processLoadedData(loadedData, "File");
            } catch (error) {
                this.showError(`Erreur lors du chargement du fichier : ${error.message}`);
                this.resetApplicationState();
            }
        };
        reader.onerror = () => {
            this.showError("Impossible de lire le fichier sélectionné.");
            this.resetApplicationState();
        };
        reader.readAsText(file);
      },
      setPreviewConnectionSide(nodeId, property, value) {
          this.previewConnectionSide = { nodeId, property, value };
      },
      clearPreviewConnectionSide() {
          this.previewConnectionSide = null;
      },
      recordHistory(actionName = "Unknown Action") {
          const serializedNodes = JSON.parse(JSON.stringify(this.nodes.map(n => { const { imageObj, ...rest } = n; return rest; })));
          const state = {
              nodes: serializedNodes,
              viewState: { offsetX: this.offsetX, offsetY: this.offsetY, scale: this.scale },
              selectedNodeId: this.selectedNodeId,
              canvasBgColor: this.canvasBgColor,
          };
          if (this.currentHistoryIndex < this.history.length - 1) {
              this.history = this.history.slice(0, this.currentHistoryIndex + 1);
          }
          this.history.push(state);
          if (this.history.length > MAX_HISTORY_STATES) { this.history.shift(); }
          this.currentHistoryIndex = this.history.length - 1;
      },
      loadStateFromHistory(state) {
          this.nodes = JSON.parse(JSON.stringify(state.nodes)).map(nodeData => {
              const node = { ...nodeData, imageObj: null }; // imageObj will be reloaded if imageUrl exists
              if (node.imageUrl) {
                  const img = new Image();
                  img.onload = () => { 
                      node.imageObj = img; 
                      this.requestCanvasRedraw(); // Request redraw after image loads
                  };
                  img.onerror = () => { node.imageUrl = null; }; // Clear if error
                  img.src = node.imageUrl;
              }
              return node;
          });
          this.offsetX = state.viewState.offsetX; 
          this.offsetY = state.viewState.offsetY;
          this.scale = state.viewState.scale; 
          this.selectedNodeId = state.selectedNodeId;
          this.canvasBgColor = state.canvasBgColor;
          this.requestCanvasRedraw();
      },
      undo() { if (this.canUndo) { this.currentHistoryIndex--; this.loadStateFromHistory(this.history[this.currentHistoryIndex]); } },
      redo() { if (this.canRedo) { this.currentHistoryIndex++; this.loadStateFromHistory(this.history[this.currentHistoryIndex]); } },

      requestCanvasRedraw() {
        this.$nextTick(() => {
          if (this.$refs.mindMapCanvasComponentRef) {
            this.$refs.mindMapCanvasComponentRef.redrawCanvas();
          }
        });
      },
      showError(message, type = 'error', title) { this.errorModalTitle = title || (type === 'success' ? 'Succès' : (type === 'info' ? 'Information' : 'Erreur')); this.errorModalMessage = String(message || "Une erreur inconnue s'est produite."); this.errorModalType = type; this.isErrorModalOpen = true; },
      findNodeById(id) { return this.nodes.find(node => node.id === id); },
      
      // getMousePos is now primarily handled by MindMapCanvas, but App.vue might need a version for context menu, etc.
      // This version uses the event's clientX/Y and the App's stored offsetX/Y/scale.
      getAppMousePos(event) {
        // This method assumes the event's clientX/Y are available.
        // If MindMapCanvas emits world coordinates, this might not be needed or would be simpler.
        // For now, let's assume we need to calculate it based on App.vue's state
        // if the canvas component doesn't provide world coordinates directly in the event payload.
        // However, the canvas component's event handlers (onMouseDown etc.) will have access to its own getMousePos.
        // So, App.vue should rely on the coordinates provided in the event payload from MindMapCanvas.
        // This function is likely redundant if MindMapCanvas emits world coordinates.
        // Let's keep it for now if context menus are positioned relative to the viewport.
        const canvasComponent = this.$refs.mindMapCanvasComponentRef;
        if (!canvasComponent || !canvasComponent.$refs.mindMapCanvasRef) { // Check the actual canvas element ref inside the component
             console.warn("MindMapCanvas or its internal canvas element is not available for getAppMousePos");
             return { x: event.clientX, y: event.clientY, screenX: event.clientX, screenY: event.clientY }; // Fallback
        }
        const rect = canvasComponent.$refs.mindMapCanvasRef.getBoundingClientRect();
        const mx = event.clientX - rect.left;
        const my = event.clientY - rect.top;
        return { 
            x: (mx - this.offsetX) / this.scale, 
            y: (my - this.offsetY) / this.scale, 
            screenX: event.clientX, 
            screenY: event.clientY 
        };
      },

      getNodeAtPos(worldX, worldY) { 
        // This logic relies on node dimensions and positions, so it's fine in App.vue
        // as long as it has access to the `nodes` array.
        for (let i = this.nodes.length - 1; i >= 0; i--) { 
            const n = this.nodes[i]; 
            const shape = n.shape || NODE_DEFAULT_SHAPE; 
            const { x, y, width, height } = n; 
            if (shape === 'rectangle') { if (worldX >= x && worldX <= x + width && worldY >= y && worldY <= y + height) return n; } 
            else if (shape === 'ellipse') { const cx = x + width / 2; const cy = y + height / 2; const rx = width / 2; const ry = height / 2; if (rx > 0 && ry > 0) { const dx = worldX - cx; const dy = worldY - cy; if ((dx * dx) / (rx * rx) + (dy * dy) / (ry * ry) <= 1) return n; } } 
            else if (shape === 'diamond') { const p1 = { x: x + width / 2, y: y }; const p2 = { x: x + width, y: y + height / 2 }; const p3 = { x: x + width / 2, y: y + height }; const p4 = { x: x, y: y + height / 2 }; const d1 = (worldX - p1.x) * (p2.y - p1.y) - (worldY - p1.y) * (p2.x - p1.x); const d2 = (worldX - p2.x) * (p3.y - p2.y) - (worldY - p2.y) * (p3.x - p2.x); const d3 = (worldX - p3.x) * (p4.y - p3.y) - (worldY - p3.y) * (p4.x - p3.x); const d4 = (worldX - p4.x) * (p1.y - p4.y) - (worldY - p4.y) * (p1.x - p4.x); if ((d1 >= 0 && d2 >= 0 && d3 >= 0 && d4 >= 0) || (d1 <= 0 && d2 <= 0 && d3 <= 0 && d4 <= 0)) return n; } 
            else if (shape === 'triangle') { const p1 = { x: x + width / 2, y: y }; const p2 = { x: x + width, y: y + height }; const p3 = { x: x, y: y + height }; const d1 = (worldX - p2.x) * (p1.y - p2.y) - (p1.x - p2.x) * (worldY - p2.y); const d2 = (worldX - p3.x) * (p2.y - p3.y) - (p2.x - p3.x) * (worldY - p3.y); const d3 = (worldX - p1.x) * (p3.y - p1.y) - (p3.x - p1.x) * (worldY - p1.y); const has_neg = (d1 < 0) || (d2 < 0) || (d3 < 0); const has_pos = (d1 > 0) || (d2 > 0) || (d3 > 0); if (!(has_neg && has_pos)) return n; } 
        } 
        return null; 
      },
      isPositionOccupied(x, y, width, height, excludeNodeId) {
          const margin = 5; 
          const checkRect = { x: x - margin, y: y - margin, width: width + 2 * margin, height: height + 2 * margin };
          for (const node of this.nodes) {
              if (node.id === excludeNodeId) continue;
              const nodeRect = { x: node.x, y: node.y, width: node.width, height: node.height };
              if (checkRect.x < nodeRect.x + nodeRect.width &&
                  checkRect.x + checkRect.width > nodeRect.x &&
                  checkRect.y < nodeRect.y + nodeRect.height &&
                  checkRect.y + checkRect.height > nodeRect.y) {
                  return true;
              }
          }
          return false;
      },
      // resizeCanvas is now primarily in MindMapCanvas.vue. App.vue might have a resizeApp if needed.
      // startAnimationLoop is now primarily in MindMapCanvas.vue.
      prepareLinkAnimation(node) { // This modifies node state, which is fine as nodes are in App.vue's data
         if (!node || node.parentId === null) return;
         const parent = this.findNodeById(node.parentId);
         if (parent && !node.isLinkAnimating) {
             // getConnectionPointCoords needs to be available or its results passed.
             // For now, assume it's available or we adapt.
             // This might need to be called from MindMapCanvas or MindMapCanvas emits data for this.
             // For now, let's assume this logic stays and `getConnectionPointCoords` is available.
             // If `getConnectionPointCoords` is moved to MindMapCanvas, this logic needs an event or different flow.
             // **Decision**: For now, keep a version of getConnectionPointCoords in App.vue for this.
             node.linkPrevP0 = this.getConnectionPointCoords(parent, node.parentExitPoint || 'auto');
             node.linkPrevP3 = this.getConnectionPointCoords(node, node.entryPoint || 'auto');
             node.linkAnimationStartTime = Date.now();
             node.isLinkAnimating = true;
             this.requestCanvasRedraw(); // Request redraw as animation flags changed
         }
      },
      resetInteractionStates() {
          this.isDragging = false;
          this.isPanning = false;
          this.draggedNodeId = null;
          this.potentialDropTargetId = null;
          this.draggingConnectionEnd = null;
          this.potentialEndPoint = null;
          this.cancelDragLinking();
          this.potentialLinkTargetId = null;
      },
      addInitialNode() {
          const canvas = this.$refs.mindMapCanvas;
          if (!canvas || canvas.width <= 0 || canvas.height <= 0) {
              this.$nextTick(() => this.addInitialNode());
              return null;
          }
          const nodeId = generateUUID();
          const nodeWidth = NODE_DEFAULT_WIDTH + 40;
          const nodeHeight = NODE_DEFAULT_HEIGHT;
          const nodeX = (canvas.width / 2) - nodeWidth / 2;
          const nodeY = (canvas.height / 2) - nodeHeight / 2;
          const worldX = (nodeX - this.offsetX) / this.scale;
          const worldY = (nodeY - this.offsetY) / this.scale;

          const newNode = {
              id: nodeId, text: "Concept Central",
              x: worldX, y: worldY, width: nodeWidth, height: nodeHeight,
              parentId: null, color: '#4A5568', textColor: '#E2E8F0',
              imageUrl: null, imageObj: null, connectionType: 'solid',
              entryPoint: 'top', parentExitPoint: null, shape: NODE_DEFAULT_SHAPE,
              icon: null, borderStyle: 'solid', borderWidth: 1, hyperlink: null,
              isAppearing: true, animationProgress: 0
          };
          this.nodes.push(newNode);
          this.selectedNodeId = nodeId;
          this.startAnimationLoop();
          this.recordHistory("Add Initial Node");
          return newNode;
      },
      addRootNode() {
          if (this.nodes.some(n => n.parentId === null)) {
              this.showError("Un concept central existe déjà.");
              return null;
          }
          const canvas = this.$refs.mindMapCanvas;
          if (!canvas || canvas.width <= 0 || canvas.height <= 0) { return null; }
          const rootId = generateUUID();
          const nodeWidth = NODE_DEFAULT_WIDTH + 40;
          const nodeHeight = NODE_DEFAULT_HEIGHT;
          const rootX = (this.contextMenu.rawX !== 0 ? this.contextMenu.rawX : (canvas.width / 2 - this.offsetX) / this.scale) - nodeWidth / 2;
          const rootY = (this.contextMenu.rawY !== 0 ? this.contextMenu.rawY : (canvas.height / 2 - this.offsetY) / this.scale) - nodeHeight / 2;
          const newNode = { id: rootId, text: "Concept Central", x: rootX, y: rootY, width: nodeWidth, height: nodeHeight, parentId: null, color: '#4A5568', textColor: '#E2E8F0', imageUrl: null, imageObj: null, connectionType: 'solid', entryPoint: 'top', parentExitPoint: null, shape: NODE_DEFAULT_SHAPE, icon: null, borderStyle: 'solid', borderWidth: 1, hyperlink: null, isAppearing: true, animationProgress: 0 };
          this.nodes.push(newNode);
          this.selectedNodeId = rootId;
          this.startAnimationLoop();
          this.recordHistory("Add Root Node");
          return newNode;
      },
      addNode(parentId = this.selectedNodeId) {
          if (!parentId) { this.showError("Veuillez sélectionner un nœud parent."); return; }
          const parentNode = this.findNodeById(parentId);
          if (!parentNode) return;

          const newNodeId = generateUUID();
          const newNodeWidth = NODE_DEFAULT_WIDTH;
          const newNodeHeight = NODE_DEFAULT_HEIGHT;

          let newNodeX = 0;
          let newNodeY = 0;

          const potentialPositions = [
              { x: parentNode.x + parentNode.width / 2 - newNodeWidth / 2, y: parentNode.y + parentNode.height + CHILD_NODE_SPACING }, // Bottom
              { x: parentNode.x + parentNode.width + CHILD_NODE_SPACING, y: parentNode.y + parentNode.height / 2 - newNodeHeight / 2 }, // Right
              { x: parentNode.x - newNodeWidth - CHILD_NODE_SPACING, y: parentNode.y + parentNode.height / 2 - newNodeHeight / 2 }, // Left
              { x: parentNode.x + parentNode.width / 2 - newNodeWidth / 2, y: parentNode.y - newNodeHeight - CHILD_NODE_SPACING }, // Top
              { x: parentNode.x + parentNode.width + CHILD_NODE_SPACING, y: parentNode.y + parentNode.height + CHILD_NODE_SPACING }, // Bottom-Right
              { x: parentNode.x - newNodeWidth - CHILD_NODE_SPACING, y: parentNode.y + parentNode.height + CHILD_NODE_SPACING }, // Bottom-Left
              { x: parentNode.x + parentNode.width + CHILD_NODE_SPACING, y: parentNode.y - newNodeHeight - CHILD_NODE_SPACING }, // Top-Right
              { x: parentNode.x - newNodeWidth - CHILD_NODE_SPACING, y: parentNode.y - newNodeHeight - CHILD_NODE_SPACING }, // Top-Left
          ];

          let positionFound = false;
          for (const pos of potentialPositions) {
              if (!this.isPositionOccupied(pos.x, pos.y, newNodeWidth, newNodeHeight, parentId)) {
                  newNodeX = pos.x;
                  newNodeY = pos.y;
                  positionFound = true;
                  break;
              }
          }

          if (!positionFound) {
               newNodeX = parentNode.x + parentNode.width / 2 - newNodeWidth / 2;
               newNodeY = parentNode.y + parentNode.height + CHILD_NODE_SPACING * 2; // Fallback further below
          }


          const tempNewNodeForCalc = { x: newNodeX, y: newNodeY, width: newNodeWidth, height: newNodeHeight };
          const { parentExitPoint, entryPoint } = this.calculateDefaultConnectionPoints(parentNode, tempNewNodeForCalc);
          const newNode = { id: newNodeId, text: "Nouveau Nœud", x: newNodeX, y: newNodeY, width: newNodeWidth, height: newNodeHeight, parentId: parentNode.id, color: NODE_DEFAULT_BG_COLOR, textColor: NODE_DEFAULT_TEXT_COLOR, imageUrl: null, imageObj: null, connectionType: 'solid', entryPoint: entryPoint, parentExitPoint: parentExitPoint, shape: NODE_DEFAULT_SHAPE, icon: null, borderStyle: 'solid', borderWidth: 1, hyperlink: null, isAppearing: true, animationProgress: 0 };
          this.nodes.push(newNode);
          this.selectedNodeId = newNodeId;
          this.startAnimationLoop();
          this.hideContextMenu();
          this.recordHistory("Add Node");
      },
      addNodeAtPosition(worldX, worldY) {
          const newNodeId = generateUUID();
          const newNode = { id: newNodeId, text: "Nouveau Nœud", x: worldX - NODE_DEFAULT_WIDTH / 2, y: worldY - NODE_DEFAULT_HEIGHT / 2, width: NODE_DEFAULT_WIDTH, height: NODE_DEFAULT_HEIGHT, parentId: null, color: NODE_DEFAULT_BG_COLOR, textColor: NODE_DEFAULT_TEXT_COLOR, imageUrl: null, imageObj: null, connectionType: 'solid', entryPoint: 'top', parentExitPoint: null, shape: NODE_DEFAULT_SHAPE, icon: null, borderStyle: 'solid', borderWidth: 1, hyperlink: null, isAppearing: true, animationProgress: 0 };
          
          // Removed the check: if (newNode.parentId === null && this.hasRootNode)

          this.nodes.push(newNode);
          this.selectedNodeId = newNodeId;
          this.startAnimationLoop();
          this.hideContextMenu();
          this.recordHistory("Add Node at Position");
      },
      openTextEditModal(nodeIdToEdit = this.selectedNodeId) { 
        if (!nodeIdToEdit) { 
            this.showError("Sélectionnez un nœud pour modifier son texte."); 
            return; 
        } 
        const node = this.findNodeById(nodeIdToEdit); 
        if (node) { 
            this.nodeToEditTextId = node.id; 
            this.nodeTextInput = node.text; // This will be passed as initialText prop
            this.isTextEditModalOpen = true; 
            this.hideContextMenu(); 
        } 
      },
      saveNodeTextFromModal(newText) { // Renamed from saveNodeText, accepts newText
        if (this.nodeToEditTextId) {
            const node = this.findNodeById(this.nodeToEditTextId);
            if (node) {
                const oldText = node.text;
                // newText is received from the event
                if (oldText !== newText) {
                    // Canvas context for text measurement is tricky here.
                    // For simplicity, we might remove dynamic width adjustment based on text content from App.vue
                    // OR pass a utility function to TextEditModal, OR TextEditModal emits desired width.
                    // For now, let's keep the width adjustment logic but acknowledge it's imperfect without direct canvas access.
                    // A more robust solution would involve the MindMapCanvas component providing text measurement.
                    let newWidth = node.width; // Default to current width
                    try {
                        const tempCtx = document.createElement('canvas').getContext('2d');
                        if (tempCtx) {
                            tempCtx.font = '14px Inter, sans-serif'; // Should match canvas font
                            const textLines = newText.split('\n');
                            let maxWidth = 0;
                            textLines.forEach(line => {
                                const tw = tempCtx.measureText(line).width;
                                if (tw > maxWidth) maxWidth = tw;
                            });
                            newWidth = Math.max(NODE_DEFAULT_WIDTH, maxWidth + 40 + (node.icon ? ICON_SIZE + 5 : 0) + (node.hyperlink ? ICON_SIZE + 5 : 0) );
                        }
                    } catch(e) {
                        console.warn("Could not measure text width for node resize:", e);
                    }
                    node.text = newText;
                    node.width = newWidth;
                    this.recordHistory("Edit Node Text");
                }
            }
        }
        this.isTextEditModalOpen = false;
        this.nodeToEditTextId = null;
      },
      requestDeleteNode(nodeId = this.selectedNodeId) { this.hideContextMenu(); if (!nodeId) { this.showError("Sélectionnez un nœud à supprimer."); return; } this.nodeToDeleteId = nodeId; this.isDeleteConfirmModalOpen = true; },
      confirmDeleteNode() {
          if (!this.nodeToDeleteId) return;
          const getSubtreeIds = (pId) => { let ids = []; const children = this.nodes.filter(node => node.parentId === pId); children.forEach(child => { ids.push(child.id); ids = ids.concat(getSubtreeIds(child.id)); }); return ids; };
          const idsToRemove = getSubtreeIds(this.nodeToDeleteId); idsToRemove.push(this.nodeToDeleteId);
          this.nodes = this.nodes.filter(node => !idsToRemove.includes(node.id));
          if (idsToRemove.includes(this.selectedNodeId)) this.selectedNodeId = null;
          if (idsToRemove.includes(this.nodeToEditTextId)) { this.isTextEditModalOpen = false; this.nodeToEditTextId = null; }
          if (this.hoveredNodeId && idsToRemove.includes(this.hoveredNodeId)) this.hoveredNodeId = null;
          if (this.potentialDropTargetId && idsToRemove.includes(this.potentialDropTargetId)) this.potentialDropTargetId = null;
          if (this.potentialLinkTargetId && idsToRemove.includes(this.potentialLinkTargetId)) this.potentialLinkTargetId = null;
          this.nodeToDeleteId = null; this.isDeleteConfirmModalOpen = false;
          this.recordHistory("Delete Node");
      },
      handleNodeBgColorChangeDirect(color) { // Renamed and simplified
        if (this.selectedNodeId) {
            const node = this.findNodeById(this.selectedNodeId);
            if (node && node.color !== color) {
                node.color = color;
                this.recordHistory("Change Node Bg Color");
            }
        }
      },
      handleNodeTextColorChangeDirect(color) { // Renamed and simplified
        if (this.selectedNodeId) {
            const node = this.findNodeById(this.selectedNodeId);
            if (node && node.textColor !== color) {
                node.textColor = color;
                this.recordHistory("Change Node Text Color");
            }
        }
      },
      triggerImageUpload(nodeId = this.selectedNodeId) { 
        this.hideContextMenu(); 
        if (!nodeId) { 
            this.showError("Sélectionnez un nœud pour ajouter une image."); 
            return; 
        }
        // The actual file input is now in Toolbar.vue. 
        // App.vue needs to ask Toolbar to click its input.
        // This requires Toolbar to expose a method or for App.vue to manage a global bus.
        // For now, this specific call for the context menu might need a different approach,
        // or we assume the toolbar's image input is used generally (which it is).
        // If context menu needs its own image upload, it would need its own hidden input.
        // Let's assume for now that `triggerImageUpload` from context menu should use the Toolbar's input.
        // This implies `this.$refs.toolbarRef.$refs.imageInputRef.click()` - but direct child ref access is fragile.
        // A better way is for Toolbar to have a method like `triggerImageUploadForNode(nodeId)`
        // Or, the context menu itself has a hidden input.
        // For now, let's assume the context menu will trigger the *general* image uploader in the toolbar.
        // This means the Toolbar's `image-selected` event will need to know which node it's for.
        // This is getting complex. Simpler: ContextMenu will have its own hidden input for this.
        // The current `triggerImageUpload` in App.vue relies on `this.$refs.imageInput` which is gone.
        // This is now handled by ContextMenu.vue emitting 'trigger-image-upload' and App.vue
        // will then call the toolbar's method to click the input.
        // However, the toolbar's image input is general, not for a specific node from context menu.
        // The most straightforward way is to add a hidden input to ContextMenu.vue for this.
        // For this step, I will assume ContextMenu.vue has such an input and emits a file.
        // The current `triggerImageUpload` here is now mostly a placeholder.
        // App.vue's `handleImageSelectedDirect` is now the main handler.
        console.warn("triggerImageUpload from App.vue called, ensure ContextMenu handles its own input or coordinates with Toolbar.");
        // If ContextMenu has its own input, it would emit 'image-selected-for-node' with {nodeId, file}
        // For now, we'll keep the existing 'image-selected' event from Toolbar and adapt App.vue
        // The ContextMenu will emit 'trigger-image-upload' and `App.vue` will decide which node.
        const toolbar = this.$refs.toolbarRef;
        if (toolbar && typeof toolbar.triggerImageInputForNode === 'function') { // Assuming Toolbar could expose this
            toolbar.triggerImageInputForNode(nodeId); 
        } else if (toolbar && toolbar.$refs.imageInputRef) { // Fallback to direct ref if method not exposed
             this.nodeToAttachImageTo = nodeId; // Store the target node ID
             toolbar.$refs.imageInputRef.click();
        } else {
            this.showError("Fonctionnalité d'upload d'image pour noeud spécifique non implémentée.");
        }
      },
      showImagePreview(imageUrl) { this.previewImageUrl = imageUrl; this.isImagePreviewModalOpen = true; },
      openHyperlinkModal(nodeId = this.selectedNodeId) { 
        this.hideContextMenu(); 
        if (!nodeId) { 
            this.showError("Sélectionnez un nœud pour ajouter un lien."); 
            return; 
        } 
        const node = this.findNodeById(nodeId); 
        if (node) { 
            this.nodeToEditHyperlinkId = nodeId; 
            this.hyperlinkInput = node.hyperlink || ''; // Passed as initialHyperlink
            this.isHyperlinkModalOpen = true; 
        } 
      },
      saveNodeHyperlinkFromModal(newHyperlink) { // Renamed and accepts newHyperlink
        if (this.nodeToEditHyperlinkId) {
            const node = this.findNodeById(this.nodeToEditHyperlinkId);
            if (node) {
                const trimmedLink = newHyperlink.trim();
                if (node.hyperlink !== trimmedLink) {
                    node.hyperlink = trimmedLink || null;
                    this.recordHistory("Add/Edit Hyperlink");
                }
            }
        }
        this.isHyperlinkModalOpen = false;
        this.nodeToEditHyperlinkId = null;
      },
      // requestDeleteNode remains the same, it opens the modal
      // confirmDeleteNode is now primarily triggered by an event from DeleteConfirmModal
      // and the modal itself handles emitting 'close'.
      // App.vue's confirmDeleteNode just needs to do the actual deletion logic.
      // The isDeleteConfirmModalOpen = false will be handled by the @close listener.
      setNodeProperty(nodeId, property, value) {
          const node = this.findNodeById(nodeId);
          let propertyChanged = false;
          if (node) {
              if (property === 'entryPoint' && node.parentId) {
                  if (node.entryPoint !== value) {
                      this.prepareLinkAnimation(node);
                      node.entryPoint = value;
                      const parentNode = this.findNodeById(node.parentId);
                      if (parentNode) {
                          const { parentExitPoint: newParentExit } = this.calculateDefaultConnectionPoints(parentNode, node);
                          node.parentExitPoint = newParentExit;
                      }
                      propertyChanged = true;
                  }
              } else if (property === 'parentExitPoint' && node.parentId) {
                  if (node.parentExitPoint !== value) {
                      this.prepareLinkAnimation(node);
                      node.parentExitPoint = value;
                      const parentNode = this.findNodeById(node.parentId);
                      if (parentNode) {
                          const { entryPoint: newEntryPoint } = this.calculateDefaultConnectionPoints(parentNode, node);
                          node.entryPoint = newEntryPoint;
                      }
                      propertyChanged = true;
                  }
              } else if (node[property] !== value) { // For other properties or if not a link property with a parent
                  if (property === 'connectionType' && node.parentId) { // Also animate for connection type change
                      this.prepareLinkAnimation(node);
                  }
                  node[property] = value;
                  propertyChanged = true;
              }
          }
          this.clearPreviewConnectionSide();
          this.hideContextMenu();
          if (propertyChanged) { this.recordHistory(`Set Node Property: ${property}`); } // RedrawCanvas will be triggered by watcher or animation loop
      },
      detachNode(nodeId) {
          const node = this.findNodeById(nodeId);
          if (node && node.parentId !== null) {
              node.parentId = null;
              node.parentExitPoint = null;
              if (node.isLinkAnimating) {
                  node.isLinkAnimating = false;
                  delete node.linkPrevP0;
                  delete node.linkPrevP3;
                  delete node.linkAnimationStartTime;
              }
              this.recordHistory("Detach Node");
          }
          this.hideContextMenu();
      },
      startLinking(nodeId) { this.hideContextMenu(); const node = this.findNodeById(nodeId); if (node) { this.isLinking = true; this.linkStartNodeId = nodeId; this.showError("Cliquez sur le nœud que vous voulez définir comme nouveau parent.", "info"); } },
      cancelLinking() { this.isLinking = false; this.linkStartNodeId = null; },
      cancelDragLinking() { this.isDraggingLink = false; this.linkDragStartNodeId = null; this.linkDragEndPos = null; this.potentialLinkTargetId = null; },
      resetInteractionStates() {
          const contextMenuEl = this.$refs.contextMenuElement;
          if (this.contextMenu.visible && contextMenuEl && !contextMenuEl.contains(event.target)) {
              this.hideContextMenu();
          }
          const fileMenuEl = this.$refs.fileMenuElement;
          const fileMenuButtonEl = this.$refs.fileMenuButton;
            const fileMenuEl = this.$refs.fileMenuElement; // This ref will be on the FileMenu component itself if extracted
            const toolbarRef = this.$refs.toolbarRef; // Ref to the Toolbar component
            const fileMenuButtonEl = toolbarRef?.$refs.fileMenuButtonRef; // Accessing the button ref inside Toolbar

          if (this.fileMenu.visible && fileMenuEl && !fileMenuEl.contains(event.target) &&
              (!fileMenuButtonEl || (fileMenuButtonEl && !fileMenuButtonEl.contains(event.target))) ) {
              this.fileMenu.visible = false;
          }
      },
      handleGlobalKeyDown(e) {
          if (e.key === 'Escape') {
              if (this.isDraggingLink) this.cancelDragLinking();
              if (this.isLinking) this.cancelLinking();
              if (this.contextMenu.visible) this.hideContextMenu();
              if (this.fileMenu.visible) this.fileMenu.visible = false;
              if (this.isTextEditModalOpen) this.isTextEditModalOpen = false;
              if (this.isImagePreviewModalOpen) this.isImagePreviewModalOpen = false;
              if (this.isDeleteConfirmModalOpen) this.isDeleteConfirmModalOpen = false;
              if (this.isErrorModalOpen) this.isErrorModalOpen = false;
              if (this.isAboutModalOpen) this.isAboutModalOpen = false;
              if (this.isHyperlinkModalOpen) this.isHyperlinkModalOpen = false;
          } else if ((e.ctrlKey || e.metaKey) && e.key === 'z') {
              e.preventDefault(); this.undo();
          } else if ((e.ctrlKey || e.metaKey) && e.key === 'y') {
              e.preventDefault(); this.redo();
          }
      },
      prepareSaveData() { const viewState = { offsetX: this.offsetX, offsetY: this.offsetY, scale: this.scale }; const nodesToSave = this.nodes.map(node => { const { imageObj, ...rest } = node; return rest; }); return { nodes: nodesToSave, view: viewState, canvasBgColor: this.canvasBgColor, history: this.history, currentHistoryIndex: this.currentHistoryIndex }; },
      saveMapToLocalStorage(isAutosave = false) { if (!isAutosave) { if(this.fileMenu.visible) this.fileMenu.visible = false; } try { const dataToSave = this.prepareSaveData(); localStorage.setItem('mindMapDataVue', JSON.stringify(dataToSave)); if (!isAutosave) { this.showError('Carte sauvegardée dans le navigateur !', 'success'); } else { console.log("Autosave to localStorage successful at " + new Date().toLocaleTimeString()); } } catch (error) { if (error.name === 'QuotaExceededError') { this.showError("Erreur d'auto-sauvegarde : Stockage local plein. La sauvegarde automatique est désactivée.", 'error'); if (this.autosaveIntervalId) { clearInterval(this.autosaveIntervalId); this.autosaveIntervalId = null; console.warn("Autosave stopped due to QuotaExceededError."); } } else { if (!isAutosave) { this.showError("Erreur sauvegarde locale.", 'error'); } console.error("Save to localStorage error:", error); } } },
      saveMapToFile() { if(this.fileMenu.visible) this.fileMenu.visible = false; try { const dataToSave = this.prepareSaveData(); const dataString = JSON.stringify(dataToSave, null, 2); const blob = new Blob([dataString], { type: 'application/json' }); const url = URL.createObjectURL(blob); const link = document.createElement('a'); link.href = url; const rootNode = this.nodes.find(n => n.parentId === null); const filename = rootNode ? `${rootNode.text.substring(0, 20).replace(/[^a-z0-9]/gi, '_')}.mapvue` : 'carte_mentale.mapvue'; link.download = filename; document.body.appendChild(link); link.click(); document.body.removeChild(link); URL.revokeObjectURL(url); } catch (error) { this.showError("Erreur lors de la création du fichier .mapvue."); } },
      async processLoadedData(loadedData, from = "unknown") {
          this.nodes.forEach(node => { if (node.imageUrl && node.imageUrl.startsWith('blob:')) URL.revokeObjectURL(node.imageUrl); });
          this.nodes = []; this.history = []; this.currentHistoryIndex = -1; this.selectedNodeId = null;
          let loadSuccessful = false;

          if (!loadedData || !Array.isArray(loadedData.nodes) || !loadedData.view) {
              console.log("No valid data found in storage or file.");
              this.offsetX = 0; this.offsetY = 0; this.scale = 1; this.canvasBgColor = DEFAULT_CANVAS_BG_COLOR;
              this.redrawCanvas();
              return loadSuccessful; // Return false as loading was not successful
          }

          this.offsetX = loadedData.view.offsetX || 0;
          this.offsetY = loadedData.view.offsetY || 0;
          this.scale = loadedData.view.scale || 1;
          this.canvasBgColor = loadedData.canvasBgColor || DEFAULT_CANVAS_BG_COLOR;

          let loadError = false;
          const imageLoadPromises = loadedData.nodes.map(nodeData => {
              return new Promise((resolve) => {
                  if (!nodeData || typeof nodeData.id === 'undefined' || nodeData.id === null) { resolve(null); return; }
                  const node = { ...nodeData, imageObj: null };
                  if (node.imageUrl && node.imageUrl.startsWith('data:image')) {
                      const img = new Image();
                      img.onload = () => { node.imageObj = img; resolve(node); };
                      img.onerror = () => { node.imageUrl = null; loadError = true; resolve(node); };
                      img.src = node.imageUrl;
                  } else {
                      node.imageUrl = null;
                      resolve(node);
                  }
              });
          });

          try {
              const resolvedNodes = await Promise.all(imageLoadPromises);
              this.nodes = resolvedNodes.filter(Boolean);
              if (loadError) this.showError("Certaines images n'ont pas pu être chargées.");
              this.selectedNodeId = this.nodes.length > 0 ? (this.nodes.find(n => n.parentId === null)?.id || this.nodes[0]?.id) : null;

              if (loadedData.history && from !== "history_navigation") {
                  this.history = loadedData.history;
                  this.currentHistoryIndex = loadedData.currentHistoryIndex !== undefined ? loadedData.currentHistoryIndex : (loadedData.history.length > 0 ? loadedData.history.length - 1 : -1);
              } else if (from !== "history_navigation" && this.nodes.length > 0) {
                   this.recordHistory("Load Map from " + from);
              }
              this.redrawCanvas();
              loadSuccessful = true;
          } catch (error) {
              console.error("Error processing loaded images:", error);
              this.showError("Une erreur s'est produite lors du chargement des images de la carte.");
              this.resetApplicationState();
              loadSuccessful = false;
          }
          return loadSuccessful;
      },
      async loadMapFromLocalStorage() { // Made async
          const savedDataString = localStorage.getItem('mindMapDataVue');
          let loadedSuccessfully = false;
          if (savedDataString) {
              try {
                  const savedData = JSON.parse(savedDataString);
                  loadedSuccessfully = await this.processLoadedData(savedData, "LocalStorage"); // await here
              } catch (error) {
                  console.error("Error parsing localStorage data:", error);
                  localStorage.removeItem('mindMapDataVue');
                  this.resetApplicationState();
                  loadedSuccessfully = false;
              }
          } else {
              this.resetApplicationState();
              loadedSuccessfully = false;
          }
          return loadedSuccessfully; // Return the result
      },
      handleMapFileSelected(event) {
          if(this.fileMenu.visible) this.fileMenu.visible = false;
          const file = event.target.files[0]; if (!file) return;
          if (!file.type.match('application/json') && !file.name.endsWith('.mapvue')) { this.showError("Veuillez sélectionner un fichier .mapvue ou .json valide."); event.target.value = ''; return; }
          const reader = new FileReader();
          reader.onload = async (e_reader) => {
              try {
                  const loadedData = JSON.parse(e_reader.target.result);
                  await this.processLoadedData(loadedData, "File");
              } catch (error) {
                  this.showError(`Erreur lors du chargement du fichier : ${error.message}`);
                  this.resetApplicationState();
              } finally {
                  event.target.value = '';
              }
          };
          reader.onerror = () => {
              this.showError("Impossible de lire le fichier sélectionné.");
              this.resetApplicationState();
          }
          reader.readAsText(file);
      },
      exportToPng() { this.hideContextMenu(); if(this.fileMenu.visible) this.fileMenu.visible = false; if (this.nodes.length === 0) { this.showError("Rien à exporter."); return;} const canvas = this.$refs.mindMapCanvas; try { const dataUrl = canvas.toDataURL('image/png'); const link = document.createElement('a'); link.href = dataUrl; const rootNode = this.nodes.find(n => n.parentId === null); const filename = rootNode ? `${rootNode.text.substring(0, 20).replace(/[^a-z0-9]/gi, '_')}.png` : 'carte-mentale.png'; link.download = filename; document.body.appendChild(link); link.click(); document.body.removeChild(link); } catch (e) { this.showError("Erreur lors de l'exportation en PNG."); } },
      resetApplicationState() {
          this.nodes.forEach(node => { if (node.imageUrl && node.imageUrl.startsWith('blob:')) URL.revokeObjectURL(node.imageUrl); });
          this.nodes = [];
          this.history = [];
          this.currentHistoryIndex = -1;
          this.selectedNodeId = null;
          this.offsetX = 0;
          this.offsetY = 0;
          this.scale = 1;
          this.canvasBgColor = DEFAULT_CANVAS_BG_COLOR;
          this.previewConnectionSide = null;
          this.hideContextMenu();
          if(this.fileMenu) this.fileMenu.visible = false;
          this.isErrorModalOpen = false;
      },
      resetApplication() {
          this.resetApplicationState();
          this.$nextTick(() => {
              this.addInitialNode();
              this.redrawCanvas();
          });
      },
      getNodeIconPreview(iconClass) { if (!iconClass) return 'Aucune'; const iconObj = AVAILABLE_ICONS.find(i => i.class === iconClass); return iconObj ? iconObj.name : 'Inconnue'; },
      getNodeShapeIcon(shape) { if (shape === 'ellipse') return 'far fa-circle'; if (shape === 'diamond') return 'far fa-gem'; if (shape === 'triangle') return 'fas fa-play fa-rotate-270'; return 'far fa-square'; },
      getConnectionTypeIndicator(type) { if (type === 'solid') return '⎯'; if (type === 'dashed') return '- -'; if (type === 'arrow') return '->'; return '?'; },
      getConnectionPointCoords(node, pointName) { if (!node) return { x: 0, y: 0 }; const shape = node.shape || NODE_DEFAULT_SHAPE; const { x, y, width: w, height: h } = node; const cx = x + w / 2; const cy = y + h / 2; if (shape === 'rectangle' || shape === 'diamond') { switch (pointName) { case 'top': return { x: cx, y: y }; case 'bottom': return { x: cx, y: y + h }; case 'left': return { x: x, y: cy }; case 'right': return { x: x + w, y: cy }; default: return { x: cx, y: y }; } } else if (shape === 'ellipse') { const rx = w / 2; const ry = h / 2; if (rx <= 0 || ry <=0) return {x: cx, y: cy}; switch (pointName) { case 'top': return { x: cx, y: cy - ry }; case 'bottom': return { x: cx, y: cy + ry }; case 'left': return { x: cx - rx, y: cy }; case 'right': return { x: cx + rx, y: cy }; default: return { x: cx, y: cy - ry }; } } else if (shape === 'triangle') { switch (pointName) { case 'top': return { x: cx, y: y }; case 'bottom': return { x: cx, y: y + h }; case 'left': return { x: x + w * 0.25, y: y + h * 0.65 }; case 'right': return { x: x + w * 0.75, y: y + h * 0.65 }; default: return { x: cx, y: y }; } } return { x: cx, y: y }; },
      isAncestor(potentialChildId, potentialParentId) { let currentId = potentialParentId; let visited = new Set(); while (currentId !== null) { if (visited.has(currentId)) return true; visited.add(currentId); const node = this.findNodeById(currentId); if (!node) return false; if (node.parentId === potentialChildId) return true; currentId = node.parentId; } return false; },
      calculateDefaultConnectionPoints(parentNode, childNode) { const dx = (childNode.x + childNode.width / 2) - (parentNode.x + parentNode.width / 2); const dy = (childNode.y + childNode.height / 2) - (parentNode.y + parentNode.height / 2); if (Math.abs(dx) > Math.abs(dy)) return dx > 0 ? { parentExitPoint: 'right', entryPoint: 'left' } : { parentExitPoint: 'left', entryPoint: 'right' }; else return dy > 0 ? { parentExitPoint: 'bottom', entryPoint: 'top' } : { parentExitPoint: 'top', entryPoint: 'bottom' }; },
      
      // Simplified getConnectionPointCoords for App.vue (might only be needed for non-drawing logic if any)
      // The main drawing version is in MindMapCanvas.vue
      getConnectionPointCoords(node, pointName) {
        if (!node) return { x: 0, y: 0 };
        const { x, y, width: w, height: h } = node;
        const cx = x + w / 2;
        const cy = y + h / 2;
        // Simplified version for logic, detailed shape-specifics in MindMapCanvas
        switch (pointName) {
            case 'top': return { x: cx, y: y };
            case 'bottom': return { x: cx, y: y + h };
            case 'left': return { x: x, y: cy };
            case 'right': return { x: x + w, y: cy };
            default: return { x: cx, y: y };
        }
      },

      getClickedConnectionEnd(pos) { 
        const REPARENT_PROXIMITY_THRESHOLD = 60; // Keep relevant constants or pass them
        const CONNECTION_END_HOVER_RADIUS = 8;
        const radius = CONNECTION_END_HOVER_RADIUS / this.scale; 
        const radiusSq = radius * radius; 
        for (const child of this.nodes) { 
            if (child.parentId !== null) { 
                const parent = this.findNodeById(child.parentId); 
                if (parent) { 
                    const startCoords = this.getConnectionPointCoords(parent, child.parentExitPoint || 'bottom'); 
                    const endCoords = this.getConnectionPointCoords(child, child.entryPoint || 'top'); 
                    let dx1 = pos.x - startCoords.x; let dy1 = pos.y - startCoords.y; 
                    if (dx1 * dx1 + dy1 * dy1 <= radiusSq) return { childId: child.id, endType: 'parentExit', nodeToAttach: parent }; 
                    let dx2 = pos.x - endCoords.x; let dy2 = pos.y - endCoords.y; 
                    if (dx2 * dx2 + dy2 * dy2 <= radiusSq) return { childId: child.id, endType: 'childEntry', nodeToAttach: child }; 
                } 
            } 
        } 
        return null; 
      },
      getNodeWithImageIndicatorClick(wx, wy) { 
        const IMAGE_INDICATOR_SIZE = 15; // Keep relevant constants
        for (let i = this.nodes.length - 1; i >= 0; i--) { 
            const n = this.nodes[i]; 
            if (!n.imageUrl || !n.imageObj) continue; 
            const rWorld = (IMAGE_INDICATOR_SIZE / 2); 
            const indicatorScreenX = n.x + n.width - (IMAGE_INDICATOR_SIZE / 2); 
            const indicatorScreenY = n.y + (IMAGE_INDICATOR_SIZE / 2); 
            const dx = wx - indicatorScreenX; const dy = wy - indicatorScreenY; 
            if (dx * dx + dy * dy <= rWorld * rWorld) return n; 
        } 
        return null; 
      },
      getHoveredConnectionId(pos) { 
        const hoverDistThreshold = 5 / this.scale; 
        const hoverDistThresholdSq = hoverDistThreshold * hoverDistThreshold; 
        const numCheckPoints = 10; 
        for (const child of this.nodes) { 
            if (child.parentId !== null) { 
                const parent = this.findNodeById(child.parentId); 
                if (parent) { 
                    const startCoords = this.getConnectionPointCoords(parent, child.parentExitPoint || 'bottom'); 
                    const endCoords = this.getConnectionPointCoords(child, child.entryPoint || 'top'); 
                    const P0 = startCoords; const P3 = endCoords; 
                    let P1 = { ...P0 }, P2 = { ...P3 }; 
                    const fixedOffset = 50; 
                    switch (child.parentExitPoint || 'bottom') { case 'top': P1.y -= fixedOffset; break; case 'bottom': P1.y += fixedOffset; break; case 'left': P1.x -= fixedOffset; break; case 'right': P1.x += fixedOffset; break;} 
                    switch (child.entryPoint || 'top') { case 'top': P2.y -= fixedOffset; break; case 'bottom': P2.y += fixedOffset; break; case 'left': P2.x -= fixedOffset; break; case 'right': P2.x += fixedOffset; break;} 
                    for (let i = 1; i < numCheckPoints; i++) { 
                        const t = i / numCheckPoints; 
                        const mt = 1 - t; 
                        const x = mt*mt*mt*P0.x + 3*mt*mt*t*P1.x + 3*mt*t*t*P2.x + t*t*t*P3.x; 
                        const y = mt*mt*mt*P0.y + 3*mt*mt*t*P1.y + 3*mt*t*t*P2.y + t*t*t*P3.y; 
                        if ((pos.x - x) ** 2 + (pos.y - y) ** 2 <= hoverDistThresholdSq) return child.id; 
                    } 
                } 
            } 
        } 
        return null; 
      },

      // Canvas Event Handlers (now receive payload from MindMapCanvas component)
      handleCanvasMousedown(payload) {
          const { event, worldPos } = payload; // Assuming MindMapCanvas emits worldPos
          this.lastMousePos = { clientX: event.clientX, clientY: event.clientY }; // Keep screen pos for panning

          const contextMenuEl = this.$refs.contextMenuElement;
          if (this.contextMenu.visible && (!contextMenuEl || !contextMenuEl.contains(event.target))) {
              this.hideContextMenu();
          }
          // File menu hiding logic (already present)

          const clickedConnEnd = this.getClickedConnectionEnd(worldPos);
          if (clickedConnEnd) {
              this.draggingConnectionEnd = clickedConnEnd;
              this.isDragging = false; this.isPanning = false; this.draggedNodeId = null;
              this.potentialDropTargetId = null; this.isDraggingLink = false;
              // Cursor style change will be handled by MindMapCanvas based on its state or this state change
              return;
          }

          if (this.isLinking) {
              event.stopPropagation();
              const clickedNode = this.getNodeAtPos(worldPos.x, worldPos.y);
              const childNode = this.findNodeById(this.linkStartNodeId);
              if (clickedNode && childNode && clickedNode.id !== childNode.id && !this.isAncestor(childNode.id, clickedNode.id)) {
                  const calculatedPoints = this.calculateDefaultConnectionPoints(clickedNode, childNode);
                  childNode.parentId = clickedNode.id;
                  childNode.parentExitPoint = calculatedPoints.parentExitPoint;
                  if (!childNode.entryPoint || childNode.entryPoint === 'auto') {
                      childNode.entryPoint = calculatedPoints.entryPoint;
                  }
                  this.recordHistory("Link Node (Context Menu)");
              } else if (clickedNode) {
                  this.showError("Connexion invalide (ex: connexion à soi-même ou à un descendant).");
              }
              this.cancelLinking();
              return;
          }

          const clickedImageIndicatorNode = this.getNodeWithImageIndicatorClick(worldPos.x, worldPos.y);
          if (clickedImageIndicatorNode) {
              this.showImagePreview(clickedImageIndicatorNode.imageUrl);
              this.resetInteractionStates();
              return;
          }

          const clickedNodeForLink = this.getNodeAtPos(worldPos.x, worldPos.y);
          if (event.altKey && clickedNodeForLink) {
              this.isDraggingLink = true; this.linkDragStartNodeId = clickedNodeForLink.id;
              this.linkDragEndPos = worldPos; // Start with current worldPos
              this.resetInteractionStates(); // Clear other states
              this.isDraggingLink = true; this.linkDragStartNodeId = clickedNodeForLink.id; // Re-set after resetInteractionStates
              this.linkDragEndPos = worldPos;
              return;
          }

          this.dragStart = worldPos;
          const clickedNode = this.getNodeAtPos(worldPos.x, worldPos.y);
          if (clickedNode) {
              this.isDragging = true; this.draggedNodeId = clickedNode.id;
              if (this.selectedNodeId !== clickedNode.id) this.selectedNodeId = clickedNode.id;
          } else {
              this.isPanning = true;
              this.panStart = { x: event.clientX - this.offsetX, y: event.clientY - this.offsetY };
              if (this.selectedNodeId !== null) this.selectedNodeId = null;
          }
      },
      handleCanvasMousemove(payload) {
          const { event, worldPos } = payload;
          this.lastMousePos = { clientX: event.clientX, clientY: event.clientY }; // Update screen pos

          if (this.isLinking) { return; }

          this.potentialLinkTargetId = null; 
          this.potentialDropTargetId = null;

          if (this.isDraggingLink) {
              this.linkDragEndPos = worldPos;
              const currentPotentialTarget = this.getNodeAtPos(worldPos.x, worldPos.y);
              let newPotentialLinkTargetId = null;
              if (currentPotentialTarget && currentPotentialTarget.id !== this.linkDragStartNodeId && !this.isAncestor(this.linkDragStartNodeId, currentPotentialTarget.id)) {
                  newPotentialLinkTargetId = currentPotentialTarget.id;
              }
              this.potentialLinkTargetId = newPotentialLinkTargetId;
          } else if (this.draggingConnectionEnd) {
              const draggedLinkOriginalChild = this.findNodeById(this.draggingConnectionEnd.childId);
              const draggedLinkOriginalParent = this.findNodeById(draggedLinkOriginalChild.parentId);
              let newHoveredNodeForEndpoint = this.getNodeAtPos(worldPos.x, worldPos.y);
              let finalNodeForSideCalc;
              let isValidNewTarget = false;

              if (newHoveredNodeForEndpoint) {
                  if (this.draggingConnectionEnd.endType === 'parentExit') {
                      if (newHoveredNodeForEndpoint.id !== draggedLinkOriginalChild.id &&
                          !this.isAncestor(draggedLinkOriginalChild.id, newHoveredNodeForEndpoint.id)) {
                          isValidNewTarget = true;
                      }
                  } else { 
                      if (newHoveredNodeForEndpoint.id !== draggedLinkOriginalParent.id &&
                          !this.isAncestor(draggedLinkOriginalParent.id, newHoveredNodeForEndpoint.id)) {
                          isValidNewTarget = true;
                      }
                  }
              }
              finalNodeForSideCalc = isValidNewTarget ? newHoveredNodeForEndpoint : this.draggingConnectionEnd.nodeToAttach;
              
              let closestSide = null; let minDistSq = Infinity; let tempCoords = null;
              // CONNECTION_POINTS should be available in App.vue
              CONNECTION_POINTS.forEach(side => {
                  const sideCoords = this.getConnectionPointCoords(finalNodeForSideCalc, side);
                  const distSq = (worldPos.x - sideCoords.x) ** 2 + (worldPos.y - sideCoords.y) ** 2;
                  if (distSq < minDistSq) { minDistSq = distSq; closestSide = side; tempCoords = sideCoords; }
              });
              this.potentialEndPoint = { node: finalNodeForSideCalc, side: closestSide, coords: tempCoords };
          } else if (this.isDragging && this.draggedNodeId) {
              const draggedNode = this.findNodeById(this.draggedNodeId);
              if (draggedNode) {
                  const dx = (worldPos.x - this.dragStart.x);
                  const dy = (worldPos.y - this.dragStart.y);
                  this.prepareLinkAnimation(draggedNode);
                  this.nodes.forEach(n => { if (n.parentId === draggedNode.id) this.prepareLinkAnimation(n); });
                  draggedNode.x += dx;
                  draggedNode.y += dy;
                  this.dragStart = worldPos;
              }
              let currentPotentialTarget = null; let minDistanceSq = Infinity; 
              const REPARENT_PROXIMITY_THRESHOLD = 60; // Keep relevant constants
              const thresholdSq = (REPARENT_PROXIMITY_THRESHOLD / this.scale) ** 2; 
              this.nodes.forEach(targetNode => { 
                  if (targetNode.id === this.draggedNodeId) return; 
                  const targetCenterX = targetNode.x + targetNode.width / 2; 
                  const targetCenterY = targetNode.y + targetNode.height / 2; 
                  const distSq = (worldPos.x - targetCenterX) ** 2 + (worldPos.y - targetCenterY) ** 2; 
                  if (distSq < minDistanceSq && distSq <= thresholdSq) { 
                      const draggedNodeData = this.findNodeById(this.draggedNodeId); 
                      if (draggedNodeData && targetNode.id !== draggedNodeData.parentId && !this.isAncestor(this.draggedNodeId, targetNode.id)) { 
                          minDistanceSq = distSq; currentPotentialTarget = targetNode.id; 
                      } 
                  } 
              }); 
              if (this.potentialDropTargetId !== currentPotentialTarget) this.potentialDropTargetId = currentPotentialTarget;
          } else if (this.isPanning) {
              this.offsetX = event.clientX - this.panStart.x;
              this.offsetY = event.clientY - this.panStart.y;
          } else {
              const nodeOver = this.getNodeAtPos(worldPos.x, worldPos.y);
              this.hoveredNodeId = nodeOver ? nodeOver.id : null;
              const connOver = this.getHoveredConnectionId(worldPos);
              this.hoveredConnectionId = connOver;
              // Cursor style updates will be implicitly handled by MindMapCanvas via prop changes
          }
      },
      handleCanvasMouseup(payload) {
          const { event, worldPos } = payload; // Assuming worldPos is provided
          const wasDragging = this.isDragging; 
          let stateChanged = false;
          
          if (this.isDraggingLink) {
              const sourceNode = this.findNodeById(this.linkDragStartNodeId);
              if (sourceNode) {
                  if (this.potentialLinkTargetId) {
                      const targetNode = this.findNodeById(this.potentialLinkTargetId);
                      if (targetNode && sourceNode.id !== targetNode.id && !this.isAncestor(sourceNode.id, targetNode.id)) {
                          if(sourceNode.parentId !== targetNode.id) {
                              this.prepareLinkAnimation(sourceNode);
                              const { parentExitPoint, entryPoint } = this.calculateDefaultConnectionPoints(targetNode, sourceNode);
                              sourceNode.parentId = targetNode.id; sourceNode.parentExitPoint = parentExitPoint; sourceNode.entryPoint = entryPoint;
                              stateChanged = true;
                          }
                      }
                  } else if (this.linkDragEndPos) { // linkDragEndPos is worldPos here
                      const newNodeId = generateUUID();
                      const newNodeX = this.linkDragEndPos.x - NODE_DEFAULT_WIDTH / 2;
                      const newNodeY = this.linkDragEndPos.y - NODE_DEFAULT_HEIGHT / 2;
                      const tempNewNodeForCalc = { x: newNodeX, y: newNodeY, width: NODE_DEFAULT_WIDTH, height: NODE_DEFAULT_HEIGHT };
                      const { parentExitPoint, entryPoint } = this.calculateDefaultConnectionPoints(sourceNode, tempNewNodeForCalc);
                      const newNode = { id: newNodeId, text: "Nouveau Nœud", x: newNodeX, y: newNodeY, width: NODE_DEFAULT_WIDTH, height: NODE_DEFAULT_HEIGHT, parentId: sourceNode.id, color: NODE_DEFAULT_BG_COLOR, textColor: NODE_DEFAULT_TEXT_COLOR, imageUrl: null, imageObj: null, connectionType: 'solid', entryPoint: entryPoint, parentExitPoint: parentExitPoint, shape: NODE_DEFAULT_SHAPE, icon: null, borderStyle: 'solid', borderWidth: 1, hyperlink: null, isAppearing: true, animationProgress: 0 };
                      this.nodes.push(newNode);
                      this.selectedNodeId = newNodeId;
                      // startAnimationLoop will be triggered by MindMapCanvas due to node.isAppearing
                      stateChanged = true;
                  }
              }
              this.cancelDragLinking();
          } else if (this.draggingConnectionEnd && this.potentialEndPoint) {
              // ... (logic for updating connection endpoints, largely remains the same but uses findNodeById, calculateDefaultConnectionPoints from App.vue)
              // Ensure this.prepareLinkAnimation is called appropriately.
              const linkedChildNode = this.findNodeById(this.draggingConnectionEnd.childId);
              const originalParentNode = this.findNodeById(linkedChildNode.parentId);
              const newEndpointNode = this.potentialEndPoint.node;
              const newEndpointSide = this.potentialEndPoint.side;

              if (this.draggingConnectionEnd.endType === 'parentExit') {
                  const newParentCandidate = newEndpointNode;
                  if (newParentCandidate.id !== originalParentNode.id || linkedChildNode.parentExitPoint !== newEndpointSide) {
                      this.prepareLinkAnimation(linkedChildNode);
                      linkedChildNode.parentId = newParentCandidate.id;
                      linkedChildNode.parentExitPoint = newEndpointSide;
                      const { entryPoint: newChildEntryPoint } = this.calculateDefaultConnectionPoints(newParentCandidate, linkedChildNode);
                      linkedChildNode.entryPoint = newChildEntryPoint;
                      stateChanged = true;
                  }
              } else { // 'childEntry'
                  const newChildCandidate = newEndpointNode;
                  if (newChildCandidate.id !== linkedChildNode.id) {
                      this.prepareLinkAnimation(newChildCandidate);
                      newChildCandidate.parentId = originalParentNode.id;
                      newChildCandidate.entryPoint = newEndpointSide;
                      const { parentExitPoint: newParentExitForCPrime } = this.calculateDefaultConnectionPoints(originalParentNode, newChildCandidate);
                      newChildCandidate.parentExitPoint = newParentExitForCPrime;
                      if (linkedChildNode.parentId === originalParentNode.id) {
                          this.prepareLinkAnimation(linkedChildNode);
                          linkedChildNode.parentId = null;
                          linkedChildNode.parentExitPoint = null;
                      }
                      stateChanged = true;
                  } else {
                      if (linkedChildNode.entryPoint !== newEndpointSide) {
                          this.prepareLinkAnimation(linkedChildNode);
                          linkedChildNode.entryPoint = newEndpointSide;
                          const { parentExitPoint: newParentExitForC } = this.calculateDefaultConnectionPoints(originalParentNode, linkedChildNode);
                          linkedChildNode.parentExitPoint = newParentExitForC;
                          stateChanged = true;
                      }
                  }
              }
          } else if (wasDragging && this.draggedNodeId && this.potentialDropTargetId) {
              const draggedNode = this.findNodeById(this.draggedNodeId);
              const targetNode = this.findNodeById(this.potentialDropTargetId);
              if (draggedNode && targetNode && draggedNode.id !== targetNode.id && draggedNode.parentId !== targetNode.id && !this.isAncestor(draggedNode.id, targetNode.id)) {
                  this.prepareLinkAnimation(draggedNode);
                  const calculatedPoints = this.calculateDefaultConnectionPoints(targetNode, draggedNode);
                  draggedNode.parentId = targetNode.id; 
                  draggedNode.parentExitPoint = calculatedPoints.parentExitPoint;
                  if(!draggedNode.entryPoint || draggedNode.entryPoint === 'auto') { 
                      draggedNode.entryPoint = calculatedPoints.entryPoint; 
                  }
                  stateChanged = true;
              }
          }
          if (wasDragging && this.draggedNodeId && !this.potentialDropTargetId) { stateChanged = true; }
          
          this.resetInteractionStates();

          if (stateChanged) {
              this.recordHistory(this.draggingConnectionEnd ? "Modify Link Attachment" : "Node Interaction/Move");
          }
          this.draggingConnectionEnd = null; 
          this.potentialEndPoint = null;
          // Cursor updates will be handled by MindMapCanvas based on prop changes
      },
      handleCanvasMouseout(payload) {
          // const { event } = payload; // Not strictly needed if MindMapCanvas handles its own boundary checks
          this.resetInteractionStates(); 
          this.hoveredConnectionId = null; 
          this.hoveredNodeId = null;
      },
      handleCanvasDblclick(payload) {
          const { event, worldPos } = payload;
          // getClickedConnectionEnd is now in App.vue
          if (this.isLinking || this.isDraggingLink || this.getClickedConnectionEnd(worldPos)) return; 
          this.hideContextMenu();
          const clickedNode = this.getNodeAtPos(worldPos.x, worldPos.y);
          if (clickedNode) {
              this.selectedNodeId = clickedNode.id;
              let iconAreaXStart = clickedNode.x + 5; // ICON_SIZE needs to be available
              let clickedOnLink = false;
              if (clickedNode.icon) iconAreaXStart += ICON_SIZE + 5;
              if (clickedNode.hyperlink) {
                  const linkIconX = iconAreaXStart;
                  const linkIconWidth = ICON_SIZE;
                  if (worldPos.x >= linkIconX && worldPos.x <= linkIconX + linkIconWidth && 
                      worldPos.y >= clickedNode.y + clickedNode.height/2 - ICON_SIZE/2 && 
                      worldPos.y <= clickedNode.y + clickedNode.height/2 + ICON_SIZE/2) {
                      clickedOnLink = true;
                  }
              }
              if (clickedOnLink) { window.open(clickedNode.hyperlink, '_blank'); return; }
              if (clickedNode.imageUrl) this.showImagePreview(clickedNode.imageUrl);
              else this.openTextEditModal(clickedNode.id);
          }
      },
      handleCanvasWheel(payload) {
          const { event } = payload; // clientX, clientY are on original event
          if (this.isLinking || this.isDraggingLink) return;
          this.hideContextMenu();
          const oldScale = this.scale;
          const delta = event.deltaY > 0 ? 0.9 : 1.1;
          
          // Zoom logic needs access to canvas dimensions for mouse-pointer zooming
          // This might need to be adjusted or canvas dimensions passed.
          // For now, simplified zoom (center zoom) or pass clientX/Y
          const canvasEl = this.$refs.mindMapCanvasComponentRef?.$refs.mindMapCanvasRef;
          let mx, my;
          if (canvasEl) {
            const rect = canvasEl.getBoundingClientRect();
            mx = event.clientX - rect.left;
            my = event.clientY - rect.top;
          } else { // Fallback to center zoom if canvas ref not available
            mx = (this.$el?.clientWidth || window.innerWidth) / 2; // Approximate
            my = (this.$el?.clientHeight || window.innerHeight) / 2; // Approximate
          }

          const wxBefore = (mx - this.offsetX) / this.scale;
          const wyBefore = (my - this.offsetY) / this.scale;
          const newScale = Math.max(0.1, Math.min(this.scale * factor, 5)); // factor was delta
          this.offsetX = mx - wxBefore * newScale;
          this.offsetY = my - wyBefore * newScale;
          this.scale = newScale;

          if (this.scale !== oldScale) this.recordHistory("Zoom View");
      },
      handleCanvasContextmenu(payload) {
          const { event, worldPos } = payload;
          this.hideContextMenu();
          if(this.fileMenu.visible) this.fileMenu.visible = false;
          if (this.isLinking || this.isDraggingLink || this.draggingConnectionEnd) return;
          
          const clickedNode = this.getNodeAtPos(worldPos.x, worldPos.y);
          if (clickedNode && this.selectedNodeId !== clickedNode.id) this.selectedNodeId = clickedNode.id;
          this.contextMenu = { visible: true, x: event.clientX, y: event.clientY, nodeId: clickedNode ? clickedNode.id : null, rawX: worldPos.x, rawY: worldPos.y };
          this.$nextTick(() => { this.adjustContextMenuPosition(); });
      },
      updateLastMousePos(pos) { // If MindMapCanvas needs to inform App.vue of its internal lastMousePos
        this.lastMousePos = pos; // {clientX, clientY}
      },
      // zoom method from App.vue is now partially integrated into handleCanvasWheel
      // resetView method remains largely the same, but uses requestCanvasRedraw
      resetView() { 
        this.hideContextMenu(); 
        const oldView = {ox: this.offsetX, oy: this.offsetY, sc: this.scale}; 
        if (this.nodes.length > 0) { 
            const firstNode = this.nodes.find(n => n.parentId === null) || this.nodes[0]; 
            if (firstNode) { 
                const canvasEl = this.$refs.mindMapCanvasComponentRef?.$refs.mindMapCanvasRef;
                const canvasWidth = canvasEl ? canvasEl.width : (this.$el?.clientWidth || window.innerWidth);
                const canvasHeight = canvasEl ? canvasEl.height : (this.$el?.clientHeight || window.innerHeight);
                const newScale = 1; 
                this.scale = newScale; 
                this.offsetX = canvasWidth / 2 - (firstNode.x + firstNode.width / 2) * newScale; 
                this.offsetY = canvasHeight / 2 - (firstNode.y + firstNode.height / 2) * newScale; 
            } 
        } else { 
            this.scale = 1; this.offsetX = 0; this.offsetY = 0; 
        } 
        if(oldView.ox !== this.offsetX || oldView.oy !== this.offsetY || oldView.sc !== this.scale) {
            this.recordHistory("Reset View");
        }
        this.requestCanvasRedraw();
      },
      // adjustContextMenuPosition, hideContextMenu, handleGlobalClick, handleGlobalKeyDown remain largely the same.
      // prepareSaveData, saveMapToLocalStorage, saveMapToFile, processLoadedData, loadMapFromLocalStorage remain.
      // resetApplicationState, resetApplication remain.
      // getNodeIconPreview, getNodeShapeIcon, getConnectionTypeIndicator remain for context menus.
      // isAncestor remains.
      // calculateDefaultConnectionPoints remains.
      // getClickedConnectionEnd, getNodeWithImageIndicatorClick, getHoveredConnectionId remain for interactions.
      // These interaction helpers (getClicked..., getNodeWith..., getHovered...) will use worldPos from canvas events.

      handleGlobalClick(event) {
  // However, modern Vue 3 prefers computed properties or methods for this.
  // For now, this is kept for direct compatibility with the original script structure.
  beforeCreate() {
    this.$filters = {
      capitalize(value) {
        if (!value) return ''
        value = value.toString()
        return value.charAt(0).toUpperCase() + value.slice(1)
      }
    }
  }
};
</script>

<style>
    body { /* These styles might be better in a global CSS file or directly on the body in index.html */
        font-family: 'Inter', sans-serif;
    }
    :root { font-family: 'Inter', system-ui, Avenir, Helvetica, Arial, sans-serif; }

    #mindMapCanvas { display: block; border-radius: 0.5rem; cursor: grab; }
    #mindMapCanvas:active { cursor: grabbing; }

    /* body styles and :root font-family are global, so they remain in App.vue or could be moved to a main.css */
    body { 
        font-family: 'Inter', sans-serif;
    }
    :root { font-family: 'Inter', system-ui, Avenir, Helvetica, Arial, sans-serif; }

    #mindMapCanvas { display: block; border-radius: 0.5rem; cursor: grab; }
    #mindMapCanvas:active { cursor: grabbing; }

    /* Toolbar specific styles moved to Toolbar.vue */
    /* Styles for color picker inputs are now in FileMenu.vue and ContextMenu.vue if needed locally, or could be global */
    /* .color-picker-label and .color-picker-input might be global or duplicated if not using Tailwind extensively */

    .modal-backdrop { background-color: rgba(0, 0, 0, 0.6); }
    .modal-content { transform: scale(0.95); opacity: 0; animation: modal-appear 0.3s forwards; }
    @keyframes modal-appear { to { opacity: 1; transform: scale(1); } }

    /* Styles for #contextMenu and related classes are now in ContextMenu.vue */
    /* Styles for #fileContextMenu and related classes are now in FileMenu.vue */
    /* [v-cloak] styles are not needed here as App.vue is a component */
</style>
