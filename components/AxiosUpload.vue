<script setup>
import axios from 'axios';
import { reactive } from 'vue';

const state = reactive({
    files: [],
    uploading: false,
    isDragOver: false,
})

// imput file
const handleFileChange = (event) => {
  state.uploading = true
  state.files = Array.from(event.target.files).map((file) => ({
    file,
    progress: 0,
    cancelTokenSource: axios.CancelToken.source(),
  }))
}

// drag and drop
const handleIsDrag = () => {
    state.isDragOver = true
}

const handleDrop = (event) => {
    state.isDragOver = false
    state.uploading = true
    const files = Array.from(event.dataTransfer.files).map((file) => ({
    file,
    progress: 0,
    cancelTokenSource: axios.CancelToken.source(),
    }));

    state.files = files
}

// cancel upload
const cancelUpload = (fileObj) => {
  const fileIndex = state.files.findIndex((file) => file === fileObj);

  if (fileIndex !== -1) {
    state.files[fileIndex].cancelTokenSource.cancel('cancel upload')
  }
}

// upload
const uploadFile = async (fileObj) => {
    const formData = new FormData()
    formData.append(fileObj.file.name, fileObj.file)
    for(let pair of formData.entries()) {
      console.log(pair[0]+', '+pair[1]);
    }
    let response

    try {
      console.time('axios:')
      response = await axios.post('https://httpbin.org/post', formData, {
          onUploadProgress: (progressEvent) => {
            const loaded = progressEvent.loaded
            const total = progressEvent.total
            const uploadPercentage = response ? 100 : Math.round((loaded * 95) / total)
            fileObj.progress = uploadPercentage
          },
          cancelToken: fileObj.cancelTokenSource.token
      })
      console.timeEnd('axios:')
      fileObj.progress = 100
    } catch (error) {
    console.error('Upload error', error);
    }
}

const uploadFiles = async () => {
    if (state.files.length === 0) {
      alert('No selected file!')
      return
    }

    const maxConcurrentUploads = 5
    let currentIndex = 0
    let completedUploads = 0

    const uploadFileAndNext = async (file) => {
      try {
        await uploadFile(file);
      } catch (error) {
        console.error(error);
        // 上傳失敗時的處理邏輯
      } finally {
        // 無論上傳成功或失敗，都增加已完成的上傳數
        completedUploads++;

        // 當上傳結束後，遞補新的檔案進行 POST
        if (currentIndex < state.files.length) {
          const nextFile = state.files[currentIndex++];
          uploadFileAndNext(nextFile)
        }

        // 如果所有檔案都已上傳完成，顯示提示
        if (completedUploads === state.files.length) {
          alert('All axios done')
          state.uploading = false
          state.files = []
        }
      }
    }

    // 初始化同時運行的 POST
    const uploadPromises = []
    for (let i = 0; i < maxConcurrentUploads && i < state.files.length; i++) {
      const file = state.files[currentIndex++]
      uploadPromises.push(uploadFileAndNext(file))
    }

    // 等待所有 POST 完成
    await Promise.all(uploadPromises)
}
</script>

<template>
    <div class="container">
      <h3>Here use axios</h3>
      <div
        @dragenter.prevent="handleIsDrag"
        @dragover.prevent="handleIsDrag"
        @dragleave.prevent="handleIsDrag"
        @drop.prevent="handleDrop"
        class="drop-area"
        :class="{ 'drag-over': state.isDragOver }"
        >
        <label class="input-label">
          <input type="file" @change="handleFileChange" multiple/>
          <span>+</span>
        </label>
      </div>
      <button @click="uploadFiles">Upload</button>
      <div v-if="state.uploading">
        <div v-for="(fileObj, index) in state.files" :key="index" class="progress">
          <p>{{ fileObj.file.name }}</p>
          <v-progress-linear
            rounded
            color="primary" 
            v-model="fileObj.progress"
            height="25px"
            class="linebar">
            <strong>{{ Math.ceil(fileObj.progress) }}%</strong>
          </v-progress-linear>
          <button @click="() => cancelUpload(fileObj)" class="cancel">
            Cancel
          </button>
        </div>
      </div>
    </div>
</template>

<style scoped>
.container{
  display: flex;
  gap: 1rem;
  align-items: center;
  flex-direction: column;
  border-radius: 25px;
  height: 80vh;
  width: 100%;
  padding: 1rem;
}

.drop-area {
  width: 15vw;
  height: 20vh;
  border: 2px dashed #ccc;
  border-radius: 25px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.drag-over {
  border-color: #2196F3;
  background-color: #E3F2FD;
}

.input-label{
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: rgba(225, 225, 225, 0.6);
  width: 100%;
  height: 100%;
  border-radius: 25px;
  cursor: pointer;

  input{
    display: none;
  }
  span{
    font-size: 3rem;
    font-weight: 500;
  }
}

button{
  padding: 0.5rem;
  border-radius: 25px;
  background-color: lightblue;
  font-size: 1rem;
}

.progress{
  width: 45vw;
  margin-top: 1rem;
  display: grid;
  gap: 1.5rem;
  grid-template-rows: 1fr;
  grid-template-columns: 1fr 3fr 1fr;

  p{
    grid-column: 1/2;
    align-self: center;
  }
  .linebar{
    grid-column: 2/3;
    align-self: center;
  }
}

.cancel{
  width: 60%;
  border-radius: 25px;
  padding: 0.5rem;
  background-color: pink;
  display: flex;
  align-items: center;
  justify-content: center;
  grid-column: 3/4;
  align-self: center;
  justify-self: end;
}
</style>