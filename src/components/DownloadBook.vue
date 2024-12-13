<template>
  <div class="download-book">
    <h2>Tải Sách</h2>
    <div class="form-group">
      <label>URL cơ sở:</label>
      <input v-model="state.baseURL" placeholder="Nhập URL cơ sở...">
    </div>

    <div class="form-group">
      <label>Document ID:</label>
      <input v-model="state.docId" placeholder="Nhập Document ID...">
    </div>

    <div class="form-group">
      <label>Subfolder:</label>
      <input v-model="state.subfolder" placeholder="Nhập Subfolder...">
    </div>

    <div class="form-group">
      <label>Số trang:</label>
      <input type="number" v-model="state.totalPages" min="1">
    </div>

    <div class="form-group">
      <label>Số file tải đồng thời:</label>
      <input type="number" v-model="state.batchSize" min="1" max="10">
    </div>

    <div class="status">
      <div v-if="state.isDownloading">
        Đang tải: {{ state.currentProgress }}/{{ state.totalPages }} trang
        <div class="progress-bar">
          <div :style="{width: progressPercent + '%'}" class="progress"></div>
        </div>
      </div>
    </div>

    <button @click="startDownload" :disabled="state.isDownloading">
      {{ state.isDownloading ? 'Đang tải...' : 'Bắt đầu tải' }}
    </button>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import axios from 'axios'

interface DownloadState {
  baseURL: string
  docId: string
  subfolder: string
  totalPages: number
  batchSize: number
  isDownloading: boolean
  currentProgress: number
  failedPages: number[]
}

const state = ref<DownloadState>({
  baseURL: 'https://elib.hcmussh.edu.vn/flowpaper/services/view.php',
  docId: '',
  subfolder: '',
  totalPages: 25,
  batchSize: 5,
  isDownloading: false,
  currentProgress: 0,
  failedPages: []
})

const progressPercent = computed(() => {
  return (state.value.currentProgress / state.value.totalPages) * 100
})

const downloadPage = async (pageNumber: number): Promise<boolean> => {
  try {
    const url = `${state.value.baseURL}?doc=${state.value.docId}&format=png&page=${pageNumber}&subfolder=${state.value.subfolder}`
    
    const response = await axios.get(url, {
      responseType: 'blob',
      timeout: 10000
    })

    const blob = new Blob([response.data], { type: 'image/png' })
    const downloadUrl = window.URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = downloadUrl
    link.download = `page_${pageNumber}.png`
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
    window.URL.revokeObjectURL(downloadUrl)

    state.value.currentProgress++
    return true
  } catch (error) {
    console.error(`Lỗi tải trang ${pageNumber}:`, error)
    state.value.failedPages.push(pageNumber)
    return false
  }
}

const downloadBatch = async (pages: number[]): Promise<boolean[]> => {
  return Promise.all(pages.map(page => downloadPage(page)))
}

const startDownload = async (): Promise<void> => {
  if (!state.value.docId || !state.value.subfolder) {
    alert('Vui lòng nhập đầy đủ thông tin!')
    return
  }

  state.value.isDownloading = true
  state.value.currentProgress = 0
  state.value.failedPages = []

  const pages = Array.from({ length: state.value.totalPages }, (_, i) => i + 1)
  const batches: number[][] = []

  for (let i = 0; i < pages.length; i += state.value.batchSize) {
    batches.push(pages.slice(i, i + state.value.batchSize))
  }

  try {
    for (let i = 0; i < batches.length; i++) {
      await downloadBatch(batches[i])
      await new Promise(resolve => setTimeout(resolve, 1000))
    }

    if (state.value.failedPages.length > 0) {
      console.log(`Thử tải lại ${state.value.failedPages.length} trang lỗi...`)
      await downloadBatch(state.value.failedPages)
    }

    alert('Tải xuống hoàn tất!')
  } catch (error) {
    console.error('Lỗi:', error)
    alert('Có lỗi xảy ra trong quá trình tải!')
  } finally {
    state.value.isDownloading = false
  }
}
</script>

<style scoped>
.download-book {
  max-width: 500px;
  margin: 0 auto;
  padding: 20px;
}

.form-group {
  margin-bottom: 15px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
}

.form-group input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

button {
  width: 100%;
  padding: 10px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:disabled {
  background-color: #cccccc;
}

.progress-bar {
  width: 100%;
  height: 20px;
  background-color: #f0f0f0;
  border-radius: 10px;
  overflow: hidden;
  margin-top: 10px;
}

.progress {
  height: 100%;
  background-color: #4CAF50;
  transition: width 0.3s ease;
}

.status {
  margin: 15px 0;
  text-align: center;
}
</style> 