<template>
  <div class="map-view">
    <h1>地图路径绘制</h1>
    <div class="instructions">
      <p>
        点击地图上的位置添加路径点，点击已有的点可以选择它，点击"删除最后一个点"可以删除最后添加的点，点击"清除所有点"可以清除所有点。
      </p>
    </div>

    <div class="map-upload">
      <h2>选择地图图片</h2>
      <div class="upload-container">
        <input type="file" @change="handleFileUpload" accept="image/*" ref="fileInput" />
        <button @click="triggerFileInput">选择图片文件</button>
      </div>
      <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
    </div>

    <div class="map-container" v-if="mapImageUrl">
      <div class="map-content">
        <MapPathDrawer
          ref="mapPathDrawerRef"
          :mapImageUrl="mapImageUrl"
          :workflowStep="workflowStep"
          @update:pathGraph="pathGraph = $event"
        />
      </div>

      <div class="workflow-steps">
        <h3>路径规划工作流</h3>
        <div class="step-buttons">
          <button @click="setWorkflowStep(1)" :class="{ 'active-mode': workflowStep === 1 }">
            步骤1: 预设点位
          </button>
          <button
            @click="setWorkflowStep(2)"
            :class="{ 'active-mode': workflowStep === 2 }"
            :disabled="!pathGraph || pathGraph.nodes.length < 2"
          >
            步骤2: 预设路径连接
          </button>
          <button
            @click="showPathPlanningDialog()"
            :class="{ 'active-mode': workflowStep === 3 }"
            :disabled="
              !pathGraph || pathGraph.nodes.length < 2 || pathGraph.connections.length === 0
            "
          >
            步骤3: 计算最优路径
          </button>
        </div>

        <div class="workflow-description">
          <div class="step-description" v-if="workflowStep === 1">
            <h4>步骤1: 预设点位</h4>
            <p>在地图上点击添加节点，可以选择已有节点进行删除。</p>
          </div>
          <div class="step-description" v-if="workflowStep === 2">
            <h4>步骤2: 预设路径连接</h4>
            <p>点击两个节点创建连接，点击已有连接可以选中并删除。</p>
          </div>
          <div class="step-description" v-if="workflowStep === 3">
            <h4>步骤3: 计算最优路径</h4>
            <p>选择起点和终点，计算最短路径或多点路径。</p>
          </div>
        </div>

        <!-- 删除操作按钮区域 -->
        <div class="delete-operations">
          <h4>删除操作</h4>
          <div class="delete-buttons">
            <button
              @click="clearPoints"
              :disabled="!pathGraph || pathGraph.nodes.length === 0"
              class="delete-button"
            >
              清除所有点
            </button>
            <button
              @click="removeLastPoint"
              :disabled="!pathGraph || pathGraph.nodes.length === 0"
              class="delete-button"
            >
              删除最后一个点
            </button>
            <button
              @click="removeSelectedPoint"
              :disabled="!pathGraph || pathGraph.nodes.length === 0"
              class="delete-button"
            >
              删除选中点
            </button>
            <button
              v-if="workflowStep === 2"
              @click="removeSelectedConnection"
              :disabled="!selectedConnectionId"
              class="delete-button"
            >
              删除选中路径
            </button>
            <button
              v-if="workflowStep === 2"
              @click="clearAllConnections"
              :disabled="!pathGraph || pathGraph.connections.length === 0"
              class="delete-button delete-all"
            >
              清除所有路径
            </button>
            <button
              v-if="workflowStep === 2"
              @click="removeLastConnection"
              :disabled="!pathGraph || pathGraph.connections.length === 0"
              class="delete-button"
            >
              删除最后一条路径
            </button>
          </div>
        </div>
      </div>
    </div>

    <div v-else class="no-map">
      <p>请先上传地图图片</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import MapPathDrawer from '@/components/MapPathDrawer.vue'
import { ref, computed } from 'vue'

const mapImageUrl = ref<string>('')
const fileInput = ref<HTMLInputElement | null>(null)
const errorMessage = ref<string>('')
const workflowStep = ref<number>(1) // 1: 预设点位, 2: 预设路径连接, 3: 计算路径
interface PathGraph {
  nodes: Array<{
    id: number
    point: { x: number; y: number }
    name: string
  }>
  connections: Array<{
    from: number
    to: number
    weight: number
  }>
}

const pathGraph = ref<PathGraph | null>(null)

// 处理文件上传
const handleFileUpload = (event: Event) => {
  const input = event.target as HTMLInputElement
  if (!input.files || input.files.length === 0) {
    errorMessage.value = '未选择文件'
    return
  }

  const file = input.files[0]
  if (!file.type.startsWith('image/')) {
    errorMessage.value = '请选择图片文件'
    return
  }

  errorMessage.value = ''
  const reader = new FileReader()
  reader.onload = (e) => {
    if (e.target && typeof e.target.result === 'string') {
      mapImageUrl.value = e.target.result
    }
  }
  reader.readAsDataURL(file)
}

const triggerFileInput = () => {
  if (fileInput.value) {
    fileInput.value.click()
  }
}

// 设置默认地图（可选）
// mapImageUrl.value = '/favicon.ico'

// 设置工作流步骤
const setWorkflowStep = (step: number) => {
  workflowStep.value = step
}

// 地图路径绘制组件引用
const mapPathDrawerRef = ref<InstanceType<typeof MapPathDrawer> | null>(null)

// 显示路径规划对话框
const showPathPlanningDialog = () => {
  workflowStep.value = 3
  // 调用子组件的方法显示路径规划对话框
  if (mapPathDrawerRef.value) {
    mapPathDrawerRef.value.showPathPlanningDialog()
  }
}

// 删除操作相关方法
const clearPoints = () => {
  if (mapPathDrawerRef.value) {
    mapPathDrawerRef.value.clearPoints()
  }
}

const removeLastPoint = () => {
  if (mapPathDrawerRef.value) {
    mapPathDrawerRef.value.removeLastPoint()
  }
}

const removeSelectedPoint = () => {
  if (mapPathDrawerRef.value) {
    mapPathDrawerRef.value.removeSelectedPoint()
  }
}

const removeSelectedConnection = () => {
  if (mapPathDrawerRef.value) {
    mapPathDrawerRef.value.removeSelectedConnection()
  }
}

// 清除所有路径
const clearAllConnections = () => {
  if (mapPathDrawerRef.value) {
    mapPathDrawerRef.value.clearAllConnections()
  }
}

// 删除最后一条路径
const removeLastConnection = () => {
  if (mapPathDrawerRef.value) {
    mapPathDrawerRef.value.removeLastConnection()
  }
}

// 获取当前选中的连接ID
const selectedConnectionId = computed(() => {
  if (mapPathDrawerRef.value) {
    return mapPathDrawerRef.value.getSelectedConnectionId()
  }
  return ''
})
</script>

<style scoped>
.map-view {
  width: 100%;
  height: 100%;
  padding: 0 20px;
}

h1 {
  font-size: 2rem;
  margin-bottom: 20px;
  color: #2c3e50;
  padding-top: 10px;
}

h2 {
  font-size: 1.5rem;
  margin-bottom: 20px;
  color: #2c3e50;
}

.instructions {
  margin-bottom: 20px;
  padding: 15px 20px;
  background-color: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #3498db;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
}

.map-upload {
  margin-bottom: 30px;
  padding: 20px;
  background-color: #f8f9fa;
  border-radius: 8px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
}

.upload-container {
  display: flex;
  align-items: center;
  gap: 15px;
}

.upload-container input[type='file'] {
  display: none;
}

.upload-container button {
  padding: 12px 24px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
  transition: all 0.2s;
}

.upload-container button:hover {
  background-color: #2980b9;
  transform: translateY(-2px);
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.upload-container button:active {
  transform: translateY(0);
}

.error-message {
  margin-top: 10px;
  color: #e74c3c;
  font-weight: bold;
}

.no-map {
  padding: 60px;
  text-align: center;
  background-color: #f8f9fa;
  border-radius: 8px;
  border: 2px dashed #bdc3c7;
  font-size: 18px;
  color: #7f8c8d;
  margin-top: 20px;
}

.map-container {
  display: flex;
  gap: 20px;
  margin-bottom: 20px;
}

.map-content {
  flex: 3;
  min-width: 0; /* 防止子元素溢出 */
}

.workflow-steps {
  flex: 1;
  padding: 20px;
  background-color: #f0f8ff;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  border: 1px solid #e9ecef;
  min-width: 250px;
  max-width: 350px;
  height: fit-content;
  position: sticky;
  top: 20px;
}

.workflow-steps h3 {
  margin-top: 0;
  color: #2c3e50;
  font-size: 1.4rem;
  border-bottom: 2px solid #3498db;
  padding-bottom: 10px;
  margin-bottom: 15px;
}

.step-buttons {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
}

.step-buttons button {
  padding: 12px 15px;
  font-size: 15px;
  text-align: left;
}

.active-mode {
  background-color: #2ecc71;
  position: relative;
}

.active-mode:hover {
  background-color: #27ae60;
}

.active-mode::before {
  content: '▶';
  position: absolute;
  left: 5px;
  font-size: 12px;
}

.workflow-description {
  margin-top: 20px;
  padding: 15px;
  background-color: #ffffff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.step-description h4 {
  margin-top: 0;
  color: #2c3e50;
  font-size: 1.1rem;
  margin-bottom: 10px;
}

.step-description p {
  margin: 0;
  color: #7f8c8d;
  font-size: 14px;
  line-height: 1.5;
}

.delete-operations {
  margin-top: 20px;
  padding: 15px;
  background-color: #fff0f0;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  border-left: 4px solid #e74c3c;
}

.delete-operations h4 {
  margin-top: 0;
  color: #c0392b;
  font-size: 1.1rem;
  margin-bottom: 12px;
}

.delete-buttons {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.delete-button {
  padding: 8px 10px;
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
  text-align: left;
  display: flex;
  align-items: center;
}

.delete-button::before {
  content: '✖';
  margin-right: 8px;
  font-size: 14px;
}

.delete-button:hover {
  background-color: #c0392b;
}

.delete-button:disabled {
  background-color: #95a5a6;
  cursor: not-allowed;
  opacity: 0.7;
}

.delete-all {
  background-color: #c0392b;
  font-weight: bold;
}

.delete-all:hover {
  background-color: #962d22;
}
</style>
