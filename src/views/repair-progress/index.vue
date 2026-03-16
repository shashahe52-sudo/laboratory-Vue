<template>
  <div class="repair-progress-page">
    <!-- 统计卡片 -->
    <a-row :gutter="24" class="stats-section">
      <a-col :xs="12" :sm="12" :md="6">
        <a-card class="stat-card">
          <a-statistic title="待处理" :value="progressStats.waiting" :value-style="{ color: '#faad14' }">
            <template #prefix>
              <ClockCircleOutlined />
            </template>
          </a-statistic>
        </a-card>
      </a-col>
      <a-col :xs="12" :sm="12" :md="6">
        <a-card class="stat-card">
          <a-statistic title="维修中" :value="progressStats.repairing" :value-style="{ color: '#1890ff' }">
            <template #prefix>
              <ToolOutlined />
            </template>
          </a-statistic>
        </a-card>
      </a-col>
      <a-col :xs="12" :sm="12" :md="6">
        <a-card class="stat-card">
          <a-statistic title="已完成" :value="progressStats.completed" :value-style="{ color: '#52c41a' }">
            <template #prefix>
              <CheckCircleOutlined />
            </template>
          </a-statistic>
        </a-card>
      </a-col>
      <a-col :xs="12" :sm="12" :md="6">
        <a-card class="stat-card">
          <a-statistic title="无需维护" :value="progressStats.noNeed" :value-style="{ color: '#722ed1' }">
            <template #prefix>
              <CloseCircleOutlined />
            </template>
          </a-statistic>
        </a-card>
      </a-col>
    </a-row>

    <!-- 搜索和筛选 -->
    <a-card class="search-card">
      <div class="search-filters">
        <a-input v-model:value="searchText" placeholder="搜索设备名称或描述" allow-clear @change="handleSearch" class="search-input">
          <template #prefix>
            <SearchOutlined />
          </template>
        </a-input>

        <a-select v-model:value="statusFilter" placeholder="状态筛选" allow-clear @change="handleFilter" class="search-input">
          <a-select-option value="0">等待维修</a-select-option>
          <a-select-option value="1">维修中</a-select-option>
          <a-select-option value="2">完成维修</a-select-option>
          <a-select-option value="3">无需维护</a-select-option>
        </a-select>

        <a-button type="primary" @click="handleSearch" :loading="loading" class="btn">
          <SearchOutlined />
          搜索
        </a-button>
        <a-button type="primary" @click="handleRefresh" :loading="loading" class="btn">
          <ReloadOutlined />
          刷新
        </a-button>
      </div>
    </a-card>

    <!-- 维修进度列表 -->
    <div class="repair-list-card">
      <a-table :columns="columns" :data-source="filteredRepairs" :scroll="{ x: 800 }" :loading="loading" :pagination="pagination" row-key="id" @change="handleTableChange" size="small">
        <template #bodyCell="{ column, record }">
          <template v-if="column.key === 'status'">
            <a-tag :color="getStatusColor(record.status)">
              {{ getStatusText(record.status) }}
            </a-tag>
          </template>

          <template v-else-if="column.key === 'issueType'">
            {{ getIssueTypeText(record.issueType) }}
          </template>

          <template v-else-if="column.key === 'actions'">
            <a-dropdown>
              <a-button type="link" size="small">
                操作
                <DownOutlined />
              </a-button>
              <template #overlay>
                <a-menu>
                  <a-menu-item key="detail" @click="viewDetail(record)">
                    <EyeOutlined />
                    详情
                  </a-menu-item>
                  <a-menu-item key="delete" @click="deleteRepair(record)" v-if="record.status === 0">
                    <DeleteOutlined />
                    删除
                  </a-menu-item>
                </a-menu>
              </template>
            </a-dropdown>
          </template>
        </template>
      </a-table>
    </div>

    <!-- 维修详情模态框 -->
    <a-modal v-model:open="detailModalVisible" title="报修详情" width="800px" :footer="null">
      <div v-if="selectedRepair" class="repair-detail">
        <a-descriptions :column="2" bordered>
          <a-descriptions-item label="报修单号">{{ selectedRepair.recordNo }}</a-descriptions-item>
          <a-descriptions-item label="问题类型">{{ getIssueTypeText(selectedRepair.issueType) }}</a-descriptions-item>
          <a-descriptions-item label="用户名">{{ selectedRepair.reporterName }}</a-descriptions-item>
          <a-descriptions-item label="报修时间">{{ selectedRepair.createTime }}</a-descriptions-item>
          <a-descriptions-item label="状态">
            <a-tag :color="getStatusColor(selectedRepair.status)">
              {{ getStatusText(selectedRepair.status) }}
            </a-tag>
          </a-descriptions-item>
          <a-descriptions-item label="实验室">{{ selectedRepair.labName }}</a-descriptions-item>
          <a-descriptions-item label="设备名称">{{ selectedRepair.equipmentName }}</a-descriptions-item>
          <a-descriptions-item label="故障描述" :span="2">
            {{ selectedRepair.description }}
          </a-descriptions-item>
        </a-descriptions>

        <!-- 维修记录 -->
        <div v-if="selectedRepair.repairRecords && selectedRepair.repairRecords.length > 0" class="repair-records">
          <h4>维修记录</h4>
          <a-timeline>
            <a-timeline-item v-for="record in selectedRepair.repairRecords" :key="record.id">
              <template #dot>
                <ClockCircleOutlined style="font-size: 16px" />
              </template>
              <div class="timeline-content">
                <div class="timeline-title">{{ record.action }}</div>
                <div class="timeline-desc">{{ record.description }}</div>
                <div class="timeline-meta">
                  <span>{{ record.operator }}</span>
                  <span>{{ record.createTime }}</span>
                </div>
              </div>
            </a-timeline-item>
          </a-timeline>
        </div>
      </div>
    </a-modal>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'
import { message } from 'ant-design-vue'
import { SearchOutlined, ReloadOutlined, ClockCircleOutlined, ToolOutlined, CheckCircleOutlined, CloseCircleOutlined, DownOutlined, EyeOutlined, DeleteOutlined } from '@ant-design/icons-vue'
import { getMyMaintenanceDetail, deleteMyMaintenance, getMyMaintenanceProgress, getMyRepairList } from '@/api/maintenance'

// 响应式数据
const loading = ref(false)
const searchText = ref('')
const statusFilter = ref(undefined)
const detailModalVisible = ref(false)
const selectedRepair = ref(null)

// 统计数据
const progressStats = ref({
  waiting: 0,
  repairing: 0,
  completed: 0,
  noNeed: 0
})

// 分页配置
const pagination = reactive({
  current: 1,
  pageSize: 10,
  total: 0,
  showSizeChanger: true,
  showQuickJumper: true,
  showTotal: (total, range) => `第 ${range[0]}-${range[1]} 条，共 ${total} 条`
})

// 维修数据
const repairs = ref([])

// 表格列配置
const columns = [
  {
    title: '报修单号',
    dataIndex: 'recordNo',
    key: 'recordNo',
    width: 140
  },
  {
    title: '实验室',
    dataIndex: 'labName',
    key: 'labName',
    sorter: true
  },
  {
    title: '设备名称',
    dataIndex: 'equipmentName',
    key: 'equipmentName'
  },
  {
    title: '问题类型',
    dataIndex: 'issueType',
    key: 'issueType',
    width: 100
  },
  {
    title: '故障描述',
    dataIndex: 'description',
    key: 'description',
    ellipsis: true
  },
  {
    title: '状态',
    dataIndex: 'status',
    key: 'status',
    width: 100
  },
  {
    title: '报修时间',
    dataIndex: 'createTime',
    key: 'createTime',
    width: 160,
    sorter: true
  },
  {
    title: '操作',
    key: 'actions',
    width: 80,
    fixed: 'right'
  }
]

// 计算属性
const filteredRepairs = computed(() => {
  let result = repairs.value

  // 搜索过滤
  if (searchText.value) {
    result = result.filter(
      (repair) => repair.equipmentName.toLowerCase().includes(searchText.value.toLowerCase()) || repair.description.toLowerCase().includes(searchText.value.toLowerCase()) || repair.recordNo.toLowerCase().includes(searchText.value.toLowerCase())
    )
  }

  // 状态过滤
  if (statusFilter.value !== undefined) {
    result = result.filter((repair) => repair.status === parseInt(statusFilter.value))
  }

  return result
})

// 方法
const getStatusColor = (status) => {
  const colorMap = {
    0: 'orange', // 等待维修
    1: 'blue', // 维修中
    2: 'green', // 完成维修
    3: 'purple' // 无需维护
  }
  return colorMap[status] || 'default'
}

const getStatusText = (status) => {
  const textMap = {
    0: '等待维修',
    1: '维修中',
    2: '完成维修',
    3: '无需维护'
  }
  return textMap[status] || '未知'
}

const getIssueTypeText = (type) => {
  const map = {
    clean: '清洁',
    repair: '维修',
    accident: '事故',
    other: '其他'
  }
  return map[type] || '未知'
}

const handleFilter = () => {
  // 筛选逻辑已在计算属性中处理
}

const handleSearch = () => {
  // 搜索逻辑已在计算属性中处理
}

const loadProgress = () => {
  return getMyMaintenanceProgress()
    .then((data) => {
      progressStats.value.waiting = data?.waiting ?? 0
      progressStats.value.repairing = data?.repairing ?? 0
      progressStats.value.completed = data?.completed ?? 0
      progressStats.value.noNeed = data?.noNeed ?? 0
    })
    .catch(() => {
      message.error('加载进度统计失败')
    })
}

const loadRepairs = () => {
  loading.value = true

  return getMyRepairList(pagination.current, pagination.pageSize)
    .then((data) => {
      const list = data?.rows ?? data?.items ?? data?.records ?? data?.list ?? (Array.isArray(data) ? data : [])
      const total = data?.total ?? data?.totalCount ?? list.length

      repairs.value = list.map((item) => mapMaintenanceRecord(item))
      pagination.total = total
    })
    .catch(() => {
      message.error('加载报修列表失败')
    })
    .finally(() => {
      loading.value = false
    })
}

const mapMaintenanceRecord = (item) => {
  return {
    ...item,
    labName: item.labName || '',
    equipmentName: item.equipmentName || '',
    issueType: item.issueType || '',
    description: item.description || '',
    reporterName: item.reporterName || '',
    createTime: item.createTime || '',
    repairRecords: Array.isArray(item.repairRecords) ? item.repairRecords : []
  }
}

const handleRefresh = () => {
  return Promise.all([loadProgress(), loadRepairs()]).then(() => {
    message.success('数据刷新成功')
  })
}

const handleTableChange = (pag, filters, sorter) => {
  pagination.current = pag.current
  pagination.pageSize = pag.pageSize
  loadRepairs()
}

const viewDetail = (record) => {
  if (!record.id) return

  loading.value = true
  getMyMaintenanceDetail(record.id)
    .then((data) => {
      selectedRepair.value = mapMaintenanceRecord(data)
      detailModalVisible.value = true
    })
    .catch(() => {
      message.error('获取报修详情失败')
    })
    .finally(() => {
      loading.value = false
    })
}

const deleteRepair = (record) => {
  if (!record.id) return

  // 确认删除
  if (record.status !== 0) {
    message.warning('只有待审核状态的报修才能删除')
    return
  }

  loading.value = true
  deleteMyMaintenance(record.id)
    .then(() => {
      message.success('报修删除成功')
      return Promise.all([loadProgress(), loadRepairs()])
    })
    .catch(() => {
      message.error('删除报修失败')
    })
    .finally(() => {
      loading.value = false
    })
}

// 生命周期
onMounted(() => {
  Promise.all([loadProgress(), loadRepairs()])
})
</script>

<style scoped lang="scss">
.stats-section {
  margin-bottom: 16px;
}

.stat-card {
  text-align: center;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.search-card {
  gap: 10px;
  margin-bottom: 16px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.search-filters {
  display: flex;
  align-items: center;
  gap: 10px;
  @media (max-width: 768px) {
    flex-direction: column;
  }
}

.search-input {
  width: 240px;
  min-width: 200px;
}

.repair-list-card {
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.repair-detail {
  padding: 16px 0;
}

.repair-records {
  margin-top: 24px;

  h4 {
    margin-bottom: 16px;
    color: #333;
  }
}

.timeline-content {
  margin-left: 8px;
}

.timeline-title {
  font-weight: 500;
  color: #333;
  margin-bottom: 4px;
}

.timeline-desc {
  color: #666;
  margin-bottom: 8px;
  line-height: 1.5;
}

.timeline-meta {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: #999;
}

/* 移动端适配 */
@media (max-width: 768px) {
  .stats-section {
    margin-bottom: 16px;
  }

  .stat-card {
    margin-bottom: 16px;
  }

  .search-input {
    width: 100%;
    min-width: auto;
  }

  .repair-list-card :deep(.ant-table-thead > tr > th) {
    padding: 8px 4px;
    font-size: 12px;
  }

  .repair-list-card :deep(.ant-table-tbody > tr > td) {
    font-size: 12px;
  }

  .repair-list-card :deep(.ant-table-tbody > tr > td .ant-btn) {
    padding: 2px 4px;
  }
}
</style>
