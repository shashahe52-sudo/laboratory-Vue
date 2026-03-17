<template>
  <div class="lab-application-page">
    <a-card title="申请信息" class="application-form-card">
      <a-form ref="formRef" :model="formData" :rules="formRules" @finish="handleSubmit">
        <a-form-item label="申请类型" name="applicationType">
          <a-radio-group v-model:value="formData.applicationType" @change="handleApplicationTypeChange">
            <a-radio value="lab">申请实验室</a-radio>
            <a-radio value="equipment">申请设备</a-radio>
          </a-radio-group>
        </a-form-item>

        <template v-if="formData.applicationType === 'lab'">
          <a-form-item label="使用类型" name="usageType">
            <a-radio-group v-model:value="formData.usageType">
              <a-radio value="personal">个人使用</a-radio>
              <a-radio value="course">课程使用</a-radio>
            </a-radio-group>
          </a-form-item>
        </template>

        <template v-if="formData.usageType === 'course' && formData.applicationType==='lab' ">
          <a-form-item  label="课程名称" name="courseName">
          <a-input v-model:value="formData.courseName" placeholder="请输入课程名称" />
        </a-form-item>

        <a-form-item label="班级" name="className">
          <a-input v-model:value="formData.className" placeholder="请输入班级" />
        </a-form-item>
        </template>
        

        <!-- 申请实验室时的表单 -->
        <template v-if="formData.applicationType === 'lab'">
          <a-form-item label="选择实验室" name="labId">
            <a-select v-model:value="formData.labId" placeholder="请选择要申请的实验室" show-search :filter-option="filterOption" @change="handleLabChange" :loading="labsLoading">
              <a-select-option v-for="lab in availableLabs" :key="lab.id" :value="lab.id" :label="`${lab.labName || ''} ${lab.location || ''}`" :disabled="lab.status !== 1">
                {{ lab.labName }} - {{ lab.location }}
                <a-tag :color="getLabStatusColor(lab.status)" size="small">
                  {{ getLabStatusText(lab.status) }}
                </a-tag>
              </a-select-option>
            </a-select>
          </a-form-item>
        </template>

        <!-- 申请设备时的表单 -->
        <template v-if="formData.applicationType === 'equipment'">
          <EquipmentApplicationForm
            :form-data="formData"
            :available-labs="availableLabs"
            :labs-loading="labsLoading"
            :available-equipment="availableEquipment"
            :equipment-loading="equipmentLoading"
            :filter-option="filterOption"
            :filter-equipment-option="filterEquipmentOption"
            :on-lab-change="handleLabChange"
            :on-equipment-change="handleEquipmentChange"
            :get-lab-status-color="getLabStatusColor"
            :get-lab-status-text="getLabStatusText"
            :get-equipment-status-color="getEquipmentStatusColor"
            :get-equipment-status-text="getEquipmentStatusText"
            :disabled-date="disabledDate"
            :disabled-time="disabledTime"
          />
        </template>

        <a-form-item v-if="formData.applicationType === 'lab'" label="使用目的" name="purpose">
          <a-textarea v-model:value="formData.purpose" placeholder="请详细描述使用实验室的目的和内容" :rows="3" />
        </a-form-item>

        <!-- 申请实验室：使用时间从后端可申请时间段中选择 -->
        <a-form-item v-if="formData.applicationType === 'lab'" label="使用时间" name="scheduleId">
          <a-select v-model:value="formData.scheduleId" placeholder="请先选择实验室，再选择可申请时间段" :loading="scheduleLoading" :disabled="!formData.labId" @change="onScheduleChange">
            <a-select-option v-for="s in labScheduleList" :key="s.id" :value="s.id"> {{ formatScheduleTime(s.startTime) }} 至 {{ formatScheduleTime(s.endTime) }} </a-select-option>
          </a-select>
        </a-form-item>

        <a-form-item v-if="formData.applicationType === 'lab'" label="预计人数" name="participantCount">
          <a-input-number v-model:value="formData.participantCount" :min="1" :max="selectedLab?.capacity || 50" />
        </a-form-item>

        <!-- 申请实验室时：选中后调用接口展示该实验室设备 -->
        <a-form-item v-if="formData.applicationType === 'lab' && formData.labId" label="实验室具有的设备">
          <div class="lab-equipment-readonly">
            <template v-if="equipmentLoading">加载中...</template>
            <template v-else-if="labEquipmentList && labEquipmentList.length">
              <a-tag v-for="eq in labEquipmentList" :key="eq.id" color="blue">
                {{ eq.equipmentName || eq.equipment_name }}{{ eq.model ? `（${eq.model}）` : '' }}
                <span v-if="eq.availableQuantity != null" class="eq-qty">可用 {{ eq.availableQuantity }} 台</span>
              </a-tag>
            </template>
            <span v-else class="no-equipment">暂无设备信息</span>
          </div>
        </a-form-item>

        <a-form-item label="特殊要求" name="specialRequirements">
          <a-textarea v-model:value="formData.specialRequirements" placeholder="如有特殊要求请在此说明" :rows="2" />
        </a-form-item>

        <a-form-item label="联系方式" name="contactInfo">
          <a-input v-model:value="formData.contactInfo" placeholder="请输入联系电话" />
        </a-form-item>

        <a-space>
          <a-button type="primary" html-type="submit" :loading="submitting"> 提交申请 </a-button>
          <a-button @click="handleReset">重置</a-button>
          <a-button @click="handlePreview">预览申请</a-button>
        </a-space>
      </a-form>
    </a-card>

    <!-- 实验室信息卡片 -->
    <a-card v-if="formData.applicationType === 'lab' && selectedLab" title="实验室信息" class="lab-info-card">
      <div class="lab-info">
        <a-descriptions :column="1" size="small">
          <a-descriptions-item label="实验室名称">
            {{ selectedLab.labName }}
          </a-descriptions-item>
          <a-descriptions-item label="实验室编号">
            {{ selectedLab.labNumber }}
          </a-descriptions-item>
          <a-descriptions-item label="位置">
            {{ selectedLab.location }}
          </a-descriptions-item>
          <a-descriptions-item label="容量"> {{ selectedLab.capacity }} 人 </a-descriptions-item>
          <a-descriptions-item label="面积"> {{ selectedLab.area }} m² </a-descriptions-item>
          <a-descriptions-item label="当前状态">
            <a-tag :color="getLabStatusColor(selectedLab.status)">
              {{ getLabStatusText(selectedLab.status) }}
            </a-tag>
          </a-descriptions-item>
        </a-descriptions>

        <a-divider />

        <div class="equipment-list">
          <h4>实验室设备</h4>
          <template v-if="equipmentLoading">
            <a-spin size="small" />
          </template>
          <a-list v-else :data-source="labEquipmentList" size="small" :pagination="false">
            <template #renderItem="{ item }">
              <a-list-item>
                <a-list-item-meta>
                  <template #title>{{ item.equipmentName || item.equipment_name }}</template>
                  <template #description>
                    {{ item.model ? `型号：${item.model}` : '' }}
                    {{ item.manufacturer ? ` 厂商：${item.manufacturer}` : '' }}
                    <span v-if="item.availableQuantity != null"> 可用 {{ item.availableQuantity }} 台</span>
                  </template>
                </a-list-item-meta>
              </a-list-item>
            </template>
          </a-list>
          <a-empty v-if="!equipmentLoading && (!labEquipmentList || !labEquipmentList.length)" description="暂无设备" />
        </div>
      </div>
    </a-card>

    <!-- 设备信息卡片 -->
    <a-card v-if="formData.applicationType === 'equipment' && selectedEquipment" title="申请信息" class="equipment-info-card">
      <div class="equipment-info">
        <a-descriptions :column="2" size="small">
            <a-descriptions-item label="所在实验室">
            {{ selectedLab?.labName }}
          </a-descriptions-item>
          <a-descriptions-item label="设备名称">
            {{ selectedEquipment.equipmentName }}
          </a-descriptions-item>
          <a-descriptions-item label="可用数量"> {{ selectedEquipment.count }} 台 </a-descriptions-item>
          <a-descriptions-item label="借用数量">
           {{ formData.participantCount }}
          </a-descriptions-item>
          <a-descriptions-item label="使用目的">
            {{formData.purpose }}
          </a-descriptions-item>
           <a-descriptions-item label="特殊要求">
            {{ formData.specialRequirements || '无' }}
          </a-descriptions-item>
          <a-descriptions-item label="联系方式">
            {{ formData.contactInfo }}
          </a-descriptions-item>
          <a-descriptions-item label="使用时间">
            {{ formData.timeRange?.[0]?.format('YYYY-MM-DD HH:mm') }} 至 {{formData.timeRange?.[1]?.format('YYYY-MM-DD HH:mm') }}
          </a-descriptions-item>
        </a-descriptions>
      </div>
    </a-card>

    <!-- 申请须知：仅展示标题、时间、类型，点击跳转详情 -->
    <a-card title="申请须知" class="notice-card">
      <a-list size="small" :data-source="protocolDocs" :loading="protocolLoading">
        <template #renderItem="{ item }">
          <a-list-item>
            <a-list-item-meta>
              <template #title>
                <a @click="viewDocument(item.id)">{{ item.title }}</a>
              </template>
              <template #description>
                <a-tag :color="getDocTypeColor(item.docType || item.doc_type)" size="small">{{ getDocTypeText(item.docType || item.doc_type) }}</a-tag>
                <span class="doc-time">{{ formatDocTime(item.createTime) }}</span>
              </template>
            </a-list-item-meta>
          </a-list-item>
        </template>
      </a-list>
    </a-card>

    <!-- 申请预览模态框 -->
    <a-modal v-model:open="previewVisible" title="申请预览" width="800px" :footer="null">
      <div class="preview-content" v-if="(formData.applicationType === 'lab' && formData.labId) || (formData.applicationType === 'equipment' && formData.equipmentId)">
        <a-descriptions :column="2" bordered>
          <a-descriptions-item label="申请类型">
            {{ formData.applicationType === 'lab' ? '申请实验室' : '申请设备' }}
          </a-descriptions-item>
          <a-descriptions-item label="使用类型">
            {{ formData.usageType === 'personal' ? '个人使用' : '课程使用' }}
          </a-descriptions-item>
          <a-descriptions-item v-if="formData.applicationType === 'lab'" label="实验室"> {{ selectedLab?.labName }}（{{ selectedLab?.labNumber }}） - {{ selectedLab?.location }} </a-descriptions-item>
          <a-descriptions-item v-if="formData.usageType === 'course'" label="课程名称">
            {{ formData.courseName }}
          </a-descriptions-item>
          <a-descriptions-item v-if="formData.usageType === 'course'" label="班级">
            {{ formData.className }}
          </a-descriptions-item>
          <a-descriptions-item v-if="formData.applicationType === 'equipment'" label="设备"> {{ selectedEquipment?.name }} ({{ selectedEquipment?.model }}) </a-descriptions-item>
          <a-descriptions-item label="使用目的" :span="2">
            {{ formData.purpose }}
          </a-descriptions-item>
          <a-descriptions-item label="使用时间">
            {{ formData.applicationType === 'lab' ? formatSelectedScheduleTime() : formatTimeRange(formData.timeRange) }}
          </a-descriptions-item>
          <a-descriptions-item :label="formData.applicationType === 'lab' ? '预计人数' : '借用数量'"> {{ formData.participantCount }} {{ formData.applicationType === 'lab' ? '人' : '台' }} </a-descriptions-item>
          <a-descriptions-item v-if="formData.applicationType === 'lab' && labEquipmentList?.length" label="实验室具有的设备" :span="2">
            <a-tag v-for="eq in labEquipmentList" :key="eq.id" size="small" color="blue"> {{ eq.equipmentName || eq.equipment_name }}{{ eq.model ? `（${eq.model}）` : '' }} </a-tag>
          </a-descriptions-item>
          <a-descriptions-item label="特殊要求" :span="2">
            {{ formData.specialRequirements || '无' }}
          </a-descriptions-item>
          <a-descriptions-item label="联系方式">
            {{ formData.contactInfo }}
          </a-descriptions-item>
        </a-descriptions>
      </div>
    </a-modal>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { message } from 'ant-design-vue'
import dayjs from 'dayjs'
import EquipmentApplicationForm from './components/EquipmentApplicationForm.vue'
import { listDocumentByPublished } from '@/api/document'
import { listAvailable, getEquipmentByLaboratoryId } from '@/api/laboratory'
import { listByLaboratoryId } from '@/api/labSchedule'
import { apply as submitLabApplication } from '@/api/labApplication'
import { apply as submitEquipmentApplication } from '@/api/equipmentApplication'

const router = useRouter()

// 响应式数据
const formRef = ref()
const submitting = ref(false)
const previewVisible = ref(false)

// 表单数据
const formData = reactive({
  applicationType: 'lab',
  usageType: 'personal',
  labId: undefined,
  equipmentId: undefined,
  purpose: '',
  timeRange: [],
  scheduleId: undefined, // 申请实验室时选中的时间段 ID（来自后端可申请时间段）
  participantCount: 1,
  specialRequirements: '',
  contactInfo: '',
  courseName: '',
  className: ''
})

// 表单验证规则
const formRules = {
  applicationType: [{ required: true, message: '请选择申请类型', trigger: 'change' }],
  usageType: [{ required: true, message: '请选择使用类型', trigger: 'change' }],
  labId: [{ required: true, message: '请选择实验室', trigger: 'change' }],
  equipmentId: [
    {
      validator: (_, value) => {
        if (formData.applicationType === 'equipment' && !value) {
          return Promise.reject('请选择设备')
        }
        return Promise.resolve()
      },
      trigger: 'change'
    }
  ],
  purpose: [{ required: true, message: '请填写使用目的', trigger: 'blur' }],
  courseName: [
    {
      validator: (_, value) => {
        if (formData.usageType === 'course' && !value) {
          return Promise.reject('请输入课程名称')
        }
        return Promise.resolve()
      },
      trigger: 'blur'
    }
  ],
  className: [
    {
      validator: (_, value) => {
        if (formData.usageType === 'course' && !value) {
          return Promise.reject('请输入班级')
        }
        return Promise.resolve()
      },
      trigger: 'blur'
    }
  ],
  // 申请实验室：校验是否选择了可申请时间段
  scheduleId: [
    {
      validator: (_, value) => {
        if (formData.applicationType === 'lab') {
          if (!value) {
            return Promise.reject('请选择可申请时间段')
          }
        }
        return Promise.resolve()
      },
      trigger: 'change'
    }
  ],
  // 申请设备：仍使用时间范围选择
  timeRange: [
    {
      validator: (_, value) => {
        if (formData.applicationType === 'equipment') {
          if (!value || value.length !== 2) {
            return Promise.reject('请选择使用时间')
          }
        }
        return Promise.resolve()
      },
      trigger: 'change'
    }
  ],
  participantCount: [{ required: true, message: '请填写数量', trigger: 'blur' }],
  contactInfo: [{ required: true, message: '请填写联系方式', trigger: 'blur' }]
}

// 可用实验室数据（由接口 /laboratory/user/available 拉取，不分页）
const availableLabs = ref([])
const labsLoading = ref(false)
// 当前选中实验室的设备列表（选中实验室后调用 /laboratory/:id/equipment）
const labEquipmentList = ref([])
const equipmentLoading = ref(false)
// 当前选中实验室的可申请时间段列表（/labSchedule/laboratory/{laboratoryId}）
const labScheduleList = ref([])
const scheduleLoading = ref(false)

// 可用设备数据（选择实验室后从后端加载）
const availableEquipment = ref([])

// 申请须知：文档列表（协议/手册/规则等），仅展示标题、时间、类型，点击跳转详情
const protocolDocs = ref([])
const protocolLoading = ref(false)

const formatDocTime = (val) => {
  if (!val) return ''
  const s = String(val)
  if (s.length >= 16) return s.slice(0, 16).replace('T', ' ')
  return s
}

const getDocTypeText = (type) => {
  const map = { protocol: '协议', manual: '手册', rule: '规则', other: '其他' }
  return map[type] || '文档'
}

const getDocTypeColor = (type) => {
  const map = { protocol: 'blue', manual: 'green', rule: 'orange', other: 'default' }
  return map[type] || 'default'
}

const viewDocument = (id) => {
  const url = router.resolve({ path: `/document-detail/${id}` }).href
  window.open(url, '_blank')
}

// 计算属性：当前选中的实验室（接口字段：labName, labNumber, location, capacity, area, status）
const selectedLab = computed(() => {
  const id = formData.labId
  if (id == null) return null
  return availableLabs.value.find((l) => l.id === id)
})

const selectedEquipment = computed(() => {
  return availableEquipment.value.find((equipment) => equipment.id === formData.equipmentId)
})

// 方法
const filterOption = (input, option) => {
  const text = (option.label ?? option.children ?? '').toString().toLowerCase()
  return text.indexOf((input || '').toLowerCase()) >= 0
}

const filterEquipmentOption = (input, option) => {
  const text = (option.label ?? '').toString().toLowerCase()
  return text.indexOf((input || '').toLowerCase()) >= 0
}

const handleApplicationTypeChange = () => {
  formData.labId = undefined
  formData.equipmentId = undefined
  formData.scheduleId = undefined
  labEquipmentList.value = []
  labScheduleList.value = []
}

const handleLabChange = () => {
  const id = formData.labId
  if (!id) return

  if (formData.applicationType === 'equipment') {
    formData.equipmentId = undefined
    availableEquipment.value = []

    equipmentLoading.value = true
    getEquipmentByLaboratoryId(id)
      .then((list) => {
        const arr = Array.isArray(list) ? list : (list?.items ?? list?.records ?? list?.list ?? [])
        availableEquipment.value = arr || []
      })
      .finally(() => {
        equipmentLoading.value = false
      })
    return
  }

  labEquipmentList.value = []
  labScheduleList.value = []
  formData.scheduleId = undefined

  equipmentLoading.value = true
  getEquipmentByLaboratoryId(id)
    .then((list) => {
      const arr = Array.isArray(list) ? list : (list?.items ?? list?.records ?? list?.list ?? [])
      labEquipmentList.value = arr || []
    })
    .finally(() => {
      equipmentLoading.value = false
    })

  scheduleLoading.value = true
  listByLaboratoryId(id)
    .then((list) => {
      const rawList = Array.isArray(list) ? list : (list?.items ?? list?.records ?? list?.list ?? [])
      const flattened = []
      rawList.forEach((rule, ruleIndex) => {
        if (!rule || rule.status !== 1) return
        const startDate = rule.startDate || ''
        let slots = []
        if (typeof rule.timeSlots === 'string') {
          slots = JSON.parse(rule.timeSlots || '[]')
        } else if (Array.isArray(rule.timeSlots)) {
          slots = rule.timeSlots
        }
        slots.forEach((slot, slotIndex) => {
          const start = (slot.start || '').length === 5 ? `${slot.start}:00` : slot.start || '00:00:00'
          const end = (slot.end || '').length === 5 ? `${slot.end}:00` : slot.end || '00:00:00'
          flattened.push({
            id: `${ruleIndex}-${slotIndex}`,
            startTime: ` ${start}`.trim(),
            endTime: ` ${end}`.trim()
          })
        })
      })
      labScheduleList.value = flattened
    })
    .finally(() => {
      scheduleLoading.value = false
    })
}

const handleEquipmentChange = (value) => {
  // 设备选择变化时的处理
}

const disabledDate = (current) => {
  // 禁用过去的日期
  return current && current < dayjs().startOf('day')
}

const disabledTime = (current, type) => {
  if (type === 'start') {
    // 开始时间不能早于当前时间
    const now = dayjs()
    if (current && current.isSame(now, 'day')) {
      return {
        disabledHours: () => Array.from({ length: now.hour() }, (_, i) => i),
        disabledMinutes: () => Array.from({ length: now.minute() + 1 }, (_, i) => i)
      }
    }
  }
  return {}
}

const getLabStatusColor = (status) => {
  const colorMap = {
    1: 'green', // 正常
    0: 'default' // 禁用
  }
  return colorMap[status] ?? 'default'
}

const getLabStatusText = (status) => {
  const textMap = {
    1: '正常',
    0: '禁用'
  }
  return textMap[status] ?? '未知'
}

const getEquipmentStatusColor = (status) => {
  const colorMap = {
    0: 'green', // 正常且空闲
    1: 'blue', // 使用中
    2: 'red' // 异常
  }
  return colorMap[status] || 'default'
}

const getEquipmentStatusText = (status) => {
  const textMap = {
    0: '正常且空闲',
    1: '使用中',
    2: '异常'
  }
  return textMap[status] || '未知'
}

const formatTimeRange = (timeRange) => {
  if (!timeRange || timeRange.length !== 2) return ''
  return `${timeRange[0].format('YYYY-MM-DD HH:mm')} 至 ${timeRange[1].format('YYYY-MM-DD HH:mm')}`
}

const formatScheduleTime = (str) => {
  if (!str) return ''
  const s = String(str).trim()
  if (s.length >= 16) return s.slice(0, 16).replace('T', ' ')
  return s
}

const formatSelectedScheduleTime = () => {
  if (!formData.scheduleId) return ''
  const slot = labScheduleList.value.find((s) => s.id === formData.scheduleId)
  if (!slot) return ''
  return `${formatScheduleTime(slot.startTime)} 至 ${formatScheduleTime(slot.endTime)}`
}

const onScheduleChange = () => {
  // 选中时间段变更，提交时从 labScheduleList 取 startTime/endTime
}

const handleSubmit = async () => {
  const valid = await formRef.value.validate().catch(() => false)
  if (!valid) return

  if (formData.applicationType === 'lab') {
    if (!formData.labId) {
      message.warning('请选择实验室')
      return
    }
    if (!formData.scheduleId) {
      message.warning('请选择使用时间')
      return
    }
    const slot = labScheduleList.value.find((s) => s.id === formData.scheduleId)
    if (!slot) {
      message.warning('请重新选择使用时间')
      return
    }

    submitting.value = true
    const purpose = formData.usageType === 'personal' ? '个人使用' : '课程使用'
    const lab = selectedLab.value || {}
    const payload = {
      laboratoryId: formData.labId,
      purpose,
      studentCount: formData.participantCount,
      startTime: slot.startTime,
      endTime: slot.endTime,
      applicantPhone: formData.contactInfo,
      labNumber: lab.labNumber,
      labName: lab.labName,
      location: lab.location
    }
    if (formData.usageType === 'course') {
      payload.courseName = formData.courseName
      payload.className = formData.className
    }
    submitLabApplication(payload)
      .then(() => {
        message.success('申请提交成功！')
      })
      .finally(() => {
        submitting.value = false
      })
    return
  }

  if (formData.applicationType === 'equipment') {
    if (!formData.labId) {
      message.warning('请选择实验室')
      return
    }
    if (!formData.equipmentId) {
      message.warning('请选择设备')
      return
    }
    if (!formData.timeRange || formData.timeRange.length !== 2) {
      message.warning('请选择使用时间')
      return
    }

    submitting.value = true
    const startTime = formData.timeRange[0].format('YYYY-MM-DD HH:mm:ss')
    const endTime = formData.timeRange[1].format('YYYY-MM-DD HH:mm:ss')
    const payload = {
      equipmentId: formData.equipmentId,
      laboratoryId: formData.labId,
      quantity: formData.participantCount,
      purpose: formData.purpose,
      startTime,
      endTime,
      applicantPhone: formData.contactInfo
    }

    submitEquipmentApplication(payload)
      .then(() => {
        message.success('申请提交成功！')
      })
      .finally(() => {
        submitting.value = false
      })
    return
  }
}

const handleReset = () => {
  formRef.value.resetFields()
}

const handlePreview = () => {
  if (formData.applicationType === 'lab' && !formData.labId) {
    message.warning('请先选择实验室')
    return
  }
  if (formData.applicationType === 'equipment' && !formData.equipmentId) {
    message.warning('请先选择设备')
    return
  }
  previewVisible.value = true
}

// 生命周期
onMounted(() => {
  loadLabs()
  loadProtocolDocs()
})

const loadLabs = () => {
  labsLoading.value = true
  listAvailable()
    .then((data) => {
      const list = Array.isArray(data) ? data : (data?.items ?? data?.records ?? data?.list ?? [])
      availableLabs.value = list || []
    })
    .finally(() => {
      labsLoading.value = false
    })
}

const loadProtocolDocs = () => {
  protocolLoading.value = true
  listDocumentByPublished(1, 50)
    .then((data) => {
      const list = data?.items ?? data?.records ?? data?.list ?? (Array.isArray(data) ? data : [])
      protocolDocs.value = list || []
    })
    .finally(() => {
      protocolLoading.value = false
    })
}
</script>

<style scoped lang="scss">
.lab-application-page {
  margin-top: 10px;
}

.lab-info {
  margin-bottom: 16px;
}

.equipment-list h4 {
  margin-bottom: 12px;
  color: #333;
}

.preview-content {
  padding: 16px 0;
}

.doc-time {
  margin-left: 8px;
  color: rgba(0, 0, 0, 0.45);
  font-size: 12px;
}

.lab-equipment-readonly {
  min-height: 24px;
}

.lab-equipment-readonly .ant-tag {
  margin-bottom: 4px;
}

.lab-equipment-readonly .eq-qty {
  margin-left: 4px;
  font-size: 12px;
  opacity: 0.85;
}

.no-equipment {
  color: rgba(0, 0, 0, 0.45);
  font-size: 12px;
}

@media (max-width: 768px) {
}
</style>
