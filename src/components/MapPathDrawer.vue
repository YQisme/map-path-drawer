<template>
  <div class="map-container">
    <div class="controls">
      <!-- 工作流步骤已移至父组件 -->

      <div class="line-settings">
        <div class="color-settings">
          <label>
            线条颜色:
            <input type="color" v-model="lineColor" @change="drawPath" />
          </label>
          <label>
            路径颜色:
            <input type="color" v-model="pathColor" @change="drawPath" />
          </label>
        </div>
        <label>
          线条宽度:
          <input type="number" v-model.number="lineWidth" min="1" max="10" @change="drawPath" />
        </label>
        <label>
          <input type="checkbox" v-model="showArrows" @change="drawPath" />
          显示方向箭头
        </label>
      </div>

      <div v-if="workflowStep === 3" class="animation-settings">
        <h4>动画设置</h4>
        <div class="setting-group">
          <label>
            动画速度:
            <input type="range" v-model.number="animationSpeed" min="1" max="3" />
            {{ animationSpeed }}
          </label>
          <button @click="replayAnimation" class="replay-button">重播动画</button>
        </div>
        <div class="setting-group">
          <label>
            <input type="checkbox" v-model="loopAnimation" />
            循环播放
          </label>
          <span v-if="animationLoopCount > 0" class="loop-counter">
            已循环: {{ animationLoopCount }}次
          </span>
        </div>
        <div class="setting-group" v-if="loopAnimation">
          <label>
            最大循环次数:
            <input type="number" v-model.number="maxLoopCount" min="0" max="100" />
            (0表示无限循环)
          </label>
        </div>
        <div class="setting-group">
          <label>
            路径样式:
            <select v-model="pathStyle" @change="drawPath">
              <option value="dashed">虚线</option>
              <option value="solid">实线</option>
              <option value="dotted">点线</option>
            </select>
          </label>
        </div>
        <div class="setting-group">
          <label>
            箭头样式:
            <select v-model="arrowStyle" @change="drawPath">
              <option value="triangle">三角形</option>
              <option value="arrow">箭形</option>
              <option value="circle">圆形</option>
            </select>
          </label>
        </div>

        <div class="setting-group">
          <label>
            <input type="checkbox" v-model="hideMiddleNodes" @change="drawPath" />
            隐藏中间节点（仅显示路径）
          </label>
        </div>
      </div>

      <div class="data-controls">
        <button @click="exportPathData">导出路径数据</button>
        <button @click="importPathData">导入路径数据</button>
      </div>
    </div>

    <div class="map-wrapper" ref="mapWrapper">
      <img :src="mapImageUrl" alt="地图" class="map-image" @load="onMapImageLoad" />
      <canvas ref="pathCanvas" class="path-canvas"></canvas>
      <!-- 在工作流模式下，节点由Canvas绘制 -->
      <template v-if="workflowStep === 0">
        <div
          v-for="(point, index) in points"
          :key="index"
          class="point-marker"
          :class="{
            'point-selected': index === selectedPointIndex,
            'point-start': index === 0,
            'point-end': index === points.length - 1,
          }"
          :style="{
            left: `${point.x}px`,
            top: `${point.y}px`,
          }"
          @click.stop="selectPoint(index)"
          @mousedown.stop="startDrag(index, $event)"
        >
          {{ index + 1 }}
        </div>
      </template>
    </div>

    <div v-if="showImportDialog" class="import-dialog">
      <h3>导入路径数据</h3>
      <textarea v-model="importData" placeholder="粘贴JSON格式的路径数据"></textarea>
      <div class="dialog-buttons">
        <button @click="confirmImport">确认导入</button>
        <button @click="cancelImport">取消</button>
      </div>
    </div>

    <div v-if="isAutoPathDialogVisible" class="auto-path-dialog">
      <h3>自动生成路径</h3>
      <div class="auto-path-settings">
        <div class="setting-group">
          <label>起点索引:</label>
          <select v-model="autoPathStartIndex">
            <option v-for="(_, index) in points" :key="`start-${index}`" :value="index">
              点 {{ index + 1 }}
            </option>
          </select>
        </div>
        <div class="setting-group">
          <label>终点索引:</label>
          <select v-model="autoPathEndIndex">
            <option v-for="(_, index) in points" :key="`end-${index}`" :value="index">
              点 {{ index + 1 }}
            </option>
          </select>
        </div>
        <div class="setting-group">
          <label>中间点数量:</label>
          <input type="number" v-model.number="autoPathPointCount" min="0" max="100" />
        </div>
        <div class="setting-group">
          <label>路径类型:</label>
          <select v-model="autoPathType">
            <option value="straight">直线</option>
            <option value="curve">曲线</option>
            <option value="zigzag">锯齿线</option>
            <option value="random">随机</option>
          </select>
        </div>
        <div class="setting-group" v-if="autoPathType === 'zigzag' || autoPathType === 'random'">
          <label>随机度:</label>
          <input type="range" v-model.number="autoPathRandomness" min="1" max="100" />
        </div>
      </div>
      <div class="dialog-buttons">
        <button @click="generateAutoPath">生成路径</button>
        <button @click="cancelAutoPath">取消</button>
      </div>
    </div>

    <div v-if="isPathPlanningDialogVisible" class="path-planning-dialog">
      <h3>步骤3: 计算最优路径</h3>

      <!-- 路径模式选择 -->
      <div class="path-planning-settings">
        <div class="setting-group">
          <label>路径模式:</label>
          <div class="radio-group">
            <label>
              <input type="radio" v-model="pathPlanningMode" value="simple" />
              简单路径（起点到终点）
            </label>
            <label>
              <input type="radio" v-model="pathPlanningMode" value="multi" />
              多点路径（经过多个点）
            </label>
          </div>
        </div>

        <!-- 简单路径模式（起点到终点） -->
        <template v-if="pathPlanningMode === 'simple'">
          <div class="setting-group">
            <label>起点:</label>
            <select v-model="pathPlanningStartNodeId">
              <option v-for="node in pathGraph.nodes" :key="`start-${node.id}`" :value="node.id">
                {{ node.name || `节点 ${node.id}` }}
              </option>
            </select>
          </div>
          <div class="setting-group">
            <label>终点:</label>
            <select v-model="pathPlanningEndNodeId">
              <option v-for="node in pathGraph.nodes" :key="`end-${node.id}`" :value="node.id">
                {{ node.name || `节点 ${node.id}` }}
              </option>
            </select>
          </div>
        </template>

        <!-- 多点路径模式 -->
        <template v-if="pathPlanningMode === 'multi'">
          <div class="setting-group">
            <label>选择途经点:</label>
            <div class="waypoints-container">
              <div
                v-for="(nodeId, index) in pathPlanningWaypoints"
                :key="index"
                class="waypoint-item"
              >
                <select v-model="pathPlanningWaypoints[index]">
                  <option v-for="node in pathGraph.nodes" :key="node.id" :value="node.id">
                    {{ node.name || `节点 ${node.id}` }}
                  </option>
                </select>
                <button @click="removeWaypoint(index)" class="small-button">×</button>
              </div>
              <button @click="addWaypoint" class="add-button">添加途经点</button>
            </div>
          </div>
          <div class="setting-group">
            <label>
              <input type="checkbox" v-model="optimizeWaypointOrder" />
              优化途经点顺序（旅行商问题）
            </label>
          </div>
        </template>
      </div>

      <div class="dialog-buttons">
        <button @click="findPath">计算最优路径</button>
        <button @click="cancelPathPlanning">取消</button>
      </div>

      <!-- 数据导入导出 -->
      <div class="import-export-section">
        <h4>导入/导出途经点</h4>
        <div class="setting-group">
          <textarea
            v-model="waypointsData"
            placeholder="输入JSON格式的途经点数据"
            rows="3"
          ></textarea>
          <div class="format-info">
            <small>格式: [节点ID1, 节点ID2, ...]</small>
          </div>
        </div>
        <div class="dialog-buttons">
          <button @click="importWaypoints">导入途经点</button>
          <button @click="exportWaypoints">导出途经点</button>
        </div>
      </div>
    </div>

    <div v-if="workflowStep === 1 || workflowStep === 2" class="workflow-controls">
      <h3>{{ workflowStep === 1 ? '步骤1: 预设点位' : '步骤2: 预设路径连接' }}</h3>

      <div class="workflow-info">
        <p v-if="workflowStep === 1">点击地图添加节点，完成后进入步骤2设置连接关系</p>
        <p v-if="workflowStep === 2">
          点击一个节点作为起点，再点击另一个节点作为终点，创建连接关系
        </p>
      </div>

      <div v-if="pathGraph.nodes.length > 0" class="node-list">
        <h4>节点列表</h4>
        <div v-for="node in pathGraph.nodes" :key="`node-${node.id}`" class="node-item">
          <input
            type="text"
            v-model="node.name"
            :placeholder="`节点 ${node.id}`"
            @change="updateNodeName(node.id, $event)"
          />
          <button @click="removeNode(node.id)" class="small-button">删除</button>
        </div>
      </div>

      <div v-if="workflowStep === 2 && pathGraph.connections.length > 0" class="connection-tip">
        提示: 点击路径可以选中它
      </div>

      <div v-if="workflowStep === 2 && pathGraph.connections.length > 0" class="connection-list">
        <h4>连接关系（双向）</h4>
        <div
          v-for="conn in uniqueConnections"
          :key="`conn-${conn.id}`"
          class="connection-item"
          :class="{ 'connection-selected': conn.id === selectedConnectionId }"
          @click="
            () => {
              selectedConnectionId = conn.id
              drawPathGraph()
            }
          "
        >
          <span>
            {{ getNodeName(conn.from) }} ⟷ {{ getNodeName(conn.to) }} (权重:
            {{ conn.weight.toFixed(1) }})
          </span>
          <button @click.stop="removeConnectionPair(conn.from, conn.to)" class="small-button">
            删除
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

interface Point {
  x: number
  y: number
}

interface PathData {
  points: Point[]
  lineColor: string
  pathColor: string
  lineWidth: number
  showArrows: boolean
  // 工作流模式下的额外数据
  workflowStep?: number
  pathGraph?: PathGraph
}

// 路径规划相关接口
interface PathConnection {
  from: number // 起点索引
  to: number // 终点索引
  weight: number // 路径权重（可以表示距离、难度等）
}

interface PathNode {
  id: number // 节点ID
  point: Point // 节点坐标
  name: string // 节点名称（可选）
}

interface PathGraph {
  nodes: PathNode[] // 所有节点
  connections: PathConnection[] // 节点间的连接关系
}

const props = defineProps<{
  mapImageUrl: string
  workflowStep?: number
}>()

const emit = defineEmits(['update:pathGraph'])

const mapWrapper = ref<HTMLDivElement | null>(null)
const pathCanvas = ref<HTMLCanvasElement | null>(null)
const points = ref<Point[]>([])
const selectedPointIndex = ref<number>(-1)
const lineColor = ref<string>('#ff0000') // 线条颜色
const pathColor = ref<string>('#2ecc71') // 路径颜色
const lineWidth = ref<number>(3)
const showArrows = ref<boolean>(true)
const isDragging = ref<boolean>(false)
const showImportDialog = ref<boolean>(false)
const importData = ref<string>('')

// 自动生成路径相关变量
const isAutoPathDialogVisible = ref<boolean>(false)
const autoPathStartIndex = ref<number>(0)
const autoPathEndIndex = ref<number>(1)
const autoPathPointCount = ref<number>(5)
const autoPathType = ref<'straight' | 'curve' | 'zigzag' | 'random'>('curve')
const autoPathRandomness = ref<number>(30)

// 路径规划相关变量
const pathGraph = ref<PathGraph>({
  nodes: [],
  connections: [],
})
const isPathPlanningDialogVisible = ref<boolean>(false)
const pathPlanningStartNodeId = ref<number>(-1)
const pathPlanningEndNodeId = ref<number>(-1)
const pathPlanningMode = ref<string>('simple') // 'simple' 或 'multi'
const pathPlanningWaypoints = ref<number[]>([]) // 多点路径的途经点
const optimizeWaypointOrder = ref<boolean>(false) // 是否优化途经点顺序
const waypointsData = ref<string>('') // 途经点数据导入/导出
const tempConnection = ref<{ from: number; to: number } | null>(null) // 临时连接
const selectedNodeId = ref<number>(-1) // 当前选中的节点ID
const selectedConnectionId = ref<string>('') // 当前选中的连接ID
const hoveredNodeId = ref<number>(-1) // 鼠标悬停的节点ID
const hoveredConnectionId = ref<string>('') // 鼠标悬停的连接ID

// 工作流步骤 - 从父组件接收
const workflowStep = ref<number>(props.workflowStep || 1) // 1: 预设点位, 2: 预设路径连接, 3: 计算路径

// 动画相关变量
const animationInProgress = ref<boolean>(false)
const animationProgress = ref<number>(0)
const animationPath = ref<number[]>([]) // 当前动画的路径节点ID
const animationSpeed = ref<number>(1) // 动画速度，值越大越快
const pathStyle = ref<'dashed' | 'solid' | 'dotted'>('dotted') // 路径样式
const arrowStyle = ref<'triangle' | 'arrow' | 'circle'>('arrow') // 箭头样式
const loopAnimation = ref<boolean>(true) // 是否循环播放动画
const animationLoopCount = ref<number>(0) // 动画循环次数
const maxLoopCount = ref<number>(0) // 最大循环次数，0表示无限循环
const hideMiddleNodes = ref<boolean>(false) // 是否隐藏中间节点
let animationFrameId: number | null = null

// 计算唯一连接（避免显示双向连接的两条记录）
const uniqueConnections = computed(() => {
  const result: Array<{ id: string; from: number; to: number; weight: number }> = []
  const processed = new Set<string>()

  for (const conn of pathGraph.value.connections) {
    // 创建唯一标识符，确保 A-B 和 B-A 被视为相同连接
    const id1 = `${Math.min(conn.from, conn.to)}-${Math.max(conn.from, conn.to)}`

    if (!processed.has(id1)) {
      processed.add(id1)
      result.push({
        id: id1,
        from: Math.min(conn.from, conn.to),
        to: Math.max(conn.from, conn.to),
        weight: conn.weight,
      })
    }
  }

  return result
})

// 处理地图图片加载
const onMapImageLoad = (event: Event) => {
  const img = event.target as HTMLImageElement
  if (pathCanvas.value && mapWrapper.value) {
    pathCanvas.value.width = img.width
    pathCanvas.value.height = img.height
  }
}

// 添加点位
const addPoint = (event: MouseEvent) => {
  if (!mapWrapper.value || isDragging.value) return

  const rect = mapWrapper.value.getBoundingClientRect()
  const x = event.clientX - rect.left
  const y = event.clientY - rect.top

  // 根据当前工作流步骤执行不同操作
  switch (workflowStep.value) {
    case 1: // 预设点位
      // 检查是否点击了已有的节点
      const clickedNodeId = findNearestNode(x, y)
      if (clickedNodeId !== -1) {
        // 选中已有的节点
        selectedNodeId.value = clickedNodeId
        // 重新绘制，显示选中状态
        drawNodesOnly()
      } else {
        // 添加新节点
        const newNodeId =
          pathGraph.value.nodes.length > 0
            ? Math.max(...pathGraph.value.nodes.map((n) => n.id)) + 1
            : 1

        pathGraph.value.nodes.push({
          id: newNodeId,
          point: { x, y },
          name: `节点 ${newNodeId}`,
        })

        // 绘制节点但不绘制连接
        drawNodesOnly()
      }
      break

    case 2: // 预设路径连接
      // 检查是否正在创建连接
      const isCreatingConnection = tempConnection.value !== null && tempConnection.value.from !== -1

      // 如果不是正在创建连接，才检查是否点击了已有的连接
      if (!isCreatingConnection) {
        const clickedConnectionId = findNearestConnection(x, y)
        if (clickedConnectionId) {
          // 选中连接
          selectedConnectionId.value = clickedConnectionId
          // 清除选中的节点和临时连接
          selectedNodeId.value = -1
          tempConnection.value = null
          // 重新绘制
          drawPathGraph()
          break
        }
      }

      // 检查是否点击了已有的节点
      const nodeId = findNearestNode(x, y)
      if (nodeId !== -1) {
        // 如果没有临时连接，则选择节点作为起点
        if (tempConnection.value === null) {
          // 同时设置选中的节点
          selectedNodeId.value = nodeId
          selectedConnectionId.value = '' // 清除选中的连接
          tempConnection.value = { from: nodeId, to: -1 }
        } else if (nodeId !== tempConnection.value.from) {
          // 选择连接终点并创建连接
          // 计算两点之间的距离作为权重
          const fromNode = pathGraph.value.nodes.find((n) => n.id === tempConnection.value!.from)
          const toNode = pathGraph.value.nodes.find((n) => n.id === nodeId)

          if (fromNode && toNode) {
            const dx = toNode.point.x - fromNode.point.x
            const dy = toNode.point.y - fromNode.point.y
            const distance = Math.sqrt(dx * dx + dy * dy)

            // 检查是否已存在相同的连接
            const existingConnection = pathGraph.value.connections.find(
              (c) =>
                (c.from === tempConnection.value!.from && c.to === nodeId) ||
                (c.from === nodeId && c.to === tempConnection.value!.from),
            )

            if (!existingConnection) {
              // 创建双向连接
              pathGraph.value.connections.push({
                from: tempConnection.value.from,
                to: nodeId,
                weight: distance,
              })

              // 添加反向连接
              pathGraph.value.connections.push({
                from: nodeId,
                to: tempConnection.value.from,
                weight: distance,
              })
            }
          }

          tempConnection.value = null
        }
      }

      // 绘制节点和连接
      drawPathGraph()
      break

    case 3: // 计算路径模式
      // 在此模式下点击不添加新点位
      break

    default: // 普通模式
      // 添加普通点位
      points.value.push({ x, y })
      selectedPointIndex.value = points.value.length - 1
      drawPath()
  }
}

// 选择点位
const selectPoint = (index: number) => {
  selectedPointIndex.value = index
}

// 开始拖拽点位
const startDrag = (index: number, event: MouseEvent) => {
  event.preventDefault()
  isDragging.value = true
  selectedPointIndex.value = index

  const handleMouseMove = (moveEvent: MouseEvent) => {
    if (!mapWrapper.value || !isDragging.value) return

    const rect = mapWrapper.value.getBoundingClientRect()
    const x = moveEvent.clientX - rect.left
    const y = moveEvent.clientY - rect.top

    // 更新点位坐标
    points.value[index] = { x, y }
  }

  const handleMouseUp = () => {
    isDragging.value = false
    document.removeEventListener('mousemove', handleMouseMove)
    document.removeEventListener('mouseup', handleMouseUp)
  }

  document.addEventListener('mousemove', handleMouseMove)
  document.addEventListener('mouseup', handleMouseUp)
}

// 删除最后一个点
const removeLastPoint = () => {
  if (workflowStep.value > 0) {
    // 工作流模式下，操作pathGraph.nodes
    if (pathGraph.value.nodes.length > 0) {
      // 获取最后一个节点的ID
      const lastNodeId = pathGraph.value.nodes[pathGraph.value.nodes.length - 1].id

      // 删除与该节点相关的所有连接
      pathGraph.value.connections = pathGraph.value.connections.filter(
        (conn) => conn.from !== lastNodeId && conn.to !== lastNodeId,
      )

      // 删除最后一个节点
      pathGraph.value.nodes.pop()

      // 根据当前步骤重新绘制
      if (workflowStep.value === 1) {
        drawNodesOnly()
      } else {
        drawPathGraph()
      }
    }
  } else {
    // 普通模式下，操作points数组
    if (points.value.length > 0) {
      points.value.pop()
      if (selectedPointIndex.value >= points.value.length) {
        selectedPointIndex.value = points.value.length - 1
      }
      drawPath()
    }
  }
}

// 删除选中的点
const removeSelectedPoint = () => {
  if (workflowStep.value > 0) {
    // 工作流模式下，删除选中的节点
    if (selectedNodeId.value !== -1) {
      // 删除与该节点相关的所有连接
      pathGraph.value.connections = pathGraph.value.connections.filter(
        (conn) => conn.from !== selectedNodeId.value && conn.to !== selectedNodeId.value,
      )

      // 删除选中的节点
      const nodeIndex = pathGraph.value.nodes.findIndex((node) => node.id === selectedNodeId.value)
      if (nodeIndex !== -1) {
        pathGraph.value.nodes.splice(nodeIndex, 1)
      }

      // 重置选中的节点ID
      selectedNodeId.value = -1

      // 如果有临时连接且连接的是被删除的节点，则清除临时连接
      if (
        tempConnection.value &&
        (tempConnection.value.from === selectedNodeId.value ||
          tempConnection.value.to === selectedNodeId.value)
      ) {
        tempConnection.value = null
      }

      // 根据当前步骤重新绘制
      if (workflowStep.value === 1) {
        drawNodesOnly()
      } else {
        drawPathGraph()
      }
    }
  } else {
    // 普通模式下，操作points数组
    if (selectedPointIndex.value >= 0 && selectedPointIndex.value < points.value.length) {
      points.value.splice(selectedPointIndex.value, 1)
      if (selectedPointIndex.value >= points.value.length) {
        selectedPointIndex.value = points.value.length - 1
      }
      drawPath()
    }
  }
}

// 清除所有点
const clearPoints = () => {
  if (workflowStep.value > 0) {
    // 工作流模式下，清空pathGraph
    pathGraph.value.nodes = []
    pathGraph.value.connections = []

    // 清除画布
    const canvas = pathCanvas.value
    if (canvas) {
      const ctx = canvas.getContext('2d')
      if (ctx) {
        ctx.clearRect(0, 0, canvas.width, canvas.height)
      }
    }
  } else {
    // 普通模式下，清空points数组
    points.value = []
    selectedPointIndex.value = -1
    drawPath()
  }
}

// 绘制路径
const drawPath = () => {
  const canvas = pathCanvas.value
  if (!canvas) return

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  // 清除画布
  ctx.clearRect(0, 0, canvas.width, canvas.height)

  if (points.value.length < 2) return

  // 绘制美化的路径
  drawBeautifulPath(ctx, points.value)

  // 绘制起点和终点标记
  if (points.value.length >= 2) {
    // 起点标记 - 始终显示
    const startPoint = points.value[0]
    ctx.fillStyle = '#2ecc71'
    ctx.beginPath()
    ctx.arc(startPoint.x, startPoint.y, 12, 0, Math.PI * 2)
    ctx.fill()

    ctx.fillStyle = 'white'
    ctx.font = 'bold 14px Arial'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText('S', startPoint.x, startPoint.y)

    // 终点标记 - 始终显示
    const endPoint = points.value[points.value.length - 1]
    ctx.fillStyle = '#9b59b6'
    ctx.beginPath()
    ctx.arc(endPoint.x, endPoint.y, 12, 0, Math.PI * 2)
    ctx.fill()

    ctx.fillStyle = 'white'
    ctx.font = 'bold 14px Arial'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText('E', endPoint.x, endPoint.y)

    // 隐藏中间节点时不添加任何额外信息标记
  }
}

// 绘制美化的路径
const drawBeautifulPath = (ctx: CanvasRenderingContext2D, pathPoints: Point[]) => {
  if (pathPoints.length < 2) return

  // 绘制发光路径 - 使用用户设置的路径颜色
  ctx.strokeStyle = pathColor.value
  ctx.lineWidth = hideMiddleNodes.value ? lineWidth.value + 4 : lineWidth.value + 2
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'

  // 添加发光效果 - 使用用户设置的路径颜色
  ctx.shadowColor = pathColor.value
  ctx.shadowBlur = hideMiddleNodes.value ? 15 : 10

  // 绘制主路径
  ctx.beginPath()
  ctx.moveTo(pathPoints[0].x, pathPoints[0].y)

  for (let i = 1; i < pathPoints.length; i++) {
    ctx.lineTo(pathPoints[i].x, pathPoints[i].y)
  }

  ctx.stroke()

  // 重置阴影
  ctx.shadowColor = 'transparent'
  ctx.shadowBlur = 0

  // 绘制路径上的点（如果不隐藏中间节点）
  if (!hideMiddleNodes.value) {
    for (let i = 1; i < pathPoints.length - 1; i++) {
      const point = pathPoints[i]

      // 绘制路径点
      ctx.fillStyle = '#3498db'
      ctx.beginPath()
      ctx.arc(point.x, point.y, 6, 0, Math.PI * 2)
      ctx.fill()

      // 绘制路径点边框
      ctx.strokeStyle = 'white'
      ctx.lineWidth = 2
      ctx.beginPath()
      ctx.arc(point.x, point.y, 6, 0, Math.PI * 2)
      ctx.stroke()
    }
  }

  // 绘制路径方向指示
  if (showArrows.value) {
    for (let i = 0; i < pathPoints.length - 1; i++) {
      const start = pathPoints[i]
      const end = pathPoints[i + 1]

      // 计算路径段上的多个点，放置箭头
      const segmentLength = Math.sqrt(Math.pow(end.x - start.x, 2) + Math.pow(end.y - start.y, 2))

      // 设置固定的箭头间距
      const baseDistance = 100 // 固定的箭头间距
      const arrowCount = Math.max(1, Math.floor(segmentLength / baseDistance))

      for (let j = 1; j <= arrowCount; j++) {
        // 计算箭头位置
        const ratio = j / (arrowCount + 1)
        const arrowX = start.x + (end.x - start.x) * ratio
        const arrowY = start.y + (end.y - start.y) * ratio

        // 计算方向
        const angle = Math.atan2(end.y - start.y, end.x - start.x)

        // 绘制箭头，使用不同的时间偏移使箭头动画不同步
        const timeOffset = i * 200 + j * 300
        drawFlashingPathArrow(ctx, arrowX, arrowY, angle, timeOffset)
      }
    }
  }
}

// 绘制带时间偏移的闪烁箭头
const drawFlashingPathArrow = (
  ctx: CanvasRenderingContext2D,
  x: number,
  y: number,
  angle: number,
  timeOffset: number = 0,
) => {
  // 使用时间创建闪烁效果，添加时间偏移使不同箭头闪烁不同步
  const pulseValue = Math.sin((Date.now() + timeOffset) / 300) * 0.5 + 0.5 // 0到1之间的脉冲值

  // 闪烁的箭头颜色
  const arrowColor = pulseValue > 0.5 ? '#e74c3c' : '#f39c12'

  // 箭头大小随脉冲值变化
  const arrowSize = 5 + pulseValue * 3

  // 绘制发光效果
  ctx.shadowColor = arrowColor
  ctx.shadowBlur = 10 * pulseValue

  // 绘制小圆点
  ctx.fillStyle = arrowColor
  ctx.beginPath()
  ctx.arc(x, y, 3 + pulseValue * 2, 0, Math.PI * 2)
  ctx.fill()

  // 绘制方向指示箭头
  ctx.save()
  ctx.translate(x, y)
  ctx.rotate(angle)

  // 绘制箭头主体
  ctx.fillStyle = arrowColor
  ctx.beginPath()
  ctx.moveTo(arrowSize, 0)
  ctx.lineTo(-3, -arrowSize / 2)
  ctx.lineTo(-3, arrowSize / 2)
  ctx.closePath()
  ctx.fill()

  // 绘制箭头轮廓
  ctx.strokeStyle = 'white'
  ctx.lineWidth = 1
  ctx.beginPath()
  ctx.moveTo(arrowSize, 0)
  ctx.lineTo(-3, -arrowSize / 2)
  ctx.lineTo(-3, arrowSize / 2)
  ctx.closePath()
  ctx.stroke()

  ctx.restore()

  // 重置阴影
  ctx.shadowColor = 'transparent'
  ctx.shadowBlur = 0
}

// 保留原始的路径方向指示函数以备使用
// eslint-disable-next-line @typescript-eslint/no-unused-vars
const drawPathDirectionIndicator = (
  ctx: CanvasRenderingContext2D,
  x: number,
  y: number,
  angle: number,
) => {
  // 使用时间创建闪烁效果
  const pulseValue = Math.sin(Date.now() / 200) * 0.5 + 0.5 // 0到1之间的脉冲值

  // 闪烁的箭头颜色
  const arrowColor = pulseValue > 0.5 ? '#e74c3c' : '#f39c12'

  // 箭头大小随脉冲值变化
  const arrowSize = 5 + pulseValue * 3

  // 绘制发光效果
  ctx.shadowColor = arrowColor
  ctx.shadowBlur = 10 * pulseValue

  // 绘制小圆点
  ctx.fillStyle = arrowColor
  ctx.beginPath()
  ctx.arc(x, y, 3 + pulseValue * 2, 0, Math.PI * 2)
  ctx.fill()

  // 绘制方向指示箭头
  ctx.save()
  ctx.translate(x, y)
  ctx.rotate(angle)

  // 绘制箭头主体
  ctx.fillStyle = arrowColor
  ctx.beginPath()
  ctx.moveTo(arrowSize, 0)
  ctx.lineTo(-3, -arrowSize / 2)
  ctx.lineTo(-3, arrowSize / 2)
  ctx.closePath()
  ctx.fill()

  // 绘制箭头轮廓
  ctx.strokeStyle = 'white'
  ctx.lineWidth = 1
  ctx.beginPath()
  ctx.moveTo(arrowSize, 0)
  ctx.lineTo(-3, -arrowSize / 2)
  ctx.lineTo(-3, arrowSize / 2)
  ctx.closePath()
  ctx.stroke()

  ctx.restore()

  // 重置阴影
  ctx.shadowColor = 'transparent'
  ctx.shadowBlur = 0
}

// 绘制箭头的函数已移到drawFlashingArrow中，支持多种样式

// 保留原始箭头绘制函数以备使用
// eslint-disable-next-line @typescript-eslint/no-unused-vars
const drawArrow = (ctx: CanvasRenderingContext2D, x: number, y: number, angle: number) => {
  const arrowSize = lineWidth.value * 2 + 5

  ctx.save()
  ctx.translate(x, y)
  ctx.rotate(angle)

  ctx.fillStyle = lineColor.value
  ctx.beginPath()
  ctx.moveTo(0, 0)
  ctx.lineTo(-arrowSize, -arrowSize / 2)
  ctx.lineTo(-arrowSize, arrowSize / 2)
  ctx.closePath()
  ctx.fill()

  ctx.restore()
}

// 导出路径数据
const exportPathData = () => {
  let exportPoints: Point[] = []

  // 根据当前工作流步骤决定导出哪些数据
  if (workflowStep.value > 0) {
    // 工作流模式下，导出 pathGraph 中的节点坐标
    exportPoints = pathGraph.value.nodes.map((node) => node.point)
  } else {
    // 普通模式下，导出 points 数组
    exportPoints = points.value
  }

  const data: PathData = {
    points: exportPoints,
    lineColor: lineColor.value,
    pathColor: pathColor.value,
    lineWidth: lineWidth.value,
    showArrows: showArrows.value,
    // 在工作流模式下，额外导出工作流相关数据
    ...(workflowStep.value > 0 && {
      workflowStep: workflowStep.value,
      pathGraph: pathGraph.value,
    }),
  }

  const jsonData = JSON.stringify(data, null, 2)

  // 创建下载链接
  const blob = new Blob([jsonData], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = 'path-data.json'
  a.click()
  URL.revokeObjectURL(url)
}

// 导入路径数据对话框
const importPathData = () => {
  showImportDialog.value = true
}

// 确认导入路径数据
const confirmImport = () => {
  try {
    const data: PathData = JSON.parse(importData.value)

    // 导入基本设置
    lineColor.value = data.lineColor || '#ff0000'
    pathColor.value = data.pathColor || '#2ecc71'
    lineWidth.value = data.lineWidth || 3
    showArrows.value = data.showArrows !== undefined ? data.showArrows : true

    // 检查是否包含工作流数据
    if (data.workflowStep && data.pathGraph) {
      // 导入工作流数据
      workflowStep.value = data.workflowStep
      pathGraph.value = data.pathGraph

      // 根据工作流步骤绘制相应内容
      if (workflowStep.value === 1) {
        drawNodesOnly()
      } else if (workflowStep.value === 2) {
        drawPathGraph()
      } else if (workflowStep.value === 3) {
        drawPathGraph()
      }
    } else {
      // 导入普通点位数据
      points.value = data.points || []
      drawPath()
    }

    showImportDialog.value = false
    importData.value = ''
  } catch {
    alert('导入数据格式错误，请检查JSON格式是否正确')
  }
}

// 取消导入
const cancelImport = () => {
  showImportDialog.value = false
  importData.value = ''
}

// 显示自动生成路径对话框
// 此函数保留以供将来使用，但当前未在界面上显示按钮
// eslint-disable-next-line @typescript-eslint/no-unused-vars
const showAutoPathDialog = () => {
  if (points.value.length < 2) return

  autoPathStartIndex.value = 0
  autoPathEndIndex.value = points.value.length - 1
  isAutoPathDialogVisible.value = true
}

// 取消自动生成路径
const cancelAutoPath = () => {
  isAutoPathDialogVisible.value = false
}

// 生成两点之间的直线路径点
const generateStraightPath = (start: Point, end: Point, count: number): Point[] => {
  const result: Point[] = []

  for (let i = 0; i <= count + 1; i++) {
    const ratio = i / (count + 1)
    const x = start.x + (end.x - start.x) * ratio
    const y = start.y + (end.y - start.y) * ratio

    result.push({ x, y })
  }

  return result
}

// 生成两点之间的曲线路径点
const generateCurvePath = (start: Point, end: Point, count: number): Point[] => {
  const result: Point[] = []

  // 计算控制点（曲线的弯曲程度）
  const controlPoint = {
    x: (start.x + end.x) / 2,
    y: (start.y + end.y) / 2,
  }

  // 将控制点偏移一定距离，使曲线有弧度
  const dx = end.x - start.x
  const dy = end.y - start.y
  const distance = Math.sqrt(dx * dx + dy * dy)

  // 垂直于起点和终点连线的方向
  const nx = -dy / distance
  const ny = dx / distance

  // 控制点偏移
  const offset = distance * 0.3
  controlPoint.x += nx * offset
  controlPoint.y += ny * offset

  for (let i = 0; i <= count + 1; i++) {
    const t = i / (count + 1)

    // 二次贝塞尔曲线公式
    const x = (1 - t) * (1 - t) * start.x + 2 * (1 - t) * t * controlPoint.x + t * t * end.x
    const y = (1 - t) * (1 - t) * start.y + 2 * (1 - t) * t * controlPoint.y + t * t * end.y

    result.push({ x, y })
  }

  return result
}

// 生成两点之间的锯齿线路径点
const generateZigzagPath = (
  start: Point,
  end: Point,
  count: number,
  randomness: number,
): Point[] => {
  const result: Point[] = []

  // 计算起点到终点的向量
  const dx = end.x - start.x
  const dy = end.y - start.y
  const distance = Math.sqrt(dx * dx + dy * dy)

  // 垂直于起点和终点连线的方向
  const nx = -dy / distance
  const ny = dx / distance

  // 锯齿的最大偏移量
  const maxOffset = distance * 0.2 * (randomness / 50)

  for (let i = 0; i <= count + 1; i++) {
    const ratio = i / (count + 1)
    const x = start.x + dx * ratio
    const y = start.y + dy * ratio

    // 对于中间点，添加锯齿效果
    if (i > 0 && i < count + 1) {
      // 交替方向的偏移
      const direction = i % 2 === 0 ? 1 : -1
      const offset = maxOffset * direction

      result.push({
        x: x + nx * offset,
        y: y + ny * offset,
      })
    } else {
      // 起点和终点保持不变
      result.push({ x, y })
    }
  }

  return result
}

// 生成两点之间的随机路径点
const generateRandomPath = (
  start: Point,
  end: Point,
  count: number,
  randomness: number,
): Point[] => {
  const result: Point[] = []

  // 计算起点到终点的向量
  const dx = end.x - start.x
  const dy = end.y - start.y
  const distance = Math.sqrt(dx * dx + dy * dy)

  // 随机偏移的最大值
  const maxOffset = distance * 0.3 * (randomness / 50)

  for (let i = 0; i <= count + 1; i++) {
    const ratio = i / (count + 1)
    const x = start.x + dx * ratio
    const y = start.y + dy * ratio

    // 对于中间点，添加随机偏移
    if (i > 0 && i < count + 1) {
      const offsetX = (Math.random() * 2 - 1) * maxOffset
      const offsetY = (Math.random() * 2 - 1) * maxOffset

      result.push({
        x: x + offsetX,
        y: y + offsetY,
      })
    } else {
      // 起点和终点保持不变
      result.push({ x, y })
    }
  }

  return result
}

// 自动生成路径
const generateAutoPath = () => {
  if (autoPathStartIndex.value === autoPathEndIndex.value) {
    alert('起点和终点不能是同一个点')
    return
  }

  const start = points.value[autoPathStartIndex.value]
  const end = points.value[autoPathEndIndex.value]

  // 创建新的点数组，保留起点和终点
  const newPoints: Point[] = []

  // 根据选择的路径类型生成中间点
  let pathPoints: Point[] = []

  switch (autoPathType.value) {
    case 'straight':
      pathPoints = generateStraightPath(start, end, autoPathPointCount.value)
      break
    case 'curve':
      pathPoints = generateCurvePath(start, end, autoPathPointCount.value)
      break
    case 'zigzag':
      pathPoints = generateZigzagPath(
        start,
        end,
        autoPathPointCount.value,
        autoPathRandomness.value,
      )
      break
    case 'random':
      pathPoints = generateRandomPath(
        start,
        end,
        autoPathPointCount.value,
        autoPathRandomness.value,
      )
      break
  }

  // 如果起点索引小于终点索引，保持原有顺序
  if (autoPathStartIndex.value < autoPathEndIndex.value) {
    // 复制起点之前的所有点
    for (let i = 0; i < autoPathStartIndex.value; i++) {
      newPoints.push(points.value[i])
    }

    // 添加生成的路径点
    newPoints.push(...pathPoints)

    // 复制终点之后的所有点
    for (let i = autoPathEndIndex.value + 1; i < points.value.length; i++) {
      newPoints.push(points.value[i])
    }
  } else {
    // 如果终点索引小于起点索引，需要反转路径点顺序
    // 复制终点之前的所有点
    for (let i = 0; i < autoPathEndIndex.value; i++) {
      newPoints.push(points.value[i])
    }

    // 添加生成的路径点（反转顺序）
    newPoints.push(...pathPoints.reverse())

    // 复制起点之后的所有点
    for (let i = autoPathStartIndex.value + 1; i < points.value.length; i++) {
      newPoints.push(points.value[i])
    }
  }

  // 更新点数组
  points.value = newPoints

  // 关闭对话框
  isAutoPathDialogVisible.value = false
}

// 工作流相关函数
// 设置当前工作流步骤
const setWorkflowStep = (step: number) => {
  // 停止任何正在进行的动画
  stopPathAnimation()

  // 清除动画相关状态
  animationPath.value = []
  animationProgress.value = 0
  animationInProgress.value = false
  animationLoopCount.value = 0

  workflowStep.value = step
  tempConnection.value = null

  // 如果进入步骤1且当前有普通点位但没有节点，询问是否转换
  if (step === 1 && pathGraph.value.nodes.length === 0 && points.value.length > 0) {
    const confirmConvert = confirm('是否将当前点位转换为规划节点？')
    if (confirmConvert) {
      convertPointsToNodes()
    }
  }

  // 清除画布
  const canvas = pathCanvas.value
  if (canvas) {
    const ctx = canvas.getContext('2d')
    if (ctx) {
      ctx.clearRect(0, 0, canvas.width, canvas.height)
    }
  }

  // 根据步骤更新显示
  switch (step) {
    case 1:
      drawNodesOnly()
      break
    case 2:
      drawPathGraph()
      break
    case 3:
      // 步骤3需要显示路径规划对话框
      if (pathGraph.value.nodes.length >= 2 && pathGraph.value.connections.length > 0) {
        // 初始化路径规划对话框
        if (pathGraph.value.nodes.length > 0) {
          pathPlanningStartNodeId.value = pathGraph.value.nodes[0].id
          pathPlanningEndNodeId.value = pathGraph.value.nodes[pathGraph.value.nodes.length - 1].id
          pathPlanningWaypoints.value = [pathGraph.value.nodes[0].id]
        }

        // 显示路径规划对话框
        isPathPlanningDialogVisible.value = true
      }
      break
    default:
      drawPath()
  }

  // 向父组件发送pathGraph更新
  emit('update:pathGraph', pathGraph.value)
}

// 将当前点位转换为节点
const convertPointsToNodes = () => {
  pathGraph.value.nodes = points.value.map((point, index) => ({
    id: index + 1,
    point,
    name: `节点 ${index + 1}`,
  }))

  // 清空连接，因为我们希望用户在步骤2中明确定义连接关系
  pathGraph.value.connections = []
}

// 只绘制节点，不绘制连接
const drawNodesOnly = () => {
  const canvas = pathCanvas.value
  if (!canvas) return

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  // 清除画布
  ctx.clearRect(0, 0, canvas.width, canvas.height)

  // 绘制节点标记
  for (const node of pathGraph.value.nodes) {
    // 检查节点是否被选中或悬停
    const isSelected = node.id === selectedNodeId.value
    const isHovered = node.id === hoveredNodeId.value && !isSelected

    // 绘制节点圆圈
    if (isSelected) {
      // 选中节点的样式 - 红色带发光效果
      ctx.fillStyle = '#e74c3c'
      ctx.shadowColor = '#e74c3c'
      ctx.shadowBlur = 10
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 12, 0, Math.PI * 2)
      ctx.fill()

      // 绘制选中指示器 - 外圈
      ctx.strokeStyle = 'white'
      ctx.lineWidth = 2
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 14, 0, Math.PI * 2)
      ctx.stroke()

      // 重置阴影
      ctx.shadowColor = 'transparent'
      ctx.shadowBlur = 0
    } else if (isHovered) {
      // 悬停节点的样式 - 蓝色带发光效果
      ctx.fillStyle = '#3498db'
      ctx.shadowColor = '#3498db'
      ctx.shadowBlur = 8
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 11, 0, Math.PI * 2)
      ctx.fill()

      // 绘制悬停指示器 - 外圈
      ctx.strokeStyle = 'white'
      ctx.lineWidth = 1.5
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 13, 0, Math.PI * 2)
      ctx.stroke()

      // 重置阴影
      ctx.shadowColor = 'transparent'
      ctx.shadowBlur = 0
    } else {
      // 普通节点样式
      ctx.fillStyle = '#3498db'
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 10, 0, Math.PI * 2)
      ctx.fill()
    }

    // 绘制节点编号
    ctx.fillStyle = 'white'
    ctx.font = '12px Arial'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(String(node.id), node.point.x, node.point.y)

    // 绘制节点名称
    if (node.name) {
      ctx.fillStyle = '#2c3e50'
      ctx.font = '12px Arial'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'top'
      ctx.fillText(node.name, node.point.x, node.point.y + 15)
    }
  }
}

// 查找最近的节点
const findNearestNode = (x: number, y: number): number => {
  const threshold = 20 // 点击容差范围
  let minDistance = Infinity
  let nearestNodeId = -1

  for (const node of pathGraph.value.nodes) {
    const dx = node.point.x - x
    const dy = node.point.y - y
    const distance = Math.sqrt(dx * dx + dy * dy)

    if (distance < minDistance && distance < threshold) {
      minDistance = distance
      nearestNodeId = node.id
    }
  }

  return nearestNodeId
}

// 查找最近的连接
const findNearestConnection = (x: number, y: number): string | null => {
  const threshold = 10 // 点击连接的容差范围
  let minDistance = Infinity
  let nearestConnectionId: string | null = null

  // 遍历所有连接
  for (const conn of pathGraph.value.connections) {
    const fromNode = pathGraph.value.nodes.find((n) => n.id === conn.from)
    const toNode = pathGraph.value.nodes.find((n) => n.id === conn.to)

    if (fromNode && toNode) {
      // 计算点到线段的距离
      const distance = distanceToLineSegment(
        x,
        y,
        fromNode.point.x,
        fromNode.point.y,
        toNode.point.x,
        toNode.point.y,
      )

      // 如果距离小于阈值且小于当前最小距离，则更新最近的连接
      if (distance < threshold && distance < minDistance) {
        minDistance = distance
        // 创建连接ID（确保连接ID是唯一的，不考虑方向）
        nearestConnectionId = `${Math.min(conn.from, conn.to)}-${Math.max(conn.from, conn.to)}`
      }
    }
  }

  return nearestConnectionId
}

// 计算点到线段的距离
const distanceToLineSegment = (
  px: number,
  py: number, // 点的坐标
  x1: number,
  y1: number, // 线段起点
  x2: number,
  y2: number, // 线段终点
): number => {
  const A = px - x1
  const B = py - y1
  const C = x2 - x1
  const D = y2 - y1

  const dot = A * C + B * D
  const lenSq = C * C + D * D
  let param = -1

  if (lenSq !== 0) {
    param = dot / lenSq
  }

  let xx, yy

  if (param < 0) {
    xx = x1
    yy = y1
  } else if (param > 1) {
    xx = x2
    yy = y2
  } else {
    xx = x1 + param * C
    yy = y1 + param * D
  }

  const dx = px - xx
  const dy = py - yy
  return Math.sqrt(dx * dx + dy * dy)
}

// 获取节点名称
const getNodeName = (nodeId: number): string => {
  const node = pathGraph.value.nodes.find((n) => n.id === nodeId)
  return node ? node.name || `节点 ${node.id}` : `未知节点 ${nodeId}`
}

// 更新节点名称
const updateNodeName = (nodeId: number, event: Event) => {
  const input = event.target as HTMLInputElement
  const node = pathGraph.value.nodes.find((n) => n.id === nodeId)
  if (node) {
    node.name = input.value
  }
}

// 删除节点
const removeNode = (nodeId: number) => {
  // 删除节点
  pathGraph.value.nodes = pathGraph.value.nodes.filter((n) => n.id !== nodeId)

  // 删除与该节点相关的所有连接
  pathGraph.value.connections = pathGraph.value.connections.filter(
    (c) => c.from !== nodeId && c.to !== nodeId,
  )

  drawPathGraph()
}

// 删除连接（保留此函数以便将来使用）
// eslint-disable-next-line @typescript-eslint/no-unused-vars
const removeConnection = (index: number) => {
  pathGraph.value.connections.splice(index, 1)
  drawPathGraph()
}

// 删除连接对（双向连接的两条记录）
const removeConnectionPair = (fromId: number, toId: number) => {
  // 删除 fromId -> toId 的连接
  pathGraph.value.connections = pathGraph.value.connections.filter(
    (conn) => !(conn.from === fromId && conn.to === toId),
  )

  // 删除 toId -> fromId 的连接
  pathGraph.value.connections = pathGraph.value.connections.filter(
    (conn) => !(conn.from === toId && conn.to === fromId),
  )

  // 如果删除的是当前选中的连接，清除选中状态
  const connId = `${Math.min(fromId, toId)}-${Math.max(fromId, toId)}`
  if (selectedConnectionId.value === connId) {
    selectedConnectionId.value = ''
  }

  drawPathGraph()
}

// 删除选中的连接
const removeSelectedConnection = () => {
  if (selectedConnectionId.value && workflowStep.value === 2) {
    // 解析连接ID，获取两个节点的ID
    const [fromId, toId] = selectedConnectionId.value.split('-').map(Number)

    // 删除连接对
    removeConnectionPair(fromId, toId)

    // 清除选中状态
    selectedConnectionId.value = ''

    // 重新绘制
    drawPathGraph()
  }
}

// 清除所有路径连接
const clearAllConnections = () => {
  if (workflowStep.value === 2 && pathGraph.value.connections.length > 0) {
    // 清空连接数组
    pathGraph.value.connections = []

    // 清除选中状态
    selectedConnectionId.value = ''

    // 重新绘制
    drawPathGraph()
  }
}

// 删除最后一条路径连接
const removeLastConnection = () => {
  if (workflowStep.value === 2 && pathGraph.value.connections.length > 0) {
    // 获取所有唯一连接
    const uniqueConns = [...uniqueConnections.value]

    if (uniqueConns.length > 0) {
      // 获取最后一条连接
      const lastConn = uniqueConns[uniqueConns.length - 1]

      // 删除这条连接（双向）
      removeConnectionPair(lastConn.from, lastConn.to)

      // 如果删除的是当前选中的连接，清除选中状态
      if (selectedConnectionId.value === lastConn.id) {
        selectedConnectionId.value = ''
      }

      // 重新绘制
      drawPathGraph()
    }
  }
}

// 绘制路径图
const drawPathGraph = () => {
  const canvas = pathCanvas.value
  if (!canvas) return

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  // 清除画布
  ctx.clearRect(0, 0, canvas.width, canvas.height)

  // 绘制连接
  for (const conn of pathGraph.value.connections) {
    const fromNode = pathGraph.value.nodes.find((n) => n.id === conn.from)
    const toNode = pathGraph.value.nodes.find((n) => n.id === conn.to)

    if (fromNode && toNode) {
      // 创建连接ID
      const connId = `${Math.min(conn.from, conn.to)}-${Math.max(conn.from, conn.to)}`

      // 检查是否是选中的连接或悬停的连接
      const isSelected = connId === selectedConnectionId.value
      const isHovered = connId === hoveredConnectionId.value && !isSelected

      // 根据选中状态或悬停状态设置样式
      if (isSelected) {
        // 选中的连接使用高亮样式
        ctx.strokeStyle = '#e74c3c' // 红色
        ctx.lineWidth = lineWidth.value + 2
        ctx.shadowColor = '#e74c3c'
        ctx.shadowBlur = 10
      } else if (isHovered) {
        // 悬停的连接使用轻微高亮样式
        ctx.strokeStyle = '#3498db' // 蓝色
        ctx.lineWidth = lineWidth.value + 1
        ctx.shadowColor = '#3498db'
        ctx.shadowBlur = 5
      } else {
        // 普通连接使用默认样式
        ctx.strokeStyle = lineColor.value
        ctx.lineWidth = lineWidth.value
        ctx.shadowColor = 'transparent'
        ctx.shadowBlur = 0
      }

      // 绘制连接线
      ctx.beginPath()
      ctx.moveTo(fromNode.point.x, fromNode.point.y)
      ctx.lineTo(toNode.point.x, toNode.point.y)
      ctx.stroke()

      // 在双向路径模式下，不绘制箭头
      // 但我们可以在连接中点绘制一个小圆点表示连接
      const midX = (fromNode.point.x + toNode.point.x) / 2
      const midY = (fromNode.point.y + toNode.point.y) / 2

      // 根据选中状态或悬停状态设置中点圆点样式
      if (isSelected) {
        ctx.fillStyle = '#e74c3c'
        ctx.beginPath()
        ctx.arc(midX, midY, 5, 0, Math.PI * 2)
        ctx.fill()

        // 绘制外圈
        ctx.strokeStyle = 'white'
        ctx.lineWidth = 2
        ctx.beginPath()
        ctx.arc(midX, midY, 7, 0, Math.PI * 2)
        ctx.stroke()
      } else if (isHovered) {
        ctx.fillStyle = '#3498db'
        ctx.beginPath()
        ctx.arc(midX, midY, 4, 0, Math.PI * 2)
        ctx.fill()

        // 绘制外圈
        ctx.strokeStyle = 'white'
        ctx.lineWidth = 1
        ctx.beginPath()
        ctx.arc(midX, midY, 6, 0, Math.PI * 2)
        ctx.stroke()
      } else {
        ctx.fillStyle = lineColor.value
        ctx.beginPath()
        ctx.arc(midX, midY, 3, 0, Math.PI * 2)
        ctx.fill()
      }

      // 重置阴影
      ctx.shadowColor = 'transparent'
      ctx.shadowBlur = 0
    }
  }

  // 绘制临时连接
  if (tempConnection.value && tempConnection.value.from !== -1) {
    const fromNode = pathGraph.value.nodes.find((n) => n.id === tempConnection.value!.from)

    if (fromNode && mapWrapper.value) {
      const rect = mapWrapper.value.getBoundingClientRect()
      const mouseX = lastMousePosition.x - rect.left
      const mouseY = lastMousePosition.y - rect.top

      ctx.strokeStyle = '#999'
      ctx.lineWidth = 2
      ctx.setLineDash([5, 5])
      ctx.beginPath()
      ctx.moveTo(fromNode.point.x, fromNode.point.y)
      ctx.lineTo(mouseX, mouseY)
      ctx.stroke()
      ctx.setLineDash([])
    }
  }

  // 绘制节点
  for (const node of pathGraph.value.nodes) {
    // 检查节点是否被选中或悬停
    const isSelected = node.id === selectedNodeId.value
    const isHovered = node.id === hoveredNodeId.value && !isSelected

    // 绘制节点圆圈
    if (isSelected) {
      // 选中节点的样式 - 红色带发光效果
      ctx.fillStyle = '#e74c3c'
      ctx.shadowColor = '#e74c3c'
      ctx.shadowBlur = 10
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 12, 0, Math.PI * 2)
      ctx.fill()

      // 绘制选中指示器 - 外圈
      ctx.strokeStyle = 'white'
      ctx.lineWidth = 2
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 14, 0, Math.PI * 2)
      ctx.stroke()

      // 重置阴影
      ctx.shadowColor = 'transparent'
      ctx.shadowBlur = 0
    } else if (isHovered) {
      // 悬停节点的样式 - 蓝色带发光效果
      ctx.fillStyle = '#3498db'
      ctx.shadowColor = '#3498db'
      ctx.shadowBlur = 8
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 11, 0, Math.PI * 2)
      ctx.fill()

      // 绘制悬停指示器 - 外圈
      ctx.strokeStyle = 'white'
      ctx.lineWidth = 1.5
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 13, 0, Math.PI * 2)
      ctx.stroke()

      // 重置阴影
      ctx.shadowColor = 'transparent'
      ctx.shadowBlur = 0
    } else {
      // 普通节点样式
      ctx.fillStyle = '#3498db'
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 10, 0, Math.PI * 2)
      ctx.fill()
    }

    // 绘制节点编号
    ctx.fillStyle = 'white'
    ctx.font = '12px Arial'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(String(node.id), node.point.x, node.point.y)

    // 绘制节点名称
    if (node.name) {
      ctx.fillStyle = '#2c3e50'
      ctx.font = '12px Arial'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'top'
      ctx.fillText(node.name, node.point.x, node.point.y + 15)
    }
  }

  // 高亮显示临时连接的起点
  if (tempConnection.value && tempConnection.value.from !== -1) {
    const fromNode = pathGraph.value.nodes.find((n) => n.id === tempConnection.value!.from)

    if (fromNode) {
      ctx.fillStyle = '#e74c3c'
      ctx.beginPath()
      ctx.arc(fromNode.point.x, fromNode.point.y, 12, 0, Math.PI * 2)
      ctx.fill()

      ctx.fillStyle = 'white'
      ctx.font = '12px Arial'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'middle'
      ctx.fillText(String(fromNode.id), fromNode.point.x, fromNode.point.y)
    }
  }
}

// 监听工作流步骤变化，当进入步骤3时显示路径规划对话框

// 取消路径规划
const cancelPathPlanning = () => {
  isPathPlanningDialogVisible.value = false
}

// 添加途经点
const addWaypoint = () => {
  if (pathGraph.value.nodes.length > 0) {
    // 添加一个默认节点（不重复添加已有的节点）
    const existingIds = new Set(pathPlanningWaypoints.value)
    const availableNode = pathGraph.value.nodes.find((node) => !existingIds.has(node.id))

    if (availableNode) {
      pathPlanningWaypoints.value.push(availableNode.id)
    } else if (pathGraph.value.nodes.length > 0) {
      // 如果所有节点都已添加，则添加第一个节点
      pathPlanningWaypoints.value.push(pathGraph.value.nodes[0].id)
    }
  }
}

// 移除途经点
const removeWaypoint = (index: number) => {
  pathPlanningWaypoints.value.splice(index, 1)
}

// 导出途经点数据
const exportWaypoints = () => {
  waypointsData.value = JSON.stringify(pathPlanningWaypoints.value)
}

// 导入途经点数据
const importWaypoints = () => {
  try {
    const data = JSON.parse(waypointsData.value)
    if (Array.isArray(data)) {
      // 验证导入的节点ID是否存在
      const validNodeIds = new Set(pathGraph.value.nodes.map((node) => node.id))
      const validWaypoints = data.filter((id) => validNodeIds.has(Number(id)))

      if (validWaypoints.length > 0) {
        pathPlanningWaypoints.value = validWaypoints.map((id) => Number(id))
      } else {
        alert('导入的途经点数据无效，请确保节点ID存在')
      }
    } else {
      alert('导入的数据格式不正确，请使用JSON数组格式')
    }
  } catch {
    alert('导入数据解析失败，请检查JSON格式是否正确')
  }
}

// 寻找最短路径 (Dijkstra算法)
const findPath = () => {
  // 先停止任何正在进行的动画并清除之前的路径
  stopPathAnimation()
  animationPath.value = []

  // 清除画布
  const canvas = pathCanvas.value
  if (canvas) {
    const ctx = canvas.getContext('2d')
    if (ctx) {
      ctx.clearRect(0, 0, canvas.width, canvas.height)
    }
  }

  // 简单路径模式（起点到终点）
  if (pathPlanningMode.value === 'simple') {
    if (pathPlanningStartNodeId.value === -1 || pathPlanningEndNodeId.value === -1) {
      alert('请选择起点和终点')
      return
    }

    if (pathPlanningStartNodeId.value === pathPlanningEndNodeId.value) {
      alert('起点和终点不能相同')
      return
    }

    // 使用Dijkstra算法寻找最短路径
    const path = dijkstra(
      pathGraph.value,
      pathPlanningStartNodeId.value,
      pathPlanningEndNodeId.value,
    )

    if (path.length === 0) {
      alert('无法找到从起点到终点的路径')
      return
    }

    // 关闭对话框
    isPathPlanningDialogVisible.value = false

    // 设置为步骤3
    workflowStep.value = 3

    // 启动路径动画
    startPathAnimation(path)

    // 显示路径信息
    const startNode = pathGraph.value.nodes.find((n) => n.id === pathPlanningStartNodeId.value)
    const endNode = pathGraph.value.nodes.find((n) => n.id === pathPlanningEndNodeId.value)

    if (startNode && endNode) {
      setTimeout(() => {
        alert(
          `已找到从 ${startNode.name || `节点 ${startNode.id}`} 到 ${endNode.name || `节点 ${endNode.id}`} 的最优路径，共经过 ${path.length} 个节点。`,
        )
      }, 100)
    }
  }
  // 多点路径模式
  else if (pathPlanningMode.value === 'multi') {
    if (pathPlanningWaypoints.value.length < 2) {
      alert('请至少添加两个途经点')
      return
    }

    // 检查是否有重复的途经点
    const uniqueWaypoints = new Set(pathPlanningWaypoints.value)
    if (uniqueWaypoints.size !== pathPlanningWaypoints.value.length) {
      alert('途经点中存在重复节点，请确保每个节点只出现一次')
      return
    }

    let waypointsToUse = [...pathPlanningWaypoints.value]

    // 如果需要优化途经点顺序（旅行商问题）
    if (optimizeWaypointOrder.value && waypointsToUse.length > 2) {
      waypointsToUse = optimizePathOrder(waypointsToUse)
    }

    // 计算多点路径
    const completePath = findMultiPointPath(waypointsToUse)

    if (completePath.length === 0) {
      alert('无法找到经过所有途经点的路径')
      return
    }

    // 关闭对话框
    isPathPlanningDialogVisible.value = false

    // 设置为步骤3
    workflowStep.value = 3

    // 启动路径动画（已在函数开始时清除了之前的路径）
    startPathAnimation(completePath)

    // 显示路径信息
    const nodeNames = waypointsToUse.map((id) => {
      const node = pathGraph.value.nodes.find((n) => n.id === id)
      return node ? node.name || `节点 ${node.id}` : `未知节点 ${id}`
    })

    setTimeout(() => {
      alert(
        `已找到经过所有途经点的最优路径，共经过 ${completePath.length} 个节点。\n途经顺序: ${nodeNames.join(' → ')}`,
      )
    }, 100)
  }
}

// 启动路径动画
const startPathAnimation = (path: number[]) => {
  // 停止之前的动画
  stopPathAnimation()

  // 设置动画参数
  animationPath.value = path
  animationProgress.value = 0
  animationInProgress.value = true
  animationLoopCount.value = 0 // 重置循环计数

  // 开始动画循环
  animatePathStep()
}

// 重播动画
const replayAnimation = () => {
  if (animationPath.value.length > 0) {
    // 清除画布
    const canvas = pathCanvas.value
    if (canvas) {
      const ctx = canvas.getContext('2d')
      if (ctx) {
        ctx.clearRect(0, 0, canvas.width, canvas.height)
      }
    }

    // 重新开始动画
    startPathAnimation([...animationPath.value])
  }
}

// 停止路径动画
const stopPathAnimation = () => {
  if (animationFrameId !== null) {
    cancelAnimationFrame(animationFrameId)
    animationFrameId = null
  }
  animationInProgress.value = false
}

// 动画步骤
const animatePathStep = () => {
  if (!animationInProgress.value || animationPath.value.length < 2) {
    stopPathAnimation()
    return
  }

  // 更新动画进度
  animationProgress.value += animationSpeed.value / 200

  // 如果动画完成
  if (animationProgress.value >= 1) {
    // 将找到的路径转换为点位，但不更新points数组
    // 这样可以避免调用drawPath()函数，该函数会导致所有节点都闪烁

    // 继续使用drawAnimatedPath绘制路径，这样只有路径上的节点会闪烁
    animationProgress.value = 1 // 确保显示完整路径
    drawAnimatedPath()

    // 检查是否需要循环播放
    if (loopAnimation.value) {
      // 增加循环计数
      animationLoopCount.value++

      // 检查是否达到最大循环次数
      if (maxLoopCount.value > 0 && animationLoopCount.value >= maxLoopCount.value) {
        // 达到最大循环次数，停止动画
        stopPathAnimation()
        return
      }

      // 重置动画进度，继续循环
      animationProgress.value = 0

      // 添加短暂延迟，使循环更加明显
      setTimeout(() => {
        // 如果动画仍然处于活动状态，请求下一帧
        if (animationInProgress.value) {
          animationFrameId = requestAnimationFrame(animatePathStep)
        }
      }, 500) // 500毫秒延迟
    } else {
      // 不循环播放，停止动画
      stopPathAnimation()
    }
    return
  }

  // 绘制当前动画帧
  drawAnimatedPath()

  // 请求下一帧
  animationFrameId = requestAnimationFrame(animatePathStep)
}

// 绘制动画路径
const drawAnimatedPath = () => {
  const canvas = pathCanvas.value
  if (!canvas) return

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  // 清除画布
  ctx.clearRect(0, 0, canvas.width, canvas.height)

  // 绘制所有节点
  for (const node of pathGraph.value.nodes) {
    // 判断节点是否在路径中
    const isInPath = animationPath.value.includes(node.id)

    // 判断是否是起点或终点
    const isStartOrEnd =
      node.id === animationPath.value[0] ||
      node.id === animationPath.value[animationPath.value.length - 1]

    // 如果隐藏中间节点且当前节点是中间节点，则跳过
    if (hideMiddleNodes.value && isInPath && !isStartOrEnd) {
      continue
    }

    // 只有在路径中的节点才会闪烁，其他节点保持原样
    if (isInPath) {
      // 绘制节点圆圈 - 路径中的节点使用闪烁效果
      const pulseValue = Math.sin(Date.now() / 300) * 0.5 + 0.5 // 0到1之间的脉冲值
      ctx.fillStyle = '#e74c3c'
      ctx.shadowColor = '#e74c3c'
      ctx.shadowBlur = 10 * pulseValue

      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 10 + pulseValue * 3, 0, Math.PI * 2)
      ctx.fill()

      // 重置阴影
      ctx.shadowColor = 'transparent'
      ctx.shadowBlur = 0
    } else {
      // 非路径中的节点使用普通样式
      ctx.fillStyle = '#3498db'
      ctx.beginPath()
      ctx.arc(node.point.x, node.point.y, 10, 0, Math.PI * 2)
      ctx.fill()
    }

    // 绘制节点编号
    ctx.fillStyle = 'white'
    ctx.font = '12px Arial'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(String(node.id), node.point.x, node.point.y)

    // 绘制节点名称
    if (node.name) {
      ctx.fillStyle = '#2c3e50'
      ctx.font = '12px Arial'
      ctx.textAlign = 'center'
      ctx.textBaseline = 'top'
      ctx.fillText(node.name, node.point.x, node.point.y + 15)
    }
  }

  // 计算当前动画进度对应的路径段
  const pathProgress = animationProgress.value * (animationPath.value.length - 1)
  const currentSegment = Math.floor(pathProgress)
  const segmentProgress = pathProgress - currentSegment

  // 绘制已完成的路径段 - 根据选择的路径样式和颜色
  ctx.strokeStyle = pathColor.value
  ctx.lineWidth = 4
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'

  // 根据路径样式设置不同的线型
  switch (pathStyle.value) {
    case 'dashed':
      // 创建虚线效果 - 根据当前时间动态变化虚线间隔
      const dashOffset = (Date.now() / 200) % 20 // 动态偏移量，使虚线看起来在移动
      const dashSize = 10 // 虚线段长度
      const gapSize = 5 // 虚线间隔长度
      ctx.setLineDash([dashSize, gapSize]) // 设置虚线样式
      ctx.lineDashOffset = -dashOffset // 设置虚线偏移，负值使虚线向前移动
      break
    case 'solid':
      // 实线不需要设置虚线样式
      ctx.setLineDash([])
      break
    case 'dotted':
      // 点线效果
      const dotOffset = (Date.now() / 160) % 16 // 动态偏移量，使点线看起来在移动
      ctx.setLineDash([2, 8]) // 设置点线样式（小点，较大间隔）
      ctx.lineDashOffset = -dotOffset
      break
  }

  // 添加发光效果
  ctx.shadowColor = pathColor.value
  ctx.shadowBlur = 10

  ctx.beginPath()

  // 绘制完整的路径段
  for (let i = 0; i < currentSegment; i++) {
    const fromNodeId = animationPath.value[i]
    const toNodeId = animationPath.value[i + 1]

    const fromNode = pathGraph.value.nodes.find((n) => n.id === fromNodeId)
    const toNode = pathGraph.value.nodes.find((n) => n.id === toNodeId)

    if (fromNode && toNode) {
      if (i === 0) {
        ctx.moveTo(fromNode.point.x, fromNode.point.y)
      }
      ctx.lineTo(toNode.point.x, toNode.point.y)
    }
  }

  // 绘制当前动画中的路径段
  if (currentSegment < animationPath.value.length - 1) {
    const fromNodeId = animationPath.value[currentSegment]
    const toNodeId = animationPath.value[currentSegment + 1]

    const fromNode = pathGraph.value.nodes.find((n) => n.id === fromNodeId)
    const toNode = pathGraph.value.nodes.find((n) => n.id === toNodeId)

    if (fromNode && toNode) {
      if (currentSegment === 0) {
        ctx.moveTo(fromNode.point.x, fromNode.point.y)
      }

      // 计算当前点的位置
      const currentX = fromNode.point.x + (toNode.point.x - fromNode.point.x) * segmentProgress
      const currentY = fromNode.point.y + (toNode.point.y - fromNode.point.y) * segmentProgress

      ctx.lineTo(currentX, currentY)

      // 绘制动画头部
      ctx.stroke()

      // 绘制动画头部的圆点
      ctx.fillStyle = '#e74c3c'
      ctx.beginPath()
      ctx.arc(currentX, currentY, 8, 0, Math.PI * 2)
      ctx.fill()

      // 绘制波纹效果
      ctx.strokeStyle = 'rgba(231, 76, 60, 0.5)'
      ctx.lineWidth = 2
      ctx.beginPath()
      ctx.arc(currentX, currentY, 15 + Math.sin(Date.now() / 200) * 5, 0, Math.PI * 2)
      ctx.stroke()

      // 计算方向角度
      const dx = toNode.point.x - fromNode.point.x
      const dy = toNode.point.y - fromNode.point.y
      const arrowAngle = Math.atan2(dy, dx)

      // 绘制闪烁的箭头
      drawFlashingArrow(ctx, currentX, currentY, arrowAngle)
    }
  } else {
    ctx.stroke()
  }

  // 重置阴影和虚线设置
  ctx.shadowColor = 'transparent'
  ctx.shadowBlur = 0
  ctx.setLineDash([])
  ctx.lineDashOffset = 0
}

// 绘制动画箭头
const drawFlashingArrow = (ctx: CanvasRenderingContext2D, x: number, y: number, angle: number) => {
  // 创建动画效果
  const pulseValue = Math.sin(Date.now() / 200) * 0.5 + 0.5 // 0到1之间的脉冲值

  // 箭头基础大小
  const baseSize = 10
  const arrowSize = baseSize + pulseValue * 3

  // 绘制发光效果
  ctx.shadowColor = '#e74c3c'
  ctx.shadowBlur = 10 * pulseValue

  // 保存上下文
  ctx.save()
  ctx.translate(x, y)
  ctx.rotate(angle)

  // 根据箭头样式绘制不同形状
  ctx.fillStyle = `rgba(231, 76, 60, ${0.7 + pulseValue * 0.3})`

  switch (arrowStyle.value) {
    case 'triangle': // 三角形箭头
      ctx.beginPath()
      ctx.moveTo(arrowSize, 0)
      ctx.lineTo(-arrowSize / 2, -arrowSize / 2)
      ctx.lineTo(-arrowSize / 2, arrowSize / 2)
      ctx.closePath()
      ctx.fill()
      break

    case 'arrow': // 箭形箭头
      ctx.beginPath()
      ctx.moveTo(arrowSize, 0)
      ctx.lineTo(-arrowSize / 2, -arrowSize / 2)
      ctx.lineTo(-arrowSize / 4, 0)
      ctx.lineTo(-arrowSize / 2, arrowSize / 2)
      ctx.closePath()
      ctx.fill()

      // 绘制箭头内部
      ctx.fillStyle = `rgba(255, 255, 255, ${pulseValue})`
      ctx.beginPath()
      ctx.moveTo(arrowSize * 0.6, 0)
      ctx.lineTo(-arrowSize / 4, -arrowSize / 4)
      ctx.lineTo(-arrowSize / 8, 0)
      ctx.lineTo(-arrowSize / 4, arrowSize / 4)
      ctx.closePath()
      ctx.fill()
      break

    case 'circle': // 圆形箭头
      // 绘制圆形
      ctx.beginPath()
      ctx.arc(arrowSize / 2, 0, arrowSize / 2, 0, Math.PI * 2)
      ctx.fill()

      // 绘制小尾巴
      ctx.beginPath()
      ctx.moveTo(0, 0)
      ctx.lineTo(-arrowSize / 2, -arrowSize / 4)
      ctx.lineTo(-arrowSize / 2, arrowSize / 4)
      ctx.closePath()
      ctx.fill()
      break
  }

  // 恢复上下文
  ctx.restore()

  // 重置阴影
  ctx.shadowColor = 'transparent'
  ctx.shadowBlur = 0
}

// Dijkstra算法实现
const dijkstra = (graph: PathGraph, startId: number, endId: number): number[] => {
  // 初始化距离表
  const distances: Record<number, number> = {}
  const previous: Record<number, number | null> = {}
  const unvisited: Set<number> = new Set()

  // 初始化所有节点的距离为无穷大
  for (const node of graph.nodes) {
    distances[node.id] = Infinity
    previous[node.id] = null
    unvisited.add(node.id)
  }

  // 起点距离为0
  distances[startId] = 0

  while (unvisited.size > 0) {
    // 找到未访问节点中距离最小的节点
    let current: number | null = null
    let minDistance = Infinity

    for (const nodeId of unvisited) {
      if (distances[nodeId] < minDistance) {
        minDistance = distances[nodeId]
        current = nodeId
      }
    }

    // 如果没有可访问的节点或已经到达终点，结束循环
    if (current === null || current === endId) break

    // 从未访问集合中移除当前节点
    unvisited.delete(current)

    // 更新相邻节点的距离
    for (const conn of graph.connections) {
      if (conn.from === current && unvisited.has(conn.to)) {
        const distance = distances[current] + conn.weight
        if (distance < distances[conn.to]) {
          distances[conn.to] = distance
          previous[conn.to] = current
        }
      }
    }
  }

  // 构建路径
  const path: number[] = []
  let current = endId

  // 如果终点不可达
  if (previous[current] === null && current !== startId) {
    return []
  }

  // 从终点回溯到起点
  while (current !== null) {
    path.unshift(current)
    const prevNode = previous[current]
    if (prevNode === null) break
    current = prevNode
  }

  return path
}

// 计算两点之间的距离
const calculateDistance = (nodeId1: number, nodeId2: number): number => {
  const node1 = pathGraph.value.nodes.find((n) => n.id === nodeId1)
  const node2 = pathGraph.value.nodes.find((n) => n.id === nodeId2)

  if (!node1 || !node2) return Infinity

  const dx = node2.point.x - node1.point.x
  const dy = node2.point.y - node1.point.y
  return Math.sqrt(dx * dx + dy * dy)
}

// 计算多点路径
const findMultiPointPath = (waypoints: number[]): number[] => {
  if (waypoints.length < 2) return []

  // 分段计算路径
  const completePath: number[] = []

  for (let i = 0; i < waypoints.length - 1; i++) {
    const segmentPath = dijkstra(pathGraph.value, waypoints[i], waypoints[i + 1])

    if (segmentPath.length === 0) {
      // 如果某段路径无法计算，则返回空路径
      return []
    }

    // 将分段路径添加到完整路径中（避免重复添加连接点）
    if (i === 0) {
      completePath.push(...segmentPath)
    } else {
      completePath.push(...segmentPath.slice(1))
    }
  }

  return completePath
}

// 优化途经点顺序（近似解决旅行商问题 - 使用贪心算法）
const optimizePathOrder = (waypoints: number[]): number[] => {
  if (waypoints.length <= 2) return waypoints

  // 使用第一个点作为起点
  const startPoint = waypoints[0]
  const remainingPoints = waypoints.slice(1)
  const optimizedPath = [startPoint]

  // 贪心算法：每次选择距离当前点最近的点
  let currentPoint = startPoint

  while (remainingPoints.length > 0) {
    // 计算当前点到所有剩余点的距离
    const distances = remainingPoints.map((point) => ({
      point,
      distance: calculateDistance(currentPoint, point),
    }))

    // 找到最近的点
    distances.sort((a, b) => a.distance - b.distance)
    const nearestPoint = distances[0].point

    // 添加到路径中并从剩余点中移除
    optimizedPath.push(nearestPoint)
    const index = remainingPoints.indexOf(nearestPoint)
    remainingPoints.splice(index, 1)

    // 更新当前点
    currentPoint = nearestPoint
  }

  return optimizedPath
}

// 跟踪鼠标位置（用于绘制临时连接线）
const lastMousePosition = { x: 0, y: 0 }

const trackMousePosition = (event: MouseEvent) => {
  lastMousePosition.x = event.clientX
  lastMousePosition.y = event.clientY

  // 在工作流步骤1（预设点位）和步骤2（预设路径连接）中处理悬停效果
  if (workflowStep.value === 1 || workflowStep.value === 2) {
    // 如果有地图画布，检测悬停
    if (pathCanvas.value) {
      const rect = pathCanvas.value.getBoundingClientRect()
      const x = event.clientX - rect.left
      const y = event.clientY - rect.top

      // 检查是否正在创建连接
      const isCreatingConnection = tempConnection.value !== null && tempConnection.value.from !== -1

      // 如果不是正在创建连接，检查鼠标是否悬停在路径或节点上
      if (!isCreatingConnection) {
        // 检查鼠标是否悬停在节点上
        const nodeId = findNearestNode(x, y)
        if (nodeId !== hoveredNodeId.value) {
          hoveredNodeId.value = nodeId

          // 在步骤1中使用drawNodesOnly，在步骤2中使用drawPathGraph
          if (workflowStep.value === 1) {
            drawNodesOnly()
          } else {
            drawPathGraph()
          }
        }

        // 只在步骤2中检查路径悬停
        if (workflowStep.value === 2) {
          // 检查鼠标是否悬停在路径上
          const connectionId = findNearestConnection(x, y)
          if (connectionId !== hoveredConnectionId.value) {
            hoveredConnectionId.value = connectionId || ''
            drawPathGraph() // 重绘以显示悬停效果
          }
        }
      } else {
        // 在创建连接过程中，暂时忽略路径选择和悬停
        selectedConnectionId.value = ''
        hoveredConnectionId.value = ''
        hoveredNodeId.value = -1
        drawPathGraph()
      }
    }
  } else {
    // 在其他工作流步骤中，清除悬停状态
    hoveredConnectionId.value = ''
    hoveredNodeId.value = -1
  }
}

onMounted(() => {
  if (mapWrapper.value) {
    mapWrapper.value.addEventListener('click', addPoint)
  }

  // 添加鼠标移动事件监听
  document.addEventListener('mousemove', trackMousePosition)

  // 初始化时根据当前步骤绘制内容
  setTimeout(() => {
    if (workflowStep.value === 1) {
      drawNodesOnly()
    } else if (workflowStep.value === 2) {
      drawPathGraph()
    }
  }, 100)
})

onUnmounted(() => {
  if (mapWrapper.value) {
    mapWrapper.value.removeEventListener('click', addPoint)
  }

  // 移除鼠标移动事件监听
  document.removeEventListener('mousemove', trackMousePosition)
})

watch(
  points,
  () => {
    drawPath()
  },
  { deep: true },
)

// 监听props中的workflowStep变化
watch(
  () => props.workflowStep,
  (newStep) => {
    if (newStep !== undefined) {
      // 即使当前已经是步骤3，也允许再次点击步骤3
      // 停止任何正在进行的动画并清除动画路径
      stopPathAnimation()
      animationPath.value = []

      // 先清除之前的状态
      setWorkflowStep(newStep)
    }
  },
)

// 监听pathGraph变化，向父组件发送更新
watch(
  pathGraph,
  (newValue) => {
    emit('update:pathGraph', newValue)
  },
  { deep: true },
)

// 显示路径规划对话框的方法，供父组件调用
const showPathPlanningDialogExposed = () => {
  // 停止任何正在进行的动画并清除动画路径
  stopPathAnimation()
  animationPath.value = []

  // 清除画布
  const canvas = pathCanvas.value
  if (canvas) {
    const ctx = canvas.getContext('2d')
    if (ctx) {
      ctx.clearRect(0, 0, canvas.width, canvas.height)
    }
  }

  // 设置为步骤3
  workflowStep.value = 3

  // 初始化路径规划对话框
  if (pathGraph.value.nodes.length > 0) {
    pathPlanningStartNodeId.value = pathGraph.value.nodes[0].id
    pathPlanningEndNodeId.value = pathGraph.value.nodes[pathGraph.value.nodes.length - 1].id
    pathPlanningWaypoints.value = [pathGraph.value.nodes[0].id]
  }

  // 显示路径规划对话框
  isPathPlanningDialogVisible.value = true
}

// 暴露方法和变量给父组件
defineExpose({
  showPathPlanningDialog: showPathPlanningDialogExposed,
  clearPoints,
  removeLastPoint,
  removeSelectedPoint,
  removeSelectedConnection,
  clearAllConnections,
  removeLastConnection,
  // 暴露selectedConnectionId变量的getter和setter
  getSelectedConnectionId: () => selectedConnectionId.value,
  setSelectedConnectionId: (id: string) => {
    selectedConnectionId.value = id
  },
})
</script>

<style scoped>
.map-container {
  display: flex;
  flex-direction: column;
  gap: 16px;
  max-width: 100%;
  position: relative;
}

.map-wrapper {
  position: relative;
  max-width: 100%;
  overflow: auto;
  border: 1px solid #ddd;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.map-image {
  display: block;
  max-width: 100%;
}

.path-canvas {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
}

.point-marker {
  position: absolute;
  width: 24px;
  height: 24px;
  background-color: #3498db;
  color: white;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  transform: translate(-50%, -50%);
  cursor: pointer;
  z-index: 10;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  transition: all 0.2s ease;
}

.point-selected {
  background-color: #e74c3c;
  transform: translate(-50%, -50%) scale(1.2);
  z-index: 20;
}

.point-start {
  background-color: #2ecc71;
  border: 2px solid white;
}

.point-end {
  background-color: #9b59b6;
  border: 2px solid white;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 20px;
  background-color: #f8f9fa;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  border: 1px solid #e9ecef;
  margin-bottom: 20px;
}

.button-group,
.line-settings,
.data-controls {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  align-items: center;
  padding: 10px;
  background-color: #ffffff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

button {
  padding: 8px 16px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}

button:hover {
  background-color: #2980b9;
}

button:disabled {
  background-color: #95a5a6;
  cursor: not-allowed;
}

.line-settings {
  padding: 10px;
  background-color: #ffffff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.color-settings {
  display: flex;
  gap: 15px;
  margin-bottom: 10px;
}

label {
  display: flex;
  align-items: center;
  gap: 8px;
}

input[type='number'] {
  width: 60px;
  padding: 4px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

input[type='color'] {
  width: 40px;
  height: 24px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.import-dialog,
.auto-path-dialog {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 100;
  width: 80%;
  max-width: 500px;
}

.import-dialog h3,
.auto-path-dialog h3 {
  margin-top: 0;
  margin-bottom: 16px;
  color: #2c3e50;
}

.import-dialog textarea {
  width: 100%;
  height: 150px;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  margin-bottom: 16px;
  font-family: monospace;
}

.dialog-buttons {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

.auto-path-settings {
  display: flex;
  flex-direction: column;
  gap: 16px;
  margin-bottom: 20px;
}

.setting-group {
  display: flex;
  align-items: center;
  gap: 10px;
}

.setting-group label {
  min-width: 100px;
  font-weight: bold;
}

.setting-group select,
.setting-group input {
  flex: 1;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.setting-group input[type='range'] {
  padding: 0;
}

/* 路径规划样式 */
.path-planning-dialog {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 100;
  width: 80%;
  max-width: 500px;
  max-height: 80vh;
  overflow-y: auto;
}

.radio-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin: 8px 0;
}

.waypoints-container {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-top: 10px;
  max-height: 200px;
  overflow-y: auto;
  border: 1px solid #eee;
  padding: 10px;
  border-radius: 4px;
}

.waypoint-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

.waypoint-item select {
  flex-grow: 1;
}

.add-button {
  margin-top: 8px;
  align-self: flex-start;
}

.import-export-section {
  margin-top: 20px;
  padding-top: 15px;
  border-top: 1px solid #eee;
}

.import-export-section textarea {
  width: 100%;
  resize: vertical;
  font-family: monospace;
  font-size: 12px;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.format-info {
  font-size: 12px;
  color: #666;
  margin-top: 4px;
}

.workflow-controls {
  margin-top: 20px;
  padding: 16px;
  background-color: #f8f9fa;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.workflow-info {
  margin-bottom: 16px;
  padding: 10px;
  background-color: #e8f4f8;
  border-left: 4px solid #3498db;
  border-radius: 4px;
}

.workflow-steps {
  margin-top: 16px;
  padding: 16px;
  background-color: #f0f8ff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.step-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 10px;
}

.node-list,
.connection-list {
  margin-top: 16px;
}

.node-item,
.connection-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px;
  margin-bottom: 8px;
  background-color: white;
  border-radius: 4px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  cursor: pointer;
  transition: all 0.2s ease;
}

.connection-item:hover {
  background-color: #f5f5f5;
  transform: translateY(-2px);
  box-shadow: 0 3px 6px rgba(0, 0, 0, 0.15);
}

.connection-selected {
  background-color: #e8f4f8;
  border-left: 4px solid #3498db;
}

.connection-controls {
  margin: 15px 0;
  padding: 10px;
  background-color: #f8f9fa;
  border-radius: 8px;
  border-left: 4px solid #3498db;
}

.delete-connection-button {
  padding: 8px 16px;
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.delete-connection-button:hover {
  background-color: #c0392b;
}

.delete-connection-button:disabled {
  background-color: #95a5a6;
  cursor: not-allowed;
}

.connection-tip {
  margin-top: 10px;
  font-style: italic;
  color: #7f8c8d;
  font-size: 14px;
}

.node-item input {
  flex: 1;
  margin-right: 8px;
  padding: 4px 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.small-button {
  padding: 4px 8px;
  font-size: 12px;
  background-color: #e74c3c;
}

.small-button:hover {
  background-color: #c0392b;
}

.active-mode {
  background-color: #2ecc71;
}

.active-mode:hover {
  background-color: #27ae60;
}

.animation-settings {
  margin-top: 16px;
  padding: 15px;
  background-color: #ffffff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  border-left: 4px solid #1abc9c;
}

.animation-settings h4 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #16a085;
}

.replay-button {
  background-color: #1abc9c;
  margin-left: 10px;
}

.replay-button:hover {
  background-color: #16a085;
}

.loop-counter {
  margin-left: 10px;
  background-color: #3498db;
  color: white;
  padding: 2px 8px;
  border-radius: 10px;
  font-size: 12px;
  font-weight: bold;
  animation: pulse 1s infinite alternate;
}

@keyframes pulse {
  from {
    opacity: 0.8;
  }
  to {
    opacity: 1;
    transform: scale(1.05);
  }
}

.dialog-buttons button:last-child {
  background-color: #95a5a6;
}

.dialog-buttons button:last-child:hover {
  background-color: #7f8c8d;
}

.data-controls button:first-child {
  background-color: #27ae60;
}

.data-controls button:first-child:hover {
  background-color: #219653;
}

.data-controls button:last-child {
  background-color: #f39c12;
}

.data-controls button:last-child:hover {
  background-color: #d35400;
}
</style>
