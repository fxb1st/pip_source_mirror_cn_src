<script setup>
import { ref, onMounted, watch, computed } from 'vue'

const packageName = ref('')
const showCopyMessage = ref(false)
const usedPackages = ref([])
const isDarkMode = ref(false)
const defaultPackages = ['pandas', 'numpy', 'requests', 'pillow', 'flask']
const showSuggestions = ref(false)
const selectedIndex = ref(-1)

// 根据输入筛选建议
const suggestions = computed(() => {
  if (!packageName.value) return []
  return usedPackages.value.filter(pkg => 
    pkg.toLowerCase().includes(packageName.value.toLowerCase()) &&
    pkg !== packageName.value
  )
})

// 处理键盘事件
const handleKeydown = (e) => {
  if (!suggestions.value.length) return
  
  switch(e.key) {
    case 'ArrowDown':
      e.preventDefault()
      selectedIndex.value = (selectedIndex.value + 1) % suggestions.value.length
      break
    case 'ArrowUp':
      e.preventDefault()
      selectedIndex.value = selectedIndex.value <= 0 ? suggestions.value.length - 1 : selectedIndex.value - 1
      break
    case 'Enter':
      if (selectedIndex.value >= 0) {
        e.preventDefault()
        packageName.value = suggestions.value[selectedIndex.value]
        showSuggestions.value = false
      }
      break
    case 'Escape':
      showSuggestions.value = false
      break
  }
}

const handleInput = () => {
  selectedIndex.value = -1
  showSuggestions.value = true
}

const selectSuggestion = (suggestion) => {
  packageName.value = suggestion
  showSuggestions.value = false
}

const mirrors = [
  {
    url: 'https://mirrors.aliyun.com/pypi/simple',
    name: '阿里云镜像站'
  },
  {
    url: 'https://pypi.mirrors.ustc.edu.cn/simple',
    name: '中国科学技术大学镜像站'
  },
  {
    url: 'https://mirrors.cloud.tencent.com/pypi/simple/',
    name: '腾讯云镜像站'
  },
  {
    url: 'https://mirror.sjtu.edu.cn/pypi/web/simple/',
    name: '上海交通大学镜像站'
  }
]

// 从localStorage加载使用过的包
const loadUsedPackages = () => {
  const stored = localStorage.getItem('usedPackages')
  if (stored) {
    usedPackages.value = JSON.parse(stored)
  } else {
    // 如果没有历史记录，使用默认包
    usedPackages.value = [...defaultPackages]
    localStorage.setItem('usedPackages', JSON.stringify(usedPackages.value))
  }
}

// 保存包到localStorage
const savePackage = (pkg) => {
  if (!pkg || usedPackages.value.includes(pkg)) return
  usedPackages.value = [pkg, ...usedPackages.value]
  localStorage.setItem('usedPackages', JSON.stringify(usedPackages.value))
}

const generateCommand = (mirror) => {
  if (!packageName.value) return ''
  return `pip install -i ${mirror.url} ${packageName.value}`
}

const copyToClipboard = async (text) => {
  try {
    await navigator.clipboard.writeText(text)
    savePackage(packageName.value)
    showCopyMessage.value = true
    setTimeout(() => {
      showCopyMessage.value = false
    }, 2000)
  } catch (err) {
    console.error('Failed to copy text: ', err)
  }
}

const selectPackage = (pkg) => {
  packageName.value = pkg
}

const deletePackage = (pkg, event) => {
  event.stopPropagation()
  usedPackages.value = usedPackages.value.filter(p => p !== pkg)
  localStorage.setItem('usedPackages', JSON.stringify(usedPackages.value))
}

// 从localStorage加载主题设置
const loadThemePreference = () => {
  const stored = localStorage.getItem('darkMode')
  isDarkMode.value = stored === 'true'
  applyTheme()
}

// 切换主题
const toggleTheme = () => {
  isDarkMode.value = !isDarkMode.value
  localStorage.setItem('darkMode', isDarkMode.value)
  applyTheme()
}

// 应用主题
const applyTheme = () => {
  document.body.classList.toggle('dark-mode', isDarkMode.value)
}

onMounted(() => {
  loadUsedPackages()
  loadThemePreference()
})
</script>

<template>
  <div class="container">
    <button class="theme-toggle" @click="toggleTheme">
      {{ isDarkMode ? '🌞' : '🌙' }}
    </button>
    
    <div class="header">
      <h1 class="title">PIP 镜像源安装命令生成器</h1>
      <p class="description">
        输入包名快速生成国内镜像源的pip安装命令，支持一键复制，常用包名可保存至本地。
      </p>
    </div>

    <div class="search-bar">
      <div class="input-wrapper">
        <input 
          type="text" 
          v-model="packageName" 
          placeholder="输入pip包名称"
          class="package-input"
          @focus="$event.target.select()"
          @input="handleInput"
          @keydown="handleKeydown"
          @blur="setTimeout(() => showSuggestions = false, 200)"
        >
        <div class="suggestions" v-if="showSuggestions && suggestions.length">
          <div
            v-for="(suggestion, index) in suggestions"
            :key="suggestion"
            class="suggestion-item"
            :class="{ 'selected': index === selectedIndex }"
            @click="selectSuggestion(suggestion)"
          >
            {{ suggestion }}
          </div>
        </div>
      </div>
    </div>

    <div class="quick-packages" v-if="usedPackages.length">
      <span 
        v-for="pkg in usedPackages" 
        :key="pkg"
        class="package-tag"
        @click="selectPackage(pkg)"
      >
        {{ pkg }}
        <span 
          class="delete-tag"
          @click="deletePackage(pkg, $event)"
        >
          ×
        </span>
      </span>
    </div>

    <div class="code-blocks" v-if="packageName">
      <div v-for="mirror in mirrors" :key="mirror.url" class="code-block">
        <div class="mirror-name">{{ mirror.name }}</div>
        <div class="code-content">
          <pre>{{ generateCommand(mirror) }}</pre>
          <button 
            class="copy-btn"
            @click="copyToClipboard(generateCommand(mirror))"
          >
            复制
          </button>
        </div>
      </div>
    </div>

    <div class="copy-message" v-show="showCopyMessage">
      复制成功！
    </div>
  </div>
</template>

<style>
/* 全局样式 */
:root {
  --bg-color: #ffffff;
  --text-color: #2c3e50;
  --border-color: #ddd;
  --block-bg: #f8f8f8;
  --block-header-bg: #e8e8e8;
  --hover-bg: #e0e0e0;
  --primary-color: #42b883;
  --secondary-text: #666;
  --theme-transition: background-color 0.5s ease,
                     color 0.5s ease,
                     border-color 0.5s ease,
                     box-shadow 0.5s ease;
}

.dark-mode {
  --bg-color: #1a1a1a;
  --text-color: #e0e0e0;
  --border-color: #333;
  --block-bg: #2a2a2a;
  --block-header-bg: #333;
  --hover-bg: #3a3a3a;
  --primary-color: #42b883;
  --secondary-text: #999;
}

body {
  background-color: var(--bg-color);
  color: var(--text-color);
  transition: var(--theme-transition);
}

* {
  transition: var(--theme-transition);
}
</style>

<style scoped>
.theme-toggle {
  position: fixed;
  top: 1rem;
  right: 1rem;
  padding: 0.5rem;
  font-size: 1.5rem;
  background: none;
  border: none;
  cursor: pointer;
  opacity: 0.7;
  transition: all 0.3s ease;
  transform-origin: center;
  animation: fadeIn 0.5s ease;
}

.theme-toggle:hover {
  opacity: 1;
  transform: scale(1.1) rotate(360deg);
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: scale(0.8) rotate(-180deg);
  }
  to {
    opacity: 0.7;
    transform: scale(1) rotate(0);
  }
}

.container {
  max-width: 800px;
  margin: 2rem auto;
  padding: 0 1rem;
}

.search-bar {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
}

.package-input {
  width: 100%;
  padding: 0.5rem 1rem;
  font-size: 1rem;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  outline: none;
  background-color: var(--bg-color);
  color: var(--text-color);
}

.package-input:focus {
  border-color: var(--primary-color);
}

.code-blocks {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.code-block {
  background-color: var(--block-bg);
  border-radius: 4px;
  overflow: hidden;
}

.mirror-name {
  padding: 0.5rem 1rem;
  background-color: var(--block-header-bg);
  color: var(--secondary-text);
  font-size: 0.9rem;
  border-bottom: 1px solid var(--border-color);
}

.code-content {
  position: relative;
  padding: 1rem;
}

pre {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-all;
  font-family: monospace;
  color: var(--text-color);
}

.copy-btn {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  padding: 0.25rem 1rem;
  background-color: var(--bg-color);
  border: 1px solid var(--border-color);
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
  color: var(--text-color);
}

.copy-btn:hover {
  background-color: var(--hover-bg);
  border-color: var(--primary-color);
  color: var(--primary-color);
}

.copy-message {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: rgba(66, 184, 131, 0.9);
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  font-size: 14px;
  animation: fadeIn 0.3s ease;
  z-index: 1000;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translate(-50%, -40%);
  }
  to {
    opacity: 1;
    transform: translate(-50%, -50%);
  }
}

.quick-packages {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 2rem;
  flex-wrap: wrap;
}

.package-tag {
  position: relative;
  padding: 4px 12px;
  background-color: var(--block-bg);
  border-radius: 16px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s ease;
  user-select: none;
}

.package-tag:hover {
  background-color: var(--hover-bg);
  color: var(--primary-color);
  padding-right: 28px;
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.delete-tag {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 16px;
  font-weight: bold;
  color: #999;
  padding: 0 2px;
  opacity: 0;
  transition: all 0.3s ease;
  pointer-events: none;
}

.package-tag:hover .delete-tag {
  opacity: 1;
  pointer-events: auto;
}

.delete-tag:hover {
  color: #ff4444;
  transform: translateY(-50%) scale(1.2);
}

.header {
  text-align: center;
  margin-bottom: 2rem;
}

.title {
  color: var(--text-color);
  font-size: 1.8rem;
  margin-bottom: 1rem;
  font-weight: 600;
}

.description {
  color: var(--secondary-text);
  font-size: 1rem;
  line-height: 1.5;
  margin: 0;
}

.input-wrapper {
  position: relative;
  width: 100%;
}

.suggestions {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background-color: var(--bg-color);
  border: 1px solid var(--border-color);
  border-top: none;
  border-radius: 0 0 4px 4px;
  max-height: 200px;
  overflow-y: auto;
  z-index: 1000;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.suggestion-item {
  padding: 8px 12px;
  cursor: pointer;
  transition: all 0.2s;
}

.suggestion-item:hover,
.suggestion-item.selected {
  background-color: var(--hover-bg);
  color: var(--primary-color);
}
</style>
