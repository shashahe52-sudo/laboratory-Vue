<template>
  <div class="home-page">
    <!-- 欢迎横幅 -->
    <a-row :gutter="24" class="welcome-section">
      <a-col :span="24">
        <a-card class="welcome-card">
          <div class="welcome-content">
            <div class="welcome-text">
              <a-typography-title :level="2" class="welcome-title"> 欢迎使用实验室管理系统 </a-typography-title>
              <a-typography-paragraph class="welcome-desc"> 高效管理实验室资源，简化申请流程，提升使用体验 </a-typography-paragraph>
            </div>
            <div class="welcome-actions">
              <a-button type="primary" size="large" @click="goToApplication"> 立即申请 </a-button>
              <a-button size="large" @click="goToRepair"> 设备报修 </a-button>
            </div>
          </div>
        </a-card>
      </a-col>
    </a-row>

    <!-- 统计卡片 -->
    <a-row :gutter="24" class="stats-section">
      <a-col :xs="12" :sm="12" :md="6">
        <a-card class="stat-card">
          <a-statistic title="可用实验室" :value="stats.availableLabs" :value-style="{ color: '#3f8600' }">
            <template #prefix>
              <ExperimentOutlined />
            </template>
          </a-statistic>
        </a-card>
      </a-col>
      <a-col :xs="12" :sm="12" :md="6">
        <a-card class="stat-card">
          <a-statistic title="使用中实验室" :value="stats.inUseLabs" :value-style="{ color: '#1890ff' }">
            <template #prefix>
              <ClockCircleOutlined />
            </template>
          </a-statistic>
        </a-card>
      </a-col>
      <a-col :xs="12" :sm="12" :md="6">
        <a-card class="stat-card">
          <a-statistic title="待维修设备" :value="stats.pendingMaintenanceEquipment" :value-style="{ color: '#cf1322' }">
            <template #prefix>
              <ToolOutlined />
            </template>
          </a-statistic>
        </a-card>
      </a-col>
      <a-col :xs="12" :sm="12" :md="6">
        <a-card class="stat-card">
          <a-statistic title="今日申请" :value="stats.todayApplications" :value-style="{ color: '#722ed1' }">
            <template #prefix>
              <FileTextOutlined />
            </template>
          </a-statistic>
        </a-card>
      </a-col>
    </a-row>

    <!-- 快速操作 -->
    <a-row :gutter="24" class="quick-actions-section">
      <a-col :span="24">
        <a-card title="快速操作" class="quick-actions-card">
          <a-row :gutter="16">
            <a-col :xs="12" :sm="12" :md="8" :lg="6" v-for="action in quickActions" :key="action.key">
              <a-card class="action-card" :hoverable="true" @click="handleQuickAction(action.key)">
                <div class="action-content">
                  <div class="action-icon">
                    <component :is="action.icon" />
                  </div>
                  <div class="action-text">
                    <div class="action-title">{{ action.title }}</div>
                    <div class="action-desc">{{ action.description }}</div>
                  </div>
                </div>
              </a-card>
            </a-col>
          </a-row>
        </a-card>
      </a-col>
    </a-row>

    <!-- 最近活动 -->
    <a-row :gutter="24" class="recent-activities-section">
      <a-col :xs="24" :lg="12">
        <a-card title="实验室申请" class="activities-card">
          <a-list :data-source="recentApplications" :loading="loading">
            <template #renderItem="{ item }">
              <a-list-item>
                <a-list-item-meta>
                  <template #title>
                    <a>{{ item.labName }}</a>
                  </template>
                  <template #description>
                    <div class="activity-meta">
                      <!-- <span class="applicant">申请人：{{ item.applicant }}</span> -->
                      <a-tag :color="getStatusColor(item.status)">{{ getStatusText(item.status) }}</a-tag>
                      <span class="activity-time">{{ item.createTime }}</span>
                    </div>
                  </template>
                </a-list-item-meta>
              </a-list-item>
            </template>
          </a-list>
        </a-card>
      </a-col>
      <a-col :xs="24" :lg="12">
        <a-card title="系统公告" class="activities-card">
          <a-list :data-source="notices" :loading="loading">
            <template #renderItem="{ item }">
              <a-list-item>
                <a-list-item-meta>
                  <template #title>
                    <a @click="viewNotice(item.id)">{{ item.title }}</a>
                  </template>
                  <template #description>
                    <div class="activity-meta">
                      <span class="activity-time">{{ formatNoticeTime(item.publishTime || item.createTime) }}</span>
                    </div>
                  </template>
                </a-list-item-meta>
              </a-list-item>
            </template>
          </a-list>
        </a-card>
      </a-col>
    </a-row>
  </div>
</template>

<script setup>
import { ExperimentOutlined, ClockCircleOutlined, ToolOutlined, FileTextOutlined, SettingOutlined, TeamOutlined, UserOutlined, HistoryOutlined } from '@ant-design/icons-vue'
import { useLabStore } from '@/stores/lab'
import { useRepairStore } from '@/stores/repair'
import { useUserStore } from '@/stores/user'
import { listByPublished } from '@/api/document'
import { getLaboratorySummary } from '@/api/laboratory'
import { listTodayApplications } from '@/api/labApplication'

const router = useRouter()
const labStore = useLabStore()
const repairStore = useRepairStore()
const userStore = useUserStore()

// 响应式数据
const loading = ref(false)

const summary = ref({
  availableLabs: 0,
  inUseLabs: 0,
  pendingMaintenanceEquipment: 0,
  todayApplications: 0
})

// 统计数据
const stats = computed(() => ({
  availableLabs: summary.value.availableLabs,
  inUseLabs: summary.value.inUseLabs,
  pendingMaintenanceEquipment: summary.value.pendingMaintenanceEquipment,
  todayApplications: summary.value.todayApplications
}))

// 快速操作配置
const quickActions = computed(() => {
  const baseActions = [
    {
      key: 'apply-lab',
      title: '申请实验室',
      description: '申请使用实验室',
      icon: ExperimentOutlined
    },
    {
      key: 'repair-report',
      title: '设备报修',
      description: '报告设备故障',
      icon: ToolOutlined
    },
    {
      key: 'repair-progress',
      title: '报修进度',
      description: '查看报修进度',
      icon: HistoryOutlined
    },
    {
      key: 'application-record',
      title: '申请记录',
      description: '查看申请记录',
      icon: FileTextOutlined
    }
  ]

  if (!userStore.isLoggedIn) {
    return baseActions.filter((item) => item.key !== 'application-record' && item.key !== 'repair-progress')
  }

  return baseActions
})

// 今日实验室申请数据
const todayLabApplications = ref([])

// 最近申请数据（今日实验室申请）
const recentApplications = computed(() => {
  return todayLabApplications.value.map((item) => ({
    id: item.id,
    labName: item.labName || item.laboratory?.labName || '',
    applicant: item.applicant || '',
    status: mapLabApplicationStatus(item.status),
    createTime: item.createTime || '',
    purpose: item.purpose || ''
  }))
})

// 系统公告数据（由接口获取）
const notices = ref([])

// 事件处理
const goToApplication = () => {
  router.push('/lab-application')
}

const goToRepair = () => {
  router.push('/repair-report')
}

const handleQuickAction = (key) => {
  const routeMap = {
    'repair-handling': '/repair-handling',
    'apply-lab': '/lab-application',
    'repair-report': '/repair-report',
    'repair-progress': '/repair-progress',
    'application-record': '/application-record',
    'personal-settings': '/setting',
    'lab-search': '/lab-search',
    'satisfaction-detail': '/satisfaction-detail'
  }

  if (routeMap[key]) {
    router.push(routeMap[key])
  }
}

const viewNotice = (id) => {
  const url = router.resolve({ path: `/notice-detail/${id}` }).href
  window.open(url, '_blank')
}

const getStatusColor = (status) => {
  const colorMap = {
    pending: 'orange', // 待审核
    approved: 'green', // 已通过
    rejected: 'red', // 已拒绝
    in_use: 'blue', // 使用中
    completed: 'green', // 已完成
    cancelled: 'gray' // 已取消
  }
  return colorMap[status] || 'default'
}

const getStatusText = (status) => {
  const textMap = {
    pending: '待审核',
    approved: '已通过',
    rejected: '已拒绝',
    in_use: '使用中',
    completed: '已完成',
    cancelled: '已取消'
  }
  return textMap[status] || '未知状态'
}

const mapLabApplicationStatus = (status) => {
  // Map numeric status to text status
  const statusMap = {
    0: 'pending', // 待审核
    1: 'approved', // 已通过
    2: 'rejected', // 已拒绝
    3: 'in_use', // 使用中
    4: 'completed', // 已完成
    5: 'cancelled' // 已取消
  }
  return statusMap[status] || 'pending'
}

// 生命周期
onMounted(() => {
  // 加载数据
  loadData()
})

const loadData = async () => {
  loading.value = true

  // 并行加载数据
  Promise.all([getLaboratorySummary(), listTodayApplications(), listByPublished(1, 10)])
    .then(([labSummary, labApplicationData, noticeData]) => {
      // 实验室汇总数据
      summary.value.availableLabs = labSummary?.availableLabs ?? 0
      summary.value.inUseLabs = labSummary?.inUseLabs ?? 0
      summary.value.pendingMaintenanceEquipment = labSummary?.pendingMaintenanceEquipment ?? 0
      summary.value.todayApplications = labSummary?.todayApplications ?? 0

      // 今日实验室申请数据
      const labApplicationList = labApplicationData?.items ?? labApplicationData?.records ?? labApplicationData?.list ?? (Array.isArray(labApplicationData) ? labApplicationData : [])
      todayLabApplications.value = labApplicationList.slice(0, 5)

      // 系统公告数据
      const noticeList = noticeData?.items ?? noticeData?.records ?? noticeData?.list ?? []
      notices.value = noticeList
    })
    .catch(() => {
      // 静默处理错误，不显示错误信息
    })
    .finally(() => {
      loading.value = false
    })
}

const formatNoticeTime = (val) => {
  if (!val) return ''
  const s = String(val)
  if (s.length >= 16) return s.slice(0, 16).replace('T', ' ')
  return s
}
</script>

<style scoped>
@import './index.scss';
</style>
