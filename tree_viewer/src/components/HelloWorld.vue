<script setup>
import { ref } from 'vue'
import TreeView from './TreeView.vue'

const fileContent = ref(null)
const isDragging = ref(false)
const isLoading = ref(false)
const error = ref(null)

const handleDragOver = (e) => {
  e.preventDefault()
  isDragging.value = true
}

const handleDragLeave = (e) => {
  e.preventDefault()
  isDragging.value = false
}

const handleDrop = async (e) => {
  e.preventDefault()
  isDragging.value = false
  const file = e.dataTransfer.files[0]
  await processFile(file)
}

const handleFileInput = async (e) => {
  const file = e.target.files[0]
  await processFile(file)
}

const processFile = async (file) => {
  if (!file) return
  isLoading.value = true
  error.value = null

  try {
    const text = await file.text()
    let data = null

    try {
      // 尝试解析为JSON
      data = JSON.parse(text)

      // 检查是否为树形结构
      if (isTreeStructure(data)) {
        // 已经是树形结构，直接使用
        console.log('检测到树形结构数据')
      } else {
        // 非树形结构的JSON，标记为原始JSON
        console.log('检测到非树形结构的JSON数据')
        data = {
          isRawJson: true,
          content: data
        }
      }
    } catch (e) {
      // JSON解析失败，尝试解析为文本格式
      console.log('JSON解析失败，尝试解析为文本')
      data = {
        isRawText: true,
        content: text
      }
    }

    fileContent.value = data
    console.log('解析后的数据:', data) // 添加调试日志
  } catch (e) {
    console.error('文件解析错误:', e) // 添加错误日志
    error.value = e.message || '文件解析失败，请确保上传正确的JSON文件或文本文件'
  } finally {
    isLoading.value = false
  }
}

const isTreeStructure = (data) => {
  // 检查是否为标准树形结构
  if (!data || typeof data !== 'object') return false
  
  // 检查是否有text和items字段的结构
  if (data.text && Array.isArray(data.items)) return true
  
  // 检查是否有name和children字段的结构
  if (data.name && Array.isArray(data.children)) {
    // 转换为标准格式
    data.text = data.name
    data.items = data.children.map(child => {
      const newChild = {...child}
      if (newChild.name) newChild.text = newChild.name
      if (newChild.children) newChild.items = newChild.children
      return newChild
    })
    delete data.name
    delete data.children
    return true
  }
  
  return false
}

const parseTextToTree = (text) => {
  // 简单的文本解析逻辑，可以根据需要扩展
  const lines = text.split('\n').filter(line => line.trim())
  return {
    text: '根目录',
    items: lines.map(line => ({ text: line.trim() }))
  }
}
</script>

<template>
  <div class="welcome-container">
    <h1>树形目录可视化工具</h1>
    <div class="upload-area" :class="{ dragging: isDragging }" @dragover="handleDragOver" @dragleave="handleDragLeave"
      @drop="handleDrop">
      <div class="upload-content">
        <div class="upload-icon">📁</div>
        <p>拖拽文件到这里，或者</p>
        <label class="upload-button">
          点击上传
          <input type="file" accept=".json,.txt" @change="handleFileInput" style="display: none">
        </label>
        <p class="upload-hint">支持 JSON 文件或文本文件</p>
      </div>
    </div>

    <div v-if="isLoading" class="status-message">
      <span>正在解析文件...</span>
    </div>

    <div v-if="error" class="error-message">
      {{ error }}
    </div>

    <div v-if="fileContent" class="tree-container">
      <TreeView :treeData="fileContent" />
    </div>
  </div>
</template>

<style scoped>
.welcome-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.upload-area {
  border: 2px dashed #ccc;
  border-radius: 8px;
  padding: 40px;
  text-align: center;
  transition: all 0.3s ease;
  margin: 20px 0;
}

.upload-area.dragging {
  border-color: #42b883;
  background-color: rgba(66, 184, 131, 0.1);
}

.upload-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}

.upload-icon {
  font-size: 48px;
  margin-bottom: 10px;
}

.upload-button {
  background-color: #42b883;
  color: white;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.upload-button:hover {
  background-color: #3aa876;
}

.upload-hint {
  color: #666;
  font-size: 0.9em;
}

.status-message {
  text-align: center;
  color: #42b883;
  margin: 20px 0;
}

.error-message {
  text-align: center;
  color: #ff4444;
  margin: 20px 0;
}

.tree-container {
  margin-top: 30px;
  padding: 20px;
  border: 1px solid #eee;
  border-radius: 8px;
}
</style>
