<template>
  <div class="recent-files">
    <div v-if="recentFiles.length === 0" class="empty-state">
      <div class="centered-group">
        {{ t('recent.noRecentFiles') }}
      </div>
    </div>
    <div v-else class="file-list">
      <div
        v-for="(file, index) in recentFiles"
        :key="index"
        class="file-item"
        @click="openFile(file)"
      >
        <div class="file-icon">
          <component :is="getFileIcon(file)" />
        </div>
        <div class="file-info">
          <div class="file-name">{{ getFileName(file) }}</div>
          <div class="file-path">{{ file }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { useEditorStore } from '@/store/editor'
import { t } from '../../i18n'
import {
  Document as FileIcon,
  Folder as FolderIcon
} from '@element-plus/icons-vue'
import path from 'path'

const editorStore = useEditorStore()
const recentFiles = ref<string[]>([])

const getFileName = (filePath: string): string => {
  return path.basename(filePath)
}

const getFileIcon = (filePath: string) => {
  return FileIcon
}

const openFile = (filePath: string) => {
  window.electron.ipcRenderer.send('mt::open-file', filePath)
}

const loadRecentFiles = async() => {
  try {
    const files = await window.electron.ipcRenderer.invoke('mt::menu::get-recently-used-documents')
    recentFiles.value = files
  } catch (error) {
    console.error('Failed to load recent files:', error)
  }
}

onMounted(() => {
  loadRecentFiles()
})
</script>

<style scoped>
.recent-files {
  height: 100%;
  overflow-y: auto;
  padding: 10px;
}

.empty-state {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.centered-group {
  color: var(--editorColor);
  text-align: center;
}

.file-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.file-item {
  display: flex;
  align-items: center;
  padding: 8px 12px;
  cursor: pointer;
  border-radius: 4px;
  transition: background-color 0.2s;
}

.file-item:hover {
  background-color: var(--itemHoverBgColor);
}

.file-icon {
  margin-right: 10px;
  color: var(--iconColor);
}

.file-info {
  flex: 1;
  min-width: 0;
}

.file-name {
  font-size: 14px;
  color: var(--editorColor);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.file-path {
  font-size: 12px;
  color: var(--editorColor);
  opacity: 0.6;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
</style>
