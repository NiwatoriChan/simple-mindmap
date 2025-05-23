<template>
  <div class="flex-grow m-2 relative overflow-hidden rounded-lg shadow-inner">
    <canvas ref="mindMapCanvasRef"
      id="mindMapCanvas"
      class="w-full h-full"
      :style="{ backgroundColor: props.canvasBgColor }"
      :class="{'linking-cursor': props.isLinking, 'drag-linking-cursor': props.isDraggingLink}"
      @mousedown="onMouseDown"
      @mousemove="onMouseMove"
      @mouseup="onMouseUp"
      @mouseout="onMouseOut"
      @dblclick="onDoubleClick"
      @wheel.prevent="onWheel"
      @contextmenu.prevent="onContextMenu">
    </canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch, defineProps, defineEmits } from 'vue';

// Props will be defined here
// Emits will be defined here
// Constants will be moved here
// Drawing helper functions will be moved here
// Canvas logic (redrawCanvas, event handlers, etc.) will be moved here

const mindMapCanvasRef = ref(null);
const props = defineProps({
  nodes: Array,
  selectedNodeId: String,
  offsetX: Number,
  offsetY: Number,
  scale: Number,
  isLinking: Boolean,
  linkStartNodeId: String,
  isDraggingLink: Boolean,
  linkDragStartNodeId: String,
  linkDragEndPos: Object,
  potentialLinkTargetId: String,
  potentialDropTargetId: String,
  hoveredNodeId: String,
  hoveredConnectionId: String,
  draggingConnectionEnd: Object,
  potentialEndPoint: Object,
  canvasBgColor: String,
  previewConnectionSide: Object,
});

const emit = defineEmits([
  'canvas-mousedown', 'canvas-mousemove', 'canvas-mouseup', 'canvas-mouseout',
  'canvas-dblclick', 'canvas-wheel', 'canvas-contextmenu',
  'update-last-mouse-pos' // For synchronizing lastMousePos if needed by App.vue for other components
]);

// Placeholder for event handlers - these will call emit()
const onMouseDown = (event) => { emit('canvas-mousedown', event); };
const onMouseMove = (event) => { emit('canvas-mousemove', event); };
const onMouseUp = (event) => { emit('canvas-mouseup', event); };
const onMouseOut = (event) => { emit('canvas-mouseout', event); };
const onDoubleClick = (event) => { emit('canvas-dblclick', event); };
const onWheel = (event) => { emit('canvas-wheel', event); };
const onContextMenu = (event) => { emit('canvas-contextmenu', event); };


// Watchers for props that should trigger redraw
watch(() => props.nodes, () => redrawCanvas(), { deep: true });
watch(() => props.offsetX, () => redrawCanvas());
watch(() => props.offsetY, () => redrawCanvas());
watch(() => props.scale, () => redrawCanvas());
watch(() => props.selectedNodeId, () => redrawCanvas());
watch(() => props.potentialDropTargetId, () => redrawCanvas());
watch(() => props.potentialLinkTargetId, () => redrawCanvas());
watch(() => props.hoveredNodeId, () => redrawCanvas());
watch(() => props.hoveredConnectionId, () => redrawCanvas());
watch(() => props.draggingConnectionEnd, () => redrawCanvas());
watch(() => props.isDraggingLink, () => redrawCanvas());
watch(() => props.isLinking, () => redrawCanvas());
watch(() => props.previewConnectionSide, () => redrawCanvas());
watch(() => props.canvasBgColor, () => redrawCanvas());


// --- CONSTANTS (To be moved from App.vue) ---
const NODE_DEFAULT_WIDTH = 120; const NODE_DEFAULT_HEIGHT = 40;
const NODE_DEFAULT_BG_COLOR = '#ffffff'; const NODE_DEFAULT_TEXT_COLOR = '#1f2937'; // Will be overridden by prop if provided
const NODE_SELECTED_BORDER_COLOR = '#3b82f6'; const NODE_DROP_TARGET_BORDER_COLOR = '#10b981';
const NODE_LINK_TARGET_BORDER_COLOR = '#f59e0b'; const CONNECTION_HOVER_COLOR = '#3b82f6';
const CONNECTION_PREVIEW_COLOR = '#facc15';
const IMAGE_INDICATOR_SIZE = 15; const IMAGE_INDICATOR_COLOR = '#10b981';
const CONNECTION_TYPES = ['solid', 'dashed', 'arrow']; const CONNECTION_POINTS = ['top', 'bottom', 'left', 'right'];
const REPARENT_PROXIMITY_THRESHOLD = 60; const CONNECTION_END_HOVER_RADIUS = 8;
const NODE_SHAPES = ['rectangle', 'ellipse', 'diamond', 'triangle']; const NODE_DEFAULT_SHAPE = 'rectangle';
const NODE_APPEAR_SPEED = 0.07; const NODE_HOVER_SCALE = 1.05;
// const MAX_HISTORY_STATES = 50; // Not needed in canvas
const LINK_ANIMATION_DURATION = 200;
const TEMP_LINK_COLOR = '#818cf8';
// const AUTOSAVE_INTERVAL = 50000; // Not needed in canvas
// const DEFAULT_CANVAS_BG_COLOR = '#2d3748'; // canvasBgColor is a prop
const CHILD_NODE_SPACING = 40;


const AVAILABLE_ICONS = [
    { name: 'Ampoule', class: 'fas fa-lightbulb', unicode: '\uf0eb' }, { name: 'Étoile', class: 'fas fa-star', unicode: '\uf005' },
    { name: 'Coche', class: 'fas fa-check', unicode: '\uf00c' }, { name: 'Question', class: 'fas fa-question-circle', unicode: '\uf059' },
    { name: 'Attention', class: 'fas fa-exclamation-triangle', unicode: '\uf071' }, { name: 'Info', class: 'fas fa-info-circle', unicode: '\uf05a' },
    { name: 'Lien', class: 'fas fa-link', unicode: '\uf0c1' }, { name: 'Dossier', class: 'fas fa-folder', unicode: '\uf07b' },
    { name: 'Personne', class: 'fas fa-user', unicode: '\uf007' },
];
const HYPERLINK_INDICATOR_UNICODE = '\uf0c1';
const ICON_SIZE = 16;

const AVAILABLE_BORDER_STYLES = [
    { name: 'Solide', value: 'solid', dash: [] }, { name: 'Tiretée', value: 'dashed', dash: [7, 4] },
    { name: 'Pointillée', value: 'dotted', dash: [2, 3] },
];

// --- DRAWING HELPER FUNCTIONS (To be moved from App.vue) ---
const getDynamicDrawingStyles = () => ({ // This could take theme props in the future
    connectionColor: '#718096',
    nodeDefaultBorderColor: '#A0AEC0',
    shadowColor: 'rgba(0,0,0,0.4)',
    // contextMenuBg: 'bg-slate-800', // Not used by canvas directly
    // contextMenuItemText: 'text-gray-200',
    // contextMenuItemHoverBg: 'hover:bg-slate-700',
    // contextMenuBorder: 'border-slate-700',
});

const fillRoundRect = (ctx, x, y, w, h, r) => { if (w < 2 * r) r = w / 2; if (h < 2 * r) r = h / 2; ctx.beginPath(); ctx.moveTo(x + r, y); ctx.arcTo(x + w, y, x + w, y + h, r); ctx.arcTo(x + w, y + h, x, y + h, r); ctx.arcTo(x, y + h, x, y, r); ctx.arcTo(x, y, x + w, y, r); ctx.closePath(); ctx.fill(); };
const strokeRoundRect = (ctx, x, y, w, h, r) => { if (w < 2 * r) r = w / 2; if (h < 2 * r) r = h / 2; ctx.beginPath(); ctx.moveTo(x + r, y); ctx.arcTo(x + w, y, x + w, y + h, r); ctx.arcTo(x + w, y + h, x, y + h, r); ctx.arcTo(x, y + h, x, y, r); ctx.arcTo(x, y, x + w, y, r); ctx.closePath(); ctx.stroke(); };
const wrapText = (context, text, x, y, maxWidth, lineHeight, hasIcon, hasLink) => {
    let iconOffsetLeft = 0; let textDrawX = x; const iconPadding = 5; let leftPadding = 10;
    if (hasIcon) iconOffsetLeft += ICON_SIZE + iconPadding; if (hasLink) iconOffsetLeft += ICON_SIZE + iconPadding;
    const availableTextWidth = maxWidth - iconOffsetLeft - 10;
    textDrawX = x - maxWidth/2 + leftPadding + iconOffsetLeft + availableTextWidth / 2;
    const lines = String(text).split('\n'); let totalLineCount = 0; const processedLines = [];
    lines.forEach(lineText => { const words = lineText.split(' '); let currentLine = ''; for (let n = 0; n < words.length; n++) { let testLine = currentLine + words[n] + ' '; let metrics = context.measureText(testLine); let testWidth = metrics.width; if (testWidth > availableTextWidth && n > 0) { processedLines.push(currentLine.trim()); currentLine = words[n] + ' '; } else { currentLine = testLine; } } processedLines.push(currentLine.trim()); });
    totalLineCount = processedLines.length; let startY = y - (totalLineCount - 1) * lineHeight / 2;
    processedLines.forEach((line, index) => { context.fillText(line, textDrawX, startY + index * lineHeight); });
};
const drawArrowhead = (context, fromX, fromY, toX, toY, headLength) => { const dx = toX - fromX; const dy = toY - fromY; const angle = Math.atan2(dy, dx); context.beginPath(); context.moveTo(toX, toY); context.lineTo(toX - headLength * Math.cos(angle - Math.PI / 6), toY - headLength * Math.sin(angle - Math.PI / 6)); context.lineTo(toX - headLength * Math.cos(angle + Math.PI / 6), toY - headLength * Math.sin(angle + Math.PI / 6)); context.closePath(); context.fill(); };
const drawRectangleNode = (ctx, node) => { fillRoundRect(ctx, node.x, node.y, node.width, node.height, 10); strokeRoundRect(ctx, node.x, node.y, node.width, node.height, 10); };
const drawEllipseNode = (ctx, node) => { const cx = node.x + node.width / 2; const cy = node.y + node.height / 2; const rx = node.width / 2; const ry = node.height / 2; if (rx <= 0 || ry <= 0) return; ctx.beginPath(); ctx.ellipse(cx, cy, rx, ry, 0, 0, Math.PI * 2); ctx.fill(); ctx.stroke(); };
const drawDiamondNode = (ctx, node) => { const { x, y, width: w, height: h } = node; const p1x = x + w / 2, p1y = y; const p2x = x + w, p2y = y + h / 2; const p3x = x + w / 2, p3y = y + h; const p4x = x, p4y = y + h / 2; ctx.beginPath(); ctx.moveTo(p1x, p1y); ctx.lineTo(p2x, p2y); ctx.lineTo(p3x, p3y); ctx.lineTo(p4x, p4y); ctx.closePath(); ctx.fill(); ctx.stroke(); };
const drawTriangleNode = (ctx, node) => { const { x, y, width: w, height: h } = node; const p1x = x + w / 2, p1y = y; const p2x = x + w, p2y = y + h; const p3x = x, p3y = y + h; ctx.beginPath(); ctx.moveTo(p1x, p1y); ctx.lineTo(p2x, p2y); ctx.lineTo(p3x, p3y); ctx.closePath(); ctx.fill(); ctx.stroke(); };
const drawNodeImageIndicator = (ctx, node, currentScale) => { const r = IMAGE_INDICATOR_SIZE / 2; const xPos = node.x + node.width - r; const yPos = node.y + r; ctx.save(); ctx.beginPath(); ctx.arc(xPos, yPos, r, 0, Math.PI * 2); ctx.fillStyle = IMAGE_INDICATOR_COLOR; ctx.fill(); ctx.strokeStyle = 'white'; ctx.lineWidth = 1 / currentScale; ctx.stroke(); ctx.restore(); };
const drawConnectionEndpointCircle = (ctx, x, y, currentScale) => { ctx.save(); ctx.beginPath(); ctx.arc(x, y, CONNECTION_END_HOVER_RADIUS / currentScale, 0, Math.PI * 2); ctx.fillStyle = 'rgba(59, 130, 246, 0.5)'; ctx.fill(); ctx.restore(); };

// --- CORE CANVAS LOGIC ---
const isAnimating = ref(false); // Local animation state

const findNodeById = (id) => props.nodes.find(node => node.id === id); // Uses props.nodes

const redrawCanvas = () => {
  const canvas = mindMapCanvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx || canvas.width <= 0 || canvas.height <= 0) return;

  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.save();
  ctx.translate(props.offsetX, props.offsetY);
  ctx.scale(props.scale, props.scale);

  const ds = getDynamicDrawingStyles();
  ctx.lineWidth = 2; // Default line width

  // Draw connections
  props.nodes.forEach(child => {
    if (!child || child.parentId === null || child.parentId === undefined) return;
    const parent = findNodeById(child.parentId);
    if (parent) {
        let tempParentExitPoint = child.parentExitPoint || 'bottom';
        let tempChildEntryPoint = child.entryPoint || 'top';
        let tempConnectionType = child.connectionType || 'solid';
        let isPreviewingConnection = false;
        let isBeingDraggedEndpoint = props.draggingConnectionEnd && props.draggingConnectionEnd.childId === child.id;

        let currentParentNode = parent;
        let currentChildNode = child;
        let currentParentExitPoint = tempParentExitPoint;
        let currentChildEntryPoint = tempChildEntryPoint;

        if (props.previewConnectionSide && props.previewConnectionSide.nodeId === child.id) {
            if (props.previewConnectionSide.property === 'parentExitPoint') { tempParentExitPoint = props.previewConnectionSide.value; isPreviewingConnection = true; }
            else if (props.previewConnectionSide.property === 'entryPoint') { tempChildEntryPoint = props.previewConnectionSide.value; isPreviewingConnection = true; }
            else if (props.previewConnectionSide.property === 'connectionType') { tempConnectionType = props.previewConnectionSide.value; isPreviewingConnection = true; }
            currentParentExitPoint = tempParentExitPoint;
            currentChildEntryPoint = tempChildEntryPoint;
        }

        let finalP0, finalP3;
        if (isBeingDraggedEndpoint && props.potentialEndPoint) {
            if (props.draggingConnectionEnd.endType === 'parentExit') {
                finalP0 = props.potentialEndPoint.coords;
                finalP3 = getConnectionPointCoords(child, currentChildEntryPoint);
            } else {
                finalP0 = getConnectionPointCoords(parent, currentParentExitPoint);
                finalP3 = props.potentialEndPoint.coords;
            }
        } else {
            finalP0 = getConnectionPointCoords(currentParentNode, currentParentExitPoint);
            finalP3 = getConnectionPointCoords(currentChildNode, currentChildEntryPoint);
        }
        
        if (!finalP0 || !finalP3 || typeof finalP0.x !== 'number' || typeof finalP3.x !== 'number') return;

        let P0 = { ...finalP0 };
        let P3 = { ...finalP3 };

        if (child.isLinkAnimating && child.linkPrevP0 && child.linkPrevP3) {
            const elapsed = Date.now() - child.linkAnimationStartTime;
            let progress = Math.min(1, elapsed / LINK_ANIMATION_DURATION);
            progress = 1 - Math.pow(1 - progress, 3); // Ease-out cubic
            
            P0.x = child.linkPrevP0.x + (finalP0.x - child.linkPrevP0.x) * progress;
            P0.y = child.linkPrevP0.y + (finalP0.y - child.linkPrevP0.y) * progress;
            P3.x = child.linkPrevP3.x + (finalP3.x - child.linkPrevP3.x) * progress;
            P3.y = child.linkPrevP3.y + (finalP3.y - child.linkPrevP3.y) * progress;

            if (progress >= 1) {
                // Animation completion might need to be handled by App.vue if it modifies node state
                // For now, assuming direct modification or App.vue handles this via events
                child.isLinkAnimating = false; 
                delete child.linkPrevP0;
                delete child.linkPrevP3;
                delete child.linkAnimationStartTime;
            }
        }

        ctx.strokeStyle = isPreviewingConnection ? CONNECTION_PREVIEW_COLOR : (child.id === props.hoveredConnectionId || isBeingDraggedEndpoint ? CONNECTION_HOVER_COLOR : ds.connectionColor);
        ctx.fillStyle = ctx.strokeStyle;

        let P1 = { x: P0.x, y: P0.y }, P2 = { x: P3.x, y: P3.y };
        const fixedOffset = 50; 
        
        let effectiveParentExitSide = currentParentExitPoint;
        let effectiveChildEntrySide = currentChildEntryPoint;

        if (isBeingDraggedEndpoint && props.potentialEndPoint) {
            if (props.draggingConnectionEnd.endType === 'parentExit') {
                effectiveParentExitSide = props.potentialEndPoint.side;
            } else { 
                effectiveChildEntrySide = props.potentialEndPoint.side;
            }
        }
        switch (effectiveParentExitSide) { case 'top': P1.y -= fixedOffset; break; case 'bottom': P1.y += fixedOffset; break; case 'left': P1.x -= fixedOffset; break; case 'right': P1.x += fixedOffset; break; }
        switch (effectiveChildEntrySide) { case 'top': P2.y -= fixedOffset; break; case 'bottom': P2.y += fixedOffset; break; case 'left': P2.x -= fixedOffset; break; case 'right': P2.x += fixedOffset; break; }
        
        if (tempConnectionType === 'dashed') ctx.setLineDash([7,4]);
        else if (tempConnectionType === 'dotted') ctx.setLineDash([2,3]);
        else ctx.setLineDash([]);

        ctx.beginPath(); ctx.moveTo(P0.x, P0.y); ctx.bezierCurveTo(P1.x, P1.y, P2.x, P2.y, P3.x, P3.y); ctx.stroke();
        if (tempConnectionType === 'arrow') drawArrowhead(ctx, P2.x, P2.y, P3.x, P3.y, 10);
        ctx.setLineDash([]);

        if (child.id === props.hoveredConnectionId || isBeingDraggedEndpoint) {
            drawConnectionEndpointCircle(ctx, finalP0.x, finalP0.y, props.scale); 
            drawConnectionEndpointCircle(ctx, finalP3.x, finalP3.y, props.scale); 
        }
    }
  });

  // Draw temporary link for Alt+Drag or context menu linking
  if (props.isDraggingLink && props.linkDragStartNodeId && props.linkDragEndPos) {
      const startNode = findNodeById(props.linkDragStartNodeId);
      if (startNode) {
          // This calculation might need to be passed from App.vue or calculated via emitted events
          // For now, simplified:
          const startCoords = getConnectionPointCoords(startNode, 'bottom'); // Simplified
          ctx.save();
          ctx.beginPath();
          ctx.moveTo(startCoords.x, startCoords.y);
          ctx.lineTo(props.linkDragEndPos.x, props.linkDragEndPos.y);
          ctx.strokeStyle = TEMP_LINK_COLOR;
          ctx.lineWidth = 2 / props.scale;
          ctx.setLineDash([3,3]);
          ctx.stroke();
          ctx.restore();
      }
  }
  if (props.isLinking && props.linkStartNodeId && props.lastMousePos && props.lastMousePos.clientX !== 0) { // lastMousePos needs to be passed or managed
       const startNode = findNodeById(props.linkStartNodeId);
       const canvasEl = mindMapCanvasRef.value;
       if (startNode && canvasEl) {
          // getMousePos is now local to MindMapCanvas.vue
          // const mousePos = getMousePos(props.lastMousePos, canvasEl);
          // For simplicity, assume lastMousePos is already in world coordinates if passed as prop.
          // This part needs careful handling of coordinate systems.
          // Let's assume App.vue converts lastMousePos to world coordinates before passing.
          // OR, MindMapCanvas.vue keeps its own local lastMousePos state updated on mousemove.
          // For now, I'll assume lastMousePos is passed as {x, y} in world coords by App.vue for this drawing.
          // This might be incorrect and need adjustment based on where lastMousePos is actually managed.
          // For now, let's use linkDragEndPos as a placeholder for the cursor position during linking.
          const mousePos = props.linkDragEndPos || {x:0, y:0}; // This is a placeholder/simplification
          const tempTargetNode = { x: mousePos.x - NODE_DEFAULT_WIDTH/2, y: mousePos.y - NODE_DEFAULT_HEIGHT/2, width: NODE_DEFAULT_WIDTH, height: NODE_DEFAULT_HEIGHT};
          const { parentExitPoint } = calculateDefaultConnectionPoints(startNode, tempTargetNode); // This function also needs to be local
          const startCoords = getConnectionPointCoords(startNode, parentExitPoint);
          ctx.save();
          ctx.beginPath();
          ctx.moveTo(startCoords.x, startCoords.y);
          ctx.lineTo(mousePos.x, mousePos.y);
          ctx.strokeStyle = TEMP_LINK_COLOR;
          ctx.lineWidth = 2 / props.scale;
          ctx.setLineDash([5,5]);
          ctx.stroke();
          ctx.restore();
       }
  }

  // Draw nodes
  props.nodes.forEach(node => {
    if (!node || typeof node.x !== 'number') return;
    let tempShape = node.shape || NODE_DEFAULT_SHAPE;
    let tempBorderStyle = node.borderStyle || 'solid';
    let tempBorderWidth = node.borderWidth || 1;

    if (props.previewConnectionSide && props.previewConnectionSide.nodeId === node.id) {
        if (props.previewConnectionSide.property === 'shape') { tempShape = props.previewConnectionSide.value; }
        else if (props.previewConnectionSide.property === 'borderStyle') { tempBorderStyle = props.previewConnectionSide.value; }
        else if (props.previewConnectionSide.property === 'borderWidth') { tempBorderWidth = props.previewConnectionSide.value; }
    }

    const centerX = node.x + node.width / 2; const centerY = node.y + node.height / 2;
    ctx.save();
    let currentTransformScale = 1.0; let currentAlpha = 1.0;
    if (node.isAppearing) { 
        node.animationProgress = Math.min(1, (node.animationProgress || 0) + NODE_APPEAR_SPEED); 
        currentTransformScale = 0.5 + node.animationProgress * 0.5; currentAlpha = node.animationProgress; 
        if (node.animationProgress >= 1) { 
            // App.vue should handle removing these flags via an event if needed
            // delete node.isAppearing; delete node.animationProgress; 
        } 
        ctx.translate(centerX, centerY); ctx.scale(currentTransformScale, currentTransformScale); ctx.translate(-centerX, -centerY); ctx.globalAlpha = currentAlpha; 
    } else if (node.id === props.hoveredNodeId && !props.isDraggingLink && !props.draggingConnectionEnd) { // Added checks
        ctx.translate(centerX, centerY); ctx.scale(NODE_HOVER_SCALE, NODE_HOVER_SCALE); ctx.translate(-centerX, -centerY); 
    }

    let borderColor = ds.nodeDefaultBorderColor;
    if (node.id === props.selectedNodeId) { borderColor = NODE_SELECTED_BORDER_COLOR; tempBorderWidth = Math.max(tempBorderWidth, 2); }
    else if (node.id === props.potentialDropTargetId) { borderColor = NODE_DROP_TARGET_BORDER_COLOR; tempBorderWidth = Math.max(tempBorderWidth, 2); }
    else if (node.id === props.potentialLinkTargetId) { borderColor = NODE_LINK_TARGET_BORDER_COLOR; tempBorderWidth = Math.max(tempBorderWidth, 2); }
    else if (props.draggingConnectionEnd && props.potentialEndPoint && node.id === props.potentialEndPoint.node.id &&
             (!props.draggingConnectionEnd.nodeToAttach || node.id !== props.draggingConnectionEnd.nodeToAttach.id)) {
        borderColor = NODE_LINK_TARGET_BORDER_COLOR;
        tempBorderWidth = Math.max(tempBorderWidth, 2);
    }

    ctx.fillStyle = node.color || NODE_DEFAULT_BG_COLOR;
    ctx.strokeStyle = borderColor;
    ctx.lineWidth = tempBorderWidth / currentTransformScale;
    const borderStyleObj = AVAILABLE_BORDER_STYLES.find(bs => bs.value === tempBorderStyle);
    ctx.setLineDash(borderStyleObj ? borderStyleObj.dash.map(d => d * (ctx.lineWidth > 1 ? 1.5 : 1)) : []);
    ctx.shadowColor = ds.shadowColor; ctx.shadowBlur = 5 / currentTransformScale; ctx.shadowOffsetX = 2 / currentTransformScale; ctx.shadowOffsetY = 2 / currentTransformScale;

    if (tempShape === 'rectangle') drawRectangleNode(ctx, node);
    else if (tempShape === 'ellipse') drawEllipseNode(ctx, node);
    else if (tempShape === 'diamond') drawDiamondNode(ctx, node);
    else if (tempShape === 'triangle') drawTriangleNode(ctx, node);
    else drawRectangleNode(ctx, node);

    ctx.setLineDash([]); ctx.shadowColor = 'transparent';
    ctx.fillStyle = node.textColor || NODE_DEFAULT_TEXT_COLOR;
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
    let currentIconX = node.x + 10;
    const iconY = node.y + node.height / 2;
    if (node.icon) { const faIcon = AVAILABLE_ICONS.find(i => i.class === node.icon); if (faIcon) { ctx.font = `900 ${ICON_SIZE}px "Font Awesome 6 Free"`; ctx.fillText(faIcon.unicode, currentIconX, iconY); currentIconX += ICON_SIZE + 5; } }
    if (node.hyperlink) { ctx.font = `900 ${ICON_SIZE}px "Font Awesome 6 Free"`; const oldFill = ctx.fillStyle; ctx.fillStyle = '#60a5fa'; ctx.fillText(HYPERLINK_INDICATOR_UNICODE, currentIconX, iconY); ctx.fillStyle = oldFill; }
    ctx.font = '14px Inter, sans-serif';
    ctx.fillStyle = node.textColor || NODE_DEFAULT_TEXT_COLOR;
    let textMaxWidth = node.width - 15 - ( (node.icon ? ICON_SIZE + 5 : 0) + (node.hyperlink ? ICON_SIZE + 5 : 0) );
    let textDrawX = node.x + node.width / 2;
    if(node.icon || node.hyperlink) {
       textDrawX = node.x + (node.icon ? ICON_SIZE + 5 : 0) + (node.hyperlink ? ICON_SIZE + 5 : 0) + textMaxWidth / 2 + 5;
    }
    let textYOffset = node.height * 0.5;
    if (tempShape === 'ellipse') textMaxWidth *= 0.8;
    else if (tempShape === 'diamond') textMaxWidth *= 0.7;
    else if (tempShape === 'triangle') { textMaxWidth *= 0.6; textYOffset = node.height * 0.65; }
    wrapText(ctx, node.text || "", textDrawX, node.y + textYOffset, textMaxWidth, 18, !!node.icon, !!node.hyperlink);
    if (node.imageUrl && node.imageObj) drawNodeImageIndicator(ctx, node, props.scale); // Use props.scale
    ctx.restore();
  });

  ctx.restore();

  let stillAnimatingThisFrame = false; 
  props.nodes.forEach(n => { if(n.isAppearing || n.isLinkAnimating) stillAnimatingThisFrame = true; });
  if (stillAnimatingThisFrame && !isAnimating.value) startAnimationLoop();
  else if (!stillAnimatingThisFrame && isAnimating.value) isAnimating.value = false;
};


const resizeCanvas = () => {
  const canvas = mindMapCanvasRef.value;
  if (!canvas || !canvas.parentElement) return;
  const parent = canvas.parentElement;
  const newWidth = Math.max(1, parent.clientWidth);
  const newHeight = Math.max(1, parent.clientHeight);
  if (canvas.width !== newWidth || canvas.height !== newHeight) {
    canvas.width = newWidth;
    canvas.height = newHeight;
    redrawCanvas();
  }
};

const startAnimationLoop = () => {
    if (isAnimating.value) return;
    isAnimating.value = true;
    const animateStep = () => {
        let stillAnimatingThisFrame = false;
        props.nodes.forEach(n => { // Check props.nodes for animation flags
            if (n.isAppearing || n.isLinkAnimating) {
                stillAnimatingThisFrame = true;
            }
        });

        if (stillAnimatingThisFrame) {
            redrawCanvas();
            requestAnimationFrame(animateStep);
        } else {
            isAnimating.value = false;
            redrawCanvas(); // Final redraw after animation stops
        }
    };
    requestAnimationFrame(animateStep);
};

// Helper: calculateDefaultConnectionPoints (moved from App.vue)
const calculateDefaultConnectionPoints = (parentNode, childNode) => {
    const dx = (childNode.x + childNode.width / 2) - (parentNode.x + parentNode.width / 2);
    const dy = (childNode.y + childNode.height / 2) - (parentNode.y + parentNode.height / 2);
    if (Math.abs(dx) > Math.abs(dy)) {
        return dx > 0 ? { parentExitPoint: 'right', entryPoint: 'left' } : { parentExitPoint: 'left', entryPoint: 'right' };
    } else {
        return dy > 0 ? { parentExitPoint: 'bottom', entryPoint: 'top' } : { parentExitPoint: 'top', entryPoint: 'bottom' };
    }
};

// Helper: getConnectionPointCoords (moved from App.vue)
const getConnectionPointCoords = (node, pointName) => {
    if (!node) return { x: 0, y: 0 };
    const shape = node.shape || NODE_DEFAULT_SHAPE;
    const { x, y, width: w, height: h } = node;
    const cx = x + w / 2;
    const cy = y + h / 2;

    if (shape === 'rectangle' || shape === 'diamond') {
        switch (pointName) {
            case 'top': return { x: cx, y: y };
            case 'bottom': return { x: cx, y: y + h };
            case 'left': return { x: x, y: cy };
            case 'right': return { x: x + w, y: cy };
            default: return { x: cx, y: y }; // Default to top for 'auto' or unknown
        }
    } else if (shape === 'ellipse') {
        const rx = w / 2;
        const ry = h / 2;
        if (rx <= 0 || ry <=0) return {x: cx, y: cy};
        switch (pointName) {
            case 'top': return { x: cx, y: cy - ry };
            case 'bottom': return { x: cx, y: cy + ry };
            case 'left': return { x: cx - rx, y: cy };
            case 'right': return { x: cx + rx, y: cy };
            default: return { x: cx, y: cy - ry };
        }
    } else if (shape === 'triangle') {
         switch (pointName) {
            case 'top': return { x: cx, y: y };
            case 'bottom': return { x: cx, y: y + h }; // Broad base center
            case 'left': return { x: x + w * 0.25, y: y + h * 0.65 }; // Adjusted for typical triangle
            case 'right': return { x: x + w * 0.75, y: y + h * 0.65 };// Adjusted
            default: return { x: cx, y: y };
        }
    }
    return { x: cx, y: y }; // Fallback for unknown shapes
};


onMounted(() => {
  resizeCanvas(); // Initial resize
  window.addEventListener('resize', resizeCanvas);
  redrawCanvas(); // Initial draw
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', resizeCanvas);
});

// Expose methods to parent if needed via defineExpose (e.g., for forcing a redraw)
// defineExpose({ redrawCanvas, getMousePos }); // getMousePos might be useful for App.vue
defineExpose({ redrawCanvas });


</script>

<style scoped>
  /* Styles from App.vue specific to canvas wrapper or canvas itself */
  .flex-grow { /* This class was on the div wrapper */
    /* flex-grow: 1; Tailwind class already handles this */
  }
  .m-2 { /* Tailwind */ }
  .relative { /* Tailwind */ }
  .overflow-hidden { /* Tailwind */ }
  .rounded-lg { /* Tailwind */ }
  .shadow-inner { /* Tailwind */ }

  #mindMapCanvas { /* ID selector might be too strong if this component is reused, consider class */
    display: block; /* Was in App.vue style */
    border-radius: 0.5rem; /* Tailwind: rounded-lg */
    cursor: grab; /* Default cursor */
  }
  #mindMapCanvas:active {
    cursor: grabbing; /* For panning */
  }
  .linking-cursor { cursor: crosshair !important; } /* These are fine as they are conditional */
  .drag-linking-cursor { cursor: alias !important; }
</style>
