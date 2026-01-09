<template>
  <!-- 登录弹窗 -->
  <view v-if="!logined" class="login-mask">
    <view class="login-box">
      <text class="login-title">ADESIGNI登录</text>
      <input v-model="loginForm.username" placeholder="用户名" class="login-in"/>
      <input v-model="loginForm.password" placeholder="密码" type="password" class="login-in"/>
      <button @click="doLogin" class="login-btn">登录</button>
    </view>
  </view>

  <!-- 主页 -->
  <view v-else class="container">
    <!-- 全局进度广播条 -->
    <view v-if="showBar" class="global-progress">
      <text class="node-name">{{ curNode }}</text>
      <view class="bar-box">
        <view class="bar-inner" :style="{width: percent+'%'}"/>
      </view>
      <text class="percent">{{ percent }}%</text>
    </view>

    <!-- 顶部标题 + 队列等待数 -->
    <view class="header">
      <text class="title">ADESIGNI - {{ userName }}</text>
      <text class="queue-badge" :style="{background: waitingCount > 0 ? '#ff9800' : '#4cd964'}">
        {{ waitingCount > 0 ? '系统任务: ' + waitingCount : '系统空闲' }}
      </text>
    </view>

    <!-- 工作流选择 -->
    <view class="section">
      <text class="label">选择工作流模板</text>
      <picker @change="onWorkflowChange" :value="wfIndex" :range="workflowList" range-key="name">
        <view class="picker-box">
          {{ workflowList[wfIndex]?workflowList[wfIndex].name:'请选择工作流' }}
        </view>
      </picker>
    </view>

    <!-- 动态表单 -->
    <view class="section" v-if="currentSchema">
      <view v-for="(item,index) in currentSchema.inputs" :key="index" class="form-item">
        <text class="item-label">{{ item.label }}</text>
        <view v-if="item.type==='image'" class="upload-area" @click="chooseImage(item.key)">
          <image v-if="formData[item.key]" :src="formData[item.key]" mode="aspectFill"/>
          <text v-else>+</text>
        </view>
        <input v-else v-model="formData[item.key]" class="input-box" placeholder="请输入内容"/>
      </view>
    </view>

    <!-- 结果 -->
    <view class="section result-section" v-if="resultImageUrl">
      <text class="label">生成结果 (点击预览)</text>
      <image :src="resultImageUrl" mode="aspectFit" class="result-image" @click="previewImage"/>
    </view>

    <!-- 功能按钮区 -->
    <view class="section menu">
      <view class="menu-item" @click="changeMyPwd">
        <text>修改我的密码</text>
      </view>
      <view v-if="isRoot" class="menu-item" @click="goRoot">
        <text>Root 管理面板</text>
      </view>
      <view class="menu-item" @click="logout">
        <text>退出登录</text>
      </view>
    </view>

    <!-- 提交 -->
    <view class="footer">
      <button class="submit-btn" :disabled="isRunning" @click="startGeneration">
        {{ isRunning?'生成中('+progress+'%)':'开始生成' }}
      </button>
      <view v-if="isRunning" class="progress-bar">
        <view class="progress-inner" :style="{width: progress+'%'}"/>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from 'vue';

/* 基础配置 */
const BASE_URL = 'http://192.168.44.173:8000';   // ←换成你的后端IP
const logined = ref(false);
const userName = ref('');
const isRoot = ref(false);
const token = ref('');

/* 登录表单 */
const loginForm = reactive({ username: '', password: '' });

/* 工作流数据 */
const workflowList = ref([]);
const wfIndex = ref(-1);
const currentSchema = ref(null);
const formData = reactive({});
const imagePaths = reactive({});
const isRunning = ref(false);
const progress = ref(0);
const resultImageUrl = ref('');

/* 全局进度广播 */
const showBar = ref(false);
const curNode = ref('');
const percent = ref(0);
let socketTask = null;
let barTimer = null;

/* 队列等待数 */
const waitingCount = ref(0);
let queueTimer = null;

/* ---------- 登录 ---------- */
function doLogin() {
  uni.request({
    url: `${BASE_URL}/api/auth/login`,
	header: { token: uni.getStorageSync('token') },
    method: 'POST',
    data: loginForm,
    success: res => {
		if (res.statusCode !== 200) {
	        return uni.showToast({
	          title: res.data.detail || '登录失败',
	          icon: 'none'
	        });
	      }
      token.value = res.data.token;
      userName.value = res.data.username;
      isRoot.value = res.data.isRoot;
      logined.value = true;
      uni.setStorageSync('token', token.value);
      uni.setStorageSync('isRoot', isRoot.value);
      uni.setStorageSync('username', userName.value);
      startAfterLogin();
    },
    fail: () => uni.showToast({ title: '用户名或密码错误', icon: 'none' })
  });
}

/* 记住我 */
function checkAutoLogin() {
  const t = uni.getStorageSync('token');
  if (t) {
    token.value = t;
    userName.value = uni.getStorageSync('username');
    isRoot.value = uni.getStorageSync('isRoot');
    logined.value = true;
    startAfterLogin();
  }
}


/* 登录后初始化 */
function startAfterLogin() {
  loadWorkflowList();
  openProgressSocket();
  pullQueueCount();
  queueTimer = setInterval(pullQueueCount, 5000);
}

onMounted(() => checkAutoLogin());
onUnmounted(() => {
  closeProgressSocket();
  clearInterval(queueTimer);
});

/* ---------- 工作流 ---------- */
function loadWorkflowList() {
  uni.request({
    url: `${BASE_URL}/api/workflows/list`,
    header: { token: token.value },
    success: res => {
      if (res.data && res.data.length) workflowList.value = res.data;
    },
    fail: () => uni.showToast({ title: '服务器连接失败', icon: 'none' })
  });
}
function onWorkflowChange(e) {
  wfIndex.value = e.detail.value;
  const wf = workflowList.value[wfIndex.value];
  resultImageUrl.value = '';
  uni.request({
    url: `${BASE_URL}/api/workflows/${wf.id}/schema`,
    header: { token: token.value },
    success: res => {
      currentSchema.value = res.data;
      Object.keys(formData).forEach(k => delete formData[k]);
      res.data.inputs.forEach(item => { formData[item.key] = ''; });
    }
  });
}

/* ---------- 图片上传 ---------- */
function chooseImage(key) {
  uni.chooseImage({
    count: 1,
    success: res => {
      formData[key] = res.tempFilePaths[0];
      imagePaths[key] = res.tempFilePaths[0];
    }
  });
}

/* ---------- 生图 ---------- */
function startGeneration() {
  if (wfIndex.value === -1) return uni.showToast({ title: '请选择模板', icon: 'none' });
  const hasImage = currentSchema.value.inputs.some(i => i.type === 'image');
  const imageKey = currentSchema.value.inputs.find(i => i.type === 'image')?.key;
  if (hasImage && !imagePaths[imageKey]) return uni.showToast({ title: '请先上传图片', icon: 'none' });

  isRunning.value = true;
  progress.value = 0;
  resultImageUrl.value = '';

  const wfId = workflowList.value[wfIndex.value].id;
  const params = {};
  currentSchema.value.inputs.forEach(item => {
    if (item.type !== 'image') params[item.key] = formData[item.key];
  });

  uni.uploadFile({
    url: `${BASE_URL}/api/tasks/run/${wfId}`,
    filePath: imagePaths[imageKey] || '',
    name: 'files',
    header: { token: token.value },
    formData: { user: userName.value, params: JSON.stringify(params) },
    success: uploadRes => {
      try {
        const data = JSON.parse(uploadRes.data);
        pollStatus(data.comfy_id);
      } catch (e) {
        isRunning.value = false;
        uni.showToast({ title: '后端响应异常', icon: 'none' });
      }
    },
    fail: () => { isRunning.value = false; uni.showToast({ title: '提交失败', icon: 'none' }); }
  });
}

function pollStatus(promptId) {
  const timer = setInterval(() => {
    uni.request({
      url: `${BASE_URL}/api/tasks/status/${promptId}`,
      header: { token: token.value },
      success: res => {
        if (res.data) {
          progress.value = res.data.progress || 0;
          if (res.data.status === 'success') {
            clearInterval(timer);
            isRunning.value = false;
            resultImageUrl.value = `${BASE_URL}/${res.data.result_url}`;
            uni.showToast({ title: '生成完成', icon: 'success' });
          } else if (res.data.status === 'failed') {
            clearInterval(timer);
            isRunning.value = false;
            uni.showToast({ title: '生成失败', icon: 'none' });
          }
        }
      }
    });
  }, 2000);
}

function previewImage() {
  uni.previewImage({ urls: [resultImageUrl.value] });
}

/* ---------- 用户功能 ---------- */
function changeMyPwd() {
  uni.showModal({
    title: '新密码',
    editable: true,
    placeholderText: '请输入新密码',
    success: res => {
      if (res.confirm && res.content) {
        uni.request({
          url: `${BASE_URL}/api/auth/me/password`,
          method: 'PUT',
          header: { token: token.value },
          data: { newPwd: res.content },
          success: () => uni.showToast({ title: '密码已修改' })
        });
      }
    }
  });
}
function goRoot() {
  uni.navigateTo({ url: '/pages/root/root' });
}
function logout() {
  uni.clearStorageSync();
  logined.value = false;
  loginForm.username = '';
  loginForm.password = '';
}

/* ---------- WebSocket 进度 ---------- */
function openProgressSocket() {
  socketTask = uni.connectSocket({ url: 'ws://192.168.44.173:8001' }); // 请确保IP正确
  
  socketTask.onMessage(res => {
    const msg = JSON.parse(res.data);

    // 1. 监听到“开始运行节点”或“有进度” -> 强制显示忙碌
    // 即使 polling 还没轮到，也先让界面变橙色，给用户反馈
    if (msg.type === 'node_start' || msg.type === 'progress') {
       if (waitingCount.value === 0) {
         waitingCount.value = 1; // 视觉上立即变更为“系统任务: 1”
       }
    }

    // 原有逻辑：显示进度条
    if (msg.type === 'node_start') {
      showBar.value = true; 
      curNode.value = msg.node; 
      percent.value = 0;
    }
    if (msg.type === 'progress') {
      percent.value = Math.round((msg.value / msg.max) * 100);
    }

    // 2. 监听到“任务完成” -> 立即刷新队列数
    // 不要等那 5 秒的定时器了，现在就去问后端还有几个任务
    if (msg.type === 'finished') {
      percent.value = 100;
      pullQueueCount(); // <--- 关键修改：任务一结束，马上拉取最新队列数
      barTimer = setTimeout(() => showBar.value = false, 1000);
    }
  });

  socketTask.onError(e => console.error('进度通道错误', e));
}

/* ---------- 队列轮询 ---------- */
function pullQueueCount() {
  uni.request({
    url: `${BASE_URL}/api/tasks/queue_count`,
    header: { token: token.value },
    success: res => waitingCount.value = res.data.waiting,
    fail: () => waitingCount.value = 0
  });
}
</script>

<style lang="scss">
/* 登录遮罩 */
.login-mask{ position: fixed; inset: 0; background: #fff; z-index: 9999; display: flex; align-items: center; justify-content: center; }
.login-box{ width: 80%; }
.login-title{ font-size: 20px; font-weight: bold; margin-bottom: 20px; text-align: center; }
.login-in{ border: 1px solid #ddd; padding: 10px; border-radius: 6px; margin-bottom: 15px; }
.login-btn{ background: #007aff; color: #fff; border-radius: 6px; }

/* 全局进度条 */
.global-progress{
  position: fixed; top: 0; left: 0; right: 0; height: 44px;
  background: #fff; box-shadow: 0 2px 6px rgba(0,0,0,.08);
  display: flex; align-items: center; padding: 0 15px; z-index: 999;
  .node-name{ font-size: 13px; color: #333; width: 100rpx; white-space: nowrap; overflow: hidden;}
  .bar-box{ flex: 1; height: 6px; background: #eee; border-radius: 3px; margin: 0 8px; overflow: hidden;}
  .bar-inner{ height: 100%; background: #4cd964; transition: width .3s;}
  .percent{ font-size: 12px; color: #666; width: 40rpx; text-align: right;}
}

/* 顶部标题 + 队列徽章 */
.header{ display: flex; align-items: center; justify-content: space-between; margin-bottom: 20px; }
.title{ font-size: 18px; font-weight: bold; color: #333; }
/* 原样式保持不变，颜色由行内 style 动态控制 */
.queue-badge{ color: #fff; font-size: 12px; padding: 2px 8px; border-radius: 10px; }
/* 菜单区 */
.menu{ padding: 0; }
.menu-item{ padding: 12px 0; border-bottom: 1px solid #f0f0f0; display: flex; justify-content: space-between; align-items: center; }
.menu-item:last-child{ border: none; }

/* 其余样式与之前相同 */
.container{ padding: 20px; background-color: #f8f8f8; min-height: 100vh; padding-top: 60px;}
.section{ background: white; padding: 15px; border-radius: 10px; margin-bottom: 15px;}
.label{ font-size: 14px; color: #666; margin-bottom: 10px; display: block;}
.picker-box{ padding: 10px; border: 1px solid #ddd; border-radius: 5px; height: 40px; display: flex; align-items: center;}
.form-item{ margin-bottom: 15px;}
.item-label{ font-size: 13px; color: #444; margin-bottom: 5px; display: block;}
.input-box{ border: 1px solid #ddd; padding: 10px; border-radius: 5px; width: 100%; box-sizing: border-box; height: 40px; font-size: 14px;}
.upload-area{ width: 80px; height: 80px; border: 2px dashed #ddd; border-radius: 8px; display: flex; align-items: center; justify-content: center; font-size: 24px; color: #999;
  image{ width: 100%; height: 100%; border-radius: 8px;}
}
.result-section{ display: flex; flex-direction: column; align-items: center;
  .result-image{ width: 100%; height: 260px; border-radius: 8px; margin-top: 5px; background-color: #eee;}
}
.footer{ margin-top: 20px; padding-bottom: 40px;}
.submit-btn{ background-color: #007aff; color: white; border-radius: 25px; height: 44px; line-height: 44px; font-size: 16px;}
.progress-bar{ width: 100%; height: 6px; background: #eee; border-radius: 3px; margin-top: 15px; overflow: hidden;}
.progress-inner{ height: 100%; background: #4cd964; transition: width .3s;}
</style>