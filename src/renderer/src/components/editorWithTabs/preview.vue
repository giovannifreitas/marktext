<template>
  <div ref="previewContainer" class="preview-content" v-html="htmlContent" />
</template>

<script setup lang="ts">
import { ref, watch, onMounted, onBeforeUnmount } from 'vue'
import bus from '@/bus'

const props = defineProps<{
  markdown?: string
}>()

const previewContainer = ref<HTMLDivElement | null>(null)
const htmlContent = ref('')
let debounceTimer: ReturnType<typeof setTimeout> | null = null

const renderPreview = (markdown: string) => {
  if (!markdown) {
    htmlContent.value = ''
    return
  }
  
  // For now, we'll use a simple markdown-to-html conversion
  // In a full implementation, we would use Muya's exportStyledHTML
  // or integrate with the existing Muya instance
  htmlContent.value = markdown
    .replace(/^# (.*$)/gim, '<h1>$1</h1>')
    .replace(/^## (.*$)/gim, '<h2>$1</h2>')
    .replace(/^### (.*$)/gim, '<h3>$1</h3>')
    .replace(/\*\*(.*)\*\*/gim, '<strong>$1</strong>')
    .replace(/\*(.*)\*/gim, '<em>$1</em>')
    .replace(/\n/gim, '<br>')
}

const debouncedRender = (markdown: string) => {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
  debounceTimer = setTimeout(() => {
    renderPreview(markdown)
  }, 300) // 300ms debounce delay
}

watch(() => props.markdown, (newMarkdown) => {
  if (newMarkdown !== undefined) {
    debouncedRender(newMarkdown)
  }
})

const handleFileChange = (payload: unknown) => {
  const { markdown } = payload as { markdown?: string }
  if (markdown !== undefined) {
    debouncedRender(markdown)
  }
}

// Expose the scroll container for synchronized scrolling
const getScrollContainer = () => {
  return previewContainer.value
}

defineExpose({
  getScrollContainer
})

onMounted(() => {
  if (props.markdown) {
    renderPreview(props.markdown)
  }
  bus.on('file-changed', handleFileChange)
})

onBeforeUnmount(() => {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }
  bus.off('file-changed', handleFileChange)
})
</script>

<style scoped>
.preview-content {
  height: 100%;
  overflow: auto;
  padding: 50px;
  box-sizing: border-box;
  background: var(--editorBgColor);
}
.preview-content h1 {
  font-size: 2em;
  font-weight: bold;
  margin: 0.67em 0;
}
.preview-content h2 {
  font-size: 1.5em;
  font-weight: bold;
  margin: 0.83em 0;
}
.preview-content h3 {
  font-size: 1.17em;
  font-weight: bold;
  margin: 1em 0;
}
.preview-content strong {
  font-weight: bold;
}
.preview-content em {
  font-style: italic;
}
</style>
