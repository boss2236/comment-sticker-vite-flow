<template>
  <article
    ref="rootRef"
    class="sticky-note-node"
    :style="noteStyle"
    @dblclick.stop="onBodyDoubleClick"
  >
    <div v-if="isEditing" class="edit-wrap nodrag nopan">
      <textarea
        ref="textareaRef"
        v-model="draftText"
        class="note-editor nodrag nopan"
        placeholder="Type note..."
        @keydown.esc.prevent="commitAndExitEdit"
      />
    </div>

    <div v-else class="note-content" @dblclick.stop="onBodyDoubleClick">
      {{ displayText }}
    </div>

    <div class="resize-handle n" @pointerdown.stop.prevent="startResize($event, 'n')" />
    <div class="resize-handle s" @pointerdown.stop.prevent="startResize($event, 's')" />
    <div class="resize-handle e" @pointerdown.stop.prevent="startResize($event, 'e')" />
    <div class="resize-handle w" @pointerdown.stop.prevent="startResize($event, 'w')" />
    <div class="resize-handle ne" @pointerdown.stop.prevent="startResize($event, 'ne')" />
    <div class="resize-handle nw" @pointerdown.stop.prevent="startResize($event, 'nw')" />
    <div class="resize-handle se" @pointerdown.stop.prevent="startResize($event, 'se')" />
    <div class="resize-handle sw" @pointerdown.stop.prevent="startResize($event, 'sw')" />

    <div class="color-dots nodrag nopan">
      <button
        v-for="color in palette"
        :key="color"
        type="button"
        class="color-dot"
        :class="{ active: currentColor.toLowerCase() === color.toLowerCase() }"
        :style="{ background: color }"
        @click.stop="setColor(color)"
      />
    </div>
  </article>
</template>

<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import { useVueFlow } from '@vue-flow/core'

const props = defineProps({
  id: {
    type: String,
    required: true,
  },
  data: {
    type: Object,
    required: true,
  },
  selected: {
    type: Boolean,
    default: false,
  },
})

const { updateNodeData, removeNodes, findNode } = useVueFlow()

const rootRef = ref(null)
const textareaRef = ref(null)
const isEditing = ref(false)
const draftText = ref('')

const minWidth = 180
const minHeight = 120

const palette = ['#FAEEDA', '#E1F5EE', '#E6F1FB', '#FBEAF0']

const currentColor = computed(() => String(props.data?.color || '#FAEEDA'))

const displayText = computed(() => {
  const text = String(props.data?.text || '').trim()
  return text || 'Double-click to write a note...'
})

const noteStyle = computed(() => ({
  width: `${Math.max(minWidth, Number(props.data?.width) || 240)}px`,
  height: `${Math.max(minHeight, Number(props.data?.height) || 160)}px`,
  background: currentColor.value,
  borderColor: props.selected ? '#c99a2d' : '#d6c696',
}))

function onBodyDoubleClick() {
  isEditing.value = true
  draftText.value = String(props.data?.text || '')
  nextTick(() => {
    textareaRef.value?.focus()
  })
}

function commitAndExitEdit() {
  updateNodeData(props.id, { text: String(draftText.value || '') })
  isEditing.value = false
}

function handleGlobalPointerDown(event) {
  if (!isEditing.value) return
  const root = rootRef.value
  if (!root) return
  if (root.contains(event.target)) return
  commitAndExitEdit()
}

function handleKeydown(event) {
  if (event.key !== 'Backspace') return
  if (!props.selected) return
  if (isEditing.value) return

  const target = event.target
  if (target instanceof HTMLElement) {
    const tagName = String(target.tagName || '').toLowerCase()
    if (tagName === 'input' || tagName === 'textarea' || target.isContentEditable) return
  }

  event.preventDefault()
  removeNodes([props.id])
}

function setColor(color) {
  updateNodeData(props.id, { color })
}

function startResize(event, direction) {
  const node = findNode(props.id)
  if (!node) return

  const startX = event.clientX
  const startY = event.clientY
  const originalWidth = Math.max(minWidth, Number(node.data?.width) || 240)
  const originalHeight = Math.max(minHeight, Number(node.data?.height) || 160)
  const originalPosX = Number(node.position?.x) || 0
  const originalPosY = Number(node.position?.y) || 0

  function onMove(moveEvent) {
    const dx = moveEvent.clientX - startX
    const dy = moveEvent.clientY - startY

    let nextX = originalPosX
    let nextY = originalPosY
    let nextWidth = originalWidth
    let nextHeight = originalHeight

    if (direction.includes('e')) nextWidth = originalWidth + dx
    if (direction.includes('s')) nextHeight = originalHeight + dy

    if (direction.includes('w')) {
      nextWidth = originalWidth - dx
      nextX = originalPosX + dx
    }

    if (direction.includes('n')) {
      nextHeight = originalHeight - dy
      nextY = originalPosY + dy
    }

    if (nextWidth < minWidth) {
      if (direction.includes('w')) {
        nextX = originalPosX + (originalWidth - minWidth)
      }
      nextWidth = minWidth
    }

    if (nextHeight < minHeight) {
      if (direction.includes('n')) {
        nextY = originalPosY + (originalHeight - minHeight)
      }
      nextHeight = minHeight
    }

    node.position = {
      x: Math.round(nextX),
      y: Math.round(nextY),
    }

    updateNodeData(props.id, {
      width: Math.round(nextWidth),
      height: Math.round(nextHeight),
    })
  }

  function onUp() {
    window.removeEventListener('pointermove', onMove)
    window.removeEventListener('pointerup', onUp)
  }

  window.addEventListener('pointermove', onMove)
  window.addEventListener('pointerup', onUp, { once: true })
}

onMounted(() => {
  window.addEventListener('pointerdown', handleGlobalPointerDown)
  window.addEventListener('keydown', handleKeydown)
})

onBeforeUnmount(() => {
  window.removeEventListener('pointerdown', handleGlobalPointerDown)
  window.removeEventListener('keydown', handleKeydown)
})
</script>

<style scoped>
.sticky-note-node {
  border: 1px solid #d6c696;
  border-radius: 12px;
  box-shadow: 0 10px 22px -16px rgba(15, 23, 42, 0.35);
  position: relative;
  overflow: visible;
  display: grid;
  grid-template-rows: 1fr auto;
  padding: 0.6rem 0.6rem 0.45rem;
  user-select: none;
}

.note-content {
  color: #4b3a10;
  font-size: 0.78rem;
  line-height: 1.35;
  white-space: pre-wrap;
  overflow: auto;
}

.edit-wrap {
  width: 100%;
  height: 100%;
}

.note-editor {
  width: 100%;
  height: 100%;
  border: 0;
  outline: 0;
  resize: none;
  background: transparent;
  color: #4b3a10;
  font-size: 0.78rem;
  line-height: 1.35;
}

.color-dots {
  display: flex;
  align-items: center;
  gap: 0.28rem;
  padding-top: 0.35rem;
}

.color-dot {
  width: 14px;
  height: 14px;
  border-radius: 999px;
  border: 1px solid rgba(71, 85, 105, 0.4);
  cursor: pointer;
  padding: 0;
}

.color-dot.active {
  border: 2px solid rgba(71, 85, 105, 0.85);
}

.resize-handle {
  position: absolute;
  background: transparent;
}

.resize-handle.n,
.resize-handle.s {
  left: 14px;
  right: 14px;
  height: 8px;
}

.resize-handle.n {
  top: -4px;
  cursor: ns-resize;
}

.resize-handle.s {
  bottom: -4px;
  cursor: ns-resize;
}

.resize-handle.e,
.resize-handle.w {
  top: 14px;
  bottom: 14px;
  width: 8px;
}

.resize-handle.e {
  right: -4px;
  cursor: ew-resize;
}

.resize-handle.w {
  left: -4px;
  cursor: ew-resize;
}

.resize-handle.ne,
.resize-handle.nw,
.resize-handle.se,
.resize-handle.sw {
  width: 12px;
  height: 12px;
}

.resize-handle.ne {
  top: -6px;
  right: -6px;
  cursor: nesw-resize;
}

.resize-handle.nw {
  top: -6px;
  left: -6px;
  cursor: nwse-resize;
}

.resize-handle.se {
  right: -6px;
  bottom: -6px;
  cursor: nwse-resize;
}

.resize-handle.sw {
  left: -6px;
  bottom: -6px;
  cursor: nesw-resize;
}
</style>
