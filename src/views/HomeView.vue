<template>
  <div class="container">
    <div class="dragging-hover" v-if="dragging"></div>
    <div v-for="note in notes" :key="note.id" class="sticky-note" :style="getNoteStyle(note)"
      @mousedown="startDrag(note, $event)" ref="noteRefs">
      <h2>{{ note.title }}</h2>
      <div v-html="renderMarkdown(note.content)" class="markdown-content"></div>
    </div>

  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import { marked } from 'marked'
import DOMPurify from 'dompurify'

export default {
  name: 'StickyNotesWall',
  setup() {
    const notes = ref([])
    const windowWidth = ref(window.innerWidth)
    const windowHeight = ref(window.innerHeight)
    const draggingNote = ref(null)
    const startX = ref(0)
    const startY = ref(0)
    const offsetX = ref(0)
    const offsetY = ref(0)
    const requestId = ref(null)
    const dragging = ref(false)

    const fetchNotes = async () => {
      try {
        const response = await fetch('/sticky-notes/sticky.json')
        const data = await response.json()
        notes.value = data.map(note => ({
          ...note,
          color: getRandomColor(), // Assign a color when fetching notes
          position: getRandomPosition(note),
          rotation: getRandomRotation()
        }))
      } catch (error) {
        console.error('Error fetching notes:', error)
      }
    }

    const getRandomColor = () => {
      const colors = [
        '#ffe6e6', '#e6ffe6', '#e6e6ff', '#fff2e6', '#e6fff2', '#ffe6f2',
        '#f2e6ff', '#fffde6', '#e6f2ff', '#ffe6e6', '#f2f2e6',
        '#e6f7ff',
        '#f7f2e6',
        '#f2e6f7',
        '#e6fff7',
        '#fff6e6',
        '#f2ffe6',
        '#e6f2f2',
        '#f2f7e6',
        '#f2e6e6'
      ]
      return colors[Math.floor(Math.random() * colors.length)]
    }

    const getNoteStyle = (note) => {
      let rot = dragging.value ? 0 : note.rotation
      return {
        backgroundColor: note.color, // Use stored color
        transform: `translate(${note.position.x}px, ${note.position.y}px) rotate(${rot}deg)`,
        position: 'absolute',
        cursor: 'move'
      }
    }

    const getRandomRotation = () => {
      return Math.random() * 6 - 3 // Random rotation between -3 and 3 degrees
    }

    const getRandomPosition = (note = null) => {
      let _x = Math.max(50, Math.random() * (windowWidth.value - 320))// 280 is max width of sticky note
      let _y = Math.max(50, Math.random() * (windowHeight.value - 320)) // 280 is max height of sticky note

      if (note === null || note.x == null || note.y == null) {
        const x = _x
        const y = _y
        return { x, y }
      }

      const x = Math.max(50, (note.x))
      const y = Math.max(50, (note.y))

      return { x, y }
    }

    const renderMarkdown = (content) => {
      const rawHtml = marked(content)
      return DOMPurify.sanitize(rawHtml)
    }

    const handleResize = () => {
      windowWidth.value = window.innerWidth
      windowHeight.value = window.innerHeight
      fetchNotes() // Reposition notes on resize
    }

    const startDrag = (note, event) => {
      dragging.value = true
      draggingNote.value = note
      startX.value = event.clientX
      startY.value = event.clientY
      offsetX.value = note.position.x
      offsetY.value = note.position.y
      window.addEventListener('mousemove', dragNote)
      window.addEventListener('mouseup', stopDrag)
    }

    const dragNote = (event) => {
      if (!draggingNote.value) return
      if (requestId.value) cancelAnimationFrame(requestId.value)

      requestId.value = requestAnimationFrame(() => {
        const dx = event.clientX - startX.value
        const dy = event.clientY - startY.value
        draggingNote.value.position = {
          x: offsetX.value + dx,
          y: offsetY.value + dy
        }
      })
    }

    const stopDrag = () => {
      dragging.value = false
      draggingNote.value = null
      window.removeEventListener('mousemove', dragNote)
      window.removeEventListener('mouseup', stopDrag)
      if (requestId.value) cancelAnimationFrame(requestId.value)
    }

    onMounted(() => {
      fetchNotes()
      window.addEventListener('resize', handleResize)
    })

    onBeforeUnmount(() => {
      window.removeEventListener('resize', handleResize)
      window.removeEventListener('mousemove', dragNote)
      window.removeEventListener('mouseup', stopDrag)
      if (requestId.value) cancelAnimationFrame(requestId.value)
    })

    return {
      notes,
      getNoteStyle,
      renderMarkdown,
      startDrag,
      dragging
    }
  }
}
</script>
<style scoped>
* {
  user-select: none;
}

.dragging-hover {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  background-color: rgba(0, 0, 0, .3);
  z-index: 1000;
}

.container {
  position: relative;
  width: 100vw;
  height: 100vh;
  padding: 1rem;
  overflow: hidden;
}

.sticky-note {
  min-height: 200px;
  min-width: 200px;
  max-width: 280px;
  max-height: 280px;
  position: absolute;
  padding: 0 1rem;
  overflow: hidden;
  border: 1px solid rgba(0, 0, 0, 0);
  box-sizing: border-box;
}

.sticky-note::before {
  content: '';
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  background:
    linear-gradient(45deg, transparent 49%, rgba(255, 255, 255, 0.2) 50%, transparent 51%),
    linear-gradient(-45deg, transparent 49%, rgba(255, 255, 255, 0.2) 50%, transparent 51%);
  background-size: 8px 8px;
  pointer-events: none;
}

.sticky-note::after {
  content: '';
  position: absolute;
  top: 0;
  left: 20px;
  width: 60px;
  height: 10px;
  background-color: rgba(0, 0, 0, 0.1);
  box-shadow: 0 0 3px rgba(0, 0, 0, 0.1);
}

.sticky-note:hover {
  z-index: 9999;
  border: 1px solid rgba(0, 0, 0, .5);
  box-shadow: 0 0 3px rgba(0, 0, 0, 0.1);
}

.sticky-note:focus {
  z-index: 9999;
}

.sticky-note:active {
  z-index: 9999;
}

.markdown-content :deep(p) {
  margin-bottom: 0.5em;
}

.markdown-content :deep(ul),
.markdown-content :deep(ol) {
  padding-left: 1.5em;
  margin-bottom: 0.5em;
}

.markdown-content :deep(h1),
.markdown-content :deep(h2),
.markdown-content :deep(h3) {
  margin-top: 0.5em;
  margin-bottom: 0.25em;
}

.markdown-content :deep(code) {
  background-color: rgba(0, 0, 0, 0.05);
  padding: 0.1em 0.2em;
  border-radius: 0.2em;
}

.markdown-content :deep(pre) {
  background-color: rgba(0, 0, 0, 0.05);
  padding: 0.5em;
  border-radius: 0.3em;
  overflow-x: auto;
}
</style>