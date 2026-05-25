<template>
  <div
    class="editor-with-tabs"
    :style="{ 'max-width': `calc(100vw - ${effectiveSideBarWidth}px)` }"
  >
    <tabs v-show="showTabBar" />
    <div class="container">
      <div v-if="splitView" class="split-pane-container">
        <div ref="editorPane" class="editor-pane">
          <editor
            ref="editorRef"
            :markdown="markdown"
            :cursor="cursor"
            :text-direction="textDirection"
            :platform="platform"
          />
        </div>
        <div ref="previewPane" class="preview-pane">
          <preview ref="previewRef" :markdown="markdown" />
        </div>
      </div>
      <template v-else>
        <editor
          :markdown="markdown"
          :cursor="cursor"
          :text-direction="textDirection"
          :platform="platform"
        />
        <source-code
          v-if="sourceCode"
          :markdown="markdown"
          :muya-index-cursor="muyaIndexCursor"
          :text-direction="textDirection"
        />
      </template>
    </div>
    <tab-notifications />
  </div>
</template>

<script setup lang="ts">
import { useLayoutStore } from '@/store/layout'
import { usePreferencesStore } from '@/store/preferences'
import { storeToRefs } from 'pinia'
import { ref, onMounted, onBeforeUnmount, nextTick, watch } from 'vue'
import Tabs from './tabs.vue'
import Editor from './editor.vue'
import SourceCode from './sourceCode.vue'
import Preview from './preview.vue'
import TabNotifications from './notifications.vue'

defineProps<{
  markdown: string
  // `cursor` originates as `IFileState.cursor` which is `unknown`
  // (see src/shared/types/files.ts); align here instead of forcing every
  // caller to widen.
  cursor: unknown
  muyaIndexCursor?: unknown
  sourceCode: boolean
  showTabBar: boolean
  textDirection: string
  platform: string
}>()

const { effectiveSideBarWidth } = storeToRefs(useLayoutStore())
const { splitView } = storeToRefs(usePreferencesStore())

const editorPane = ref<HTMLDivElement | null>(null)
const previewPane = ref<HTMLDivElement | null>(null)
const editorRef = ref<any>(null)
const previewRef = ref<any>(null)
let isSyncingScroll = false
let editorScrollListenerAttached = false
let previewScrollListenerAttached = false

const attachScrollListeners = () => {
  if (editorRef.value?.getScrollContainer() && !editorScrollListenerAttached) {
    editorRef.value.getScrollContainer().addEventListener('scroll', handleEditorScroll)
    editorScrollListenerAttached = true
  }
  if (previewRef.value?.getScrollContainer() && !previewScrollListenerAttached) {
    previewRef.value.getScrollContainer().addEventListener('scroll', handlePreviewScroll)
    previewScrollListenerAttached = true
  }
}

const detachScrollListeners = () => {
  if (editorRef.value?.getScrollContainer() && editorScrollListenerAttached) {
    editorRef.value.getScrollContainer().removeEventListener('scroll', handleEditorScroll)
    editorScrollListenerAttached = false
  }
  if (previewRef.value?.getScrollContainer() && previewScrollListenerAttached) {
    previewRef.value.getScrollContainer().removeEventListener('scroll', handlePreviewScroll)
    previewScrollListenerAttached = false
  }
}

const handleEditorScroll = () => {
  if (!isSyncingScroll && editorRef.value && previewRef.value) {
    isSyncingScroll = true
    const editorContainer = editorRef.value.getScrollContainer()
    const previewContainer = previewRef.value.getScrollContainer()
    
    if (!editorContainer || !previewContainer) {
      isSyncingScroll = false
      return
    }
    
    const editorScrollTop = editorContainer.scrollTop
    const editorScrollHeight = editorContainer.scrollHeight - editorContainer.clientHeight
    const scrollRatio = editorScrollHeight > 0 ? editorScrollTop / editorScrollHeight : 0
    
    const previewScrollHeight = previewContainer.scrollHeight - previewContainer.clientHeight
    previewContainer.scrollTop = scrollRatio * previewScrollHeight
    
    requestAnimationFrame(() => {
      isSyncingScroll = false
    })
  }
}

const handlePreviewScroll = () => {
  if (!isSyncingScroll && previewRef.value && editorRef.value) {
    isSyncingScroll = true
    const editorContainer = editorRef.value.getScrollContainer()
    const previewContainer = previewRef.value.getScrollContainer()
    
    if (!editorContainer || !previewContainer) {
      isSyncingScroll = false
      return
    }
    
    const previewScrollTop = previewContainer.scrollTop
    const previewScrollHeight = previewContainer.scrollHeight - previewContainer.clientHeight
    const scrollRatio = previewScrollHeight > 0 ? previewScrollTop / previewScrollHeight : 0
    
    const editorScrollHeight = editorContainer.scrollHeight - editorContainer.clientHeight
    editorContainer.scrollTop = scrollRatio * editorScrollHeight
    
    requestAnimationFrame(() => {
      isSyncingScroll = false
    })
  }
}

// Watch for when the editor is ready
watch([editorRef, previewRef], () => {
  attachScrollListeners()
}, { deep: true })

onMounted(() => {
  nextTick(() => {
    attachScrollListeners()
  })
})

onBeforeUnmount(() => {
  detachScrollListeners()
})
</script>

<style scoped>
.editor-with-tabs {
  position: relative;
  height: 100%;
  flex: 1;
  display: flex;
  flex-direction: column;

  overflow: hidden;
  background: var(--editorBgColor);
  & > .container {
    flex: 1;
    overflow: hidden;
  }
  & .split-pane-container {
    display: flex;
    height: 100%;
    width: 100%;
    overflow: hidden;
  }
  & .editor-pane {
    flex: 1;
    overflow: hidden;
    min-width: 0;
  }
  & .preview-pane {
    flex: 1;
    overflow: hidden;
    min-width: 0;
    background: var(--editorBgColor);
  }
}
</style>
