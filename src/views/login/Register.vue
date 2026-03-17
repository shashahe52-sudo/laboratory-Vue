<template>
  <div class="register-page">
    <div class="register-container">
      <div class="register-form-wrapper">
        <div class="register-header">
          <h1 class="register-title">实验室管理系统</h1>
          <p class="register-subtitle">创建新账户</p>
        </div>

        <a-form ref="formRef" :model="formData" :rules="formRules" class="register-form" @finish="handleRegister">
          <a-form-item name="username">
            <a-input v-model:value="formData.username" size="large" placeholder="用户名" :prefix="h(UserOutlined)" />
          </a-form-item>

          <a-form-item name="realName">
            <a-input v-model:value="formData.realName" size="large" placeholder="真实姓名" :prefix="h(UserOutlined)" />
          </a-form-item>

          <a-form-item name="studentId">
            <a-input v-model:value="formData.studentId" size="large" placeholder="学号/学工号" :prefix="h(UserOutlined)" />
          </a-form-item>

          <a-form-item name="phone">
            <a-input v-model:value="formData.phone" size="large" placeholder="手机号" :prefix="h(PhoneOutlined)" />
          </a-form-item>

          <a-form-item name="email">
            <a-input v-model:value="formData.email" size="large" placeholder="邮箱" :prefix="h(MailOutlined)" />
          </a-form-item>

          <a-form-item name="role">
            <a-radio-group v-model:value="formData.role" class="user-type-group">
              <a-radio-button value="teacher">教师</a-radio-button>
              <a-radio-button value="student">学生</a-radio-button>
            </a-radio-group>
          </a-form-item>

          <a-form-item name="password">
            <a-input-password v-model:value="formData.password" size="large" placeholder="密码"
              :prefix="h(LockOutlined)" />
          </a-form-item>

          <a-form-item name="confirmPassword">
            <a-input-password v-model:value="formData.confirmPassword" size="large" placeholder="确认密码"
              :prefix="h(LockOutlined)" />
          </a-form-item>

          <a-form-item>
            <a-button type="primary" html-type="submit" size="large" :loading="loading" class="register-button" block>
              注册 </a-button>
          </a-form-item>

          <div class="register-footer">
            <span>已有账户？</span>
            <a @click="goToLogin">立即登录</a>
          </div>
        </a-form>
      </div>

      <div class="register-banner">
        <div class="banner-content">
          <h2>加入实验室管理系统</h2>
          <p>享受便捷的实验室管理服务，提升学习和工作效率</p>
          <div class="feature-list">
            <div class="feature-item">
              <ExperimentOutlined />
              <span>实验室预约</span>
            </div>
            <div class="feature-item">
              <ToolOutlined />
              <span>设备管理</span>
            </div>
            <div class="feature-item">
              <FileTextOutlined />
              <span>申请流程</span>
            </div>
            <div class="feature-item">
              <TeamOutlined />
              <span>用户管理</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, h } from 'vue'
import { useRouter } from 'vue-router'
import { message } from 'ant-design-vue'
import { UserOutlined, MailOutlined, PhoneOutlined, LockOutlined, ExperimentOutlined, ToolOutlined, FileTextOutlined, TeamOutlined } from '@ant-design/icons-vue'
import { userRegisterService } from '@/api/user'

const router = useRouter()

// 响应式数据
const formRef = ref()
const loading = ref(false)

// 表单数据
const formData = reactive({
  username: '',
  realName: '',
  studentId: '',
  email: '',
  phone: '',
  role: 'student',
  password: '',
  confirmPassword: '',
  agreement: false
})

// 表单验证规则
const formRules = {
  username: [
    { required: true, message: '请输入用户名', trigger: 'blur' }
  ],
  realName: [
    { required: true, message: '请输入真实姓名', trigger: 'blur' },
    { min: 2, max: 20, message: '姓名长度在2到20个字符', trigger: 'blur' }
  ],
  studentId: [
    { required: true, message: '请输入学号/学工号', trigger: 'blur' }
  ],
  email: [
    { required: true, message: '请输入邮箱', trigger: 'blur' },
    { type: 'email', message: '请输入正确的邮箱格式', trigger: 'blur' }
  ],
  phone: [
    { required: true, message: '请输入手机号', trigger: 'blur' },
    { pattern: /^1[3-9]\d{9}$/, message: '请输入正确的手机号', trigger: 'blur' }
  ],
  role: [{ required: true, message: '请选择用户类型', trigger: 'change' }],
  password: [
    { required: true, message: '请输入密码', trigger: 'blur' },
    { min: 6, max: 20, message: '密码长度在6到20个字符', trigger: 'blur' }
  ],
  confirmPassword: [
    { required: true, message: '请确认密码', trigger: 'blur' },
    {
      validator: (rule, value) => {
        if (value !== formData.password) {
          return Promise.reject('两次输入的密码不一致')
        }
        return Promise.resolve()
      },
      trigger: 'blur'
    }
  ],
  agreement: [
    {
      validator: (rule, value) => {
        if (!value) {
          return Promise.reject('请同意用户协议和隐私政策')
        }
        return Promise.resolve()
      },
      trigger: 'change'
    }
  ]
}

// 事件处理
const handleRegister = async () => {
  await formRef.value.validate()
  loading.value = true

  // 只提交后端需要的字段
  const payload = {
    // 后端 RegisterRequest 需要 username，这里用学号/工号作为用户名
    username: formData.username,
    password: formData.password,
    realName: formData.realName,
    studentId: formData.studentId,
    phone: formData.phone,
    email: formData.email,
    role: formData.role
  }

  const result = await userRegisterService(payload).catch((error) => {
    // 这里不会抛出异常到外面，避免 Unhandled error
    message.error(error.message || '注册失败，请稍后重试')
    loading.value = false
    return null
  })

  if (result !== null) {
    message.success('注册成功！请登录')
    router.push('/login')
    loading.value = false
  }
}

const goToLogin = () => {
  router.push('/login')
}
</script>

<style scoped>
.register-page {
  max-height: 100vh;
  padding: 60px 0;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.register-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  display: flex;
  max-width: 1000px;
  width: 100%;
  max-height: 90vh;
}

.register-form-wrapper {
  flex: 1;
  padding: 30px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  overflow-y: auto;
}

.register-header {
  text-align: center;
  margin-bottom: 30px;
}

.register-title {
  font-size: 28px;
  font-weight: 600;
  color: #333;
  margin-bottom: 8px;
}

.register-subtitle {
  color: #666;
  font-size: 16px;
}

.register-form {
  max-width: 400px;
  margin: 0 auto;
  width: 100%;
}

.user-type-group {
  width: 100%;
  display: flex;
  justify-content: center;
}

.user-type-group :deep(.ant-radio-button-wrapper) {
  flex: 1;
  text-align: center;
}

.register-button {
  height: 48px;
  font-size: 16px;
  font-weight: 500;
  margin-top: 10px;
}

.register-footer {
  text-align: center;
  margin-top: 20px;
  color: #666;
}

.register-footer a {
  color: #1890ff;
  text-decoration: none;
  margin-left: 5px;
}

.register-footer a:hover {
  text-decoration: underline;
}

.register-banner {
  flex: 1;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 60px 40px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.banner-content {
  text-align: center;
  max-width: 400px;
}

.banner-content h2 {
  font-size: 32px;
  font-weight: 600;
  margin-bottom: 20px;
}

.banner-content p {
  font-size: 18px;
  margin-bottom: 40px;
  opacity: 0.9;
}

.feature-list {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}

.feature-item {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 16px;
}

.feature-item .anticon {
  font-size: 20px;
  color: #ffd700;
}

/* 移动端适配 - 768px以下 */
@media (max-width: 768px) {
  .register-page {
    padding: 20px;
  }

  .register-container {
    flex-direction: column;
    max-width: 100%;
    max-height: 100vh;
  }

  .register-banner {
    display: none;
  }

  .register-form-wrapper {
    padding: 20px;
  }

  .register-title {
    font-size: 24px;
  }

  .register-subtitle {
    font-size: 14px;
  }

  .user-type-group {
    flex-direction: column;
  }

  .user-type-group :deep(.ant-radio-button-wrapper) {
    margin-bottom: 8px;
  }

  .feature-list {
    grid-template-columns: 1fr;
    gap: 15px;
  }
}
</style>
