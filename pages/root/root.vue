<template>
  <view class="box">
    <view class="title">Root 管理 - 用户列表</view>

    <view v-for="u in users" :key="u.id" class="row">
      <view class="user-info">
        <text class="name">{{ u.username }}</text>
        <text v-if="u.is_root" class="tag-admin">管理员</text>
      </view>
      
      <view class="btn-group">
        <button size="mini" type="primary" class="mini-btn" @click="editUser(u)">编辑</button>
        <button size="mini" class="mini-btn" @click="resetPwd(u.id)">重置密码</button>
        <button size="mini" type="warn" class="mini-btn" @click="del(u.id)">删除</button>
      </view>
    </view>

    <view class="divider"></view>

    <view class="add-section">
      <view class="form-header">
        <text class="sub-title">{{ editingId ? '修改用户权限' : '新建用户' }}</text>
        <text v-if="editingId" class="cancel-link" @click="cancelEdit">取消编辑</text>
      </view>

      <input 
        v-model="form.username" 
        :disabled="!!editingId" 
        placeholder="用户名" 
        class="in"
        :class="{ disabled: !!editingId }"
      />
      
      <input 
        v-model="form.password" 
        :placeholder="editingId ? '若不修改密码请留空' : '设置密码'" 
        type="password" 
        class="in" 
      />

      <view class="chk-row">
        <label class="chk-label" @click="form.is_root = !form.is_root">
          <checkbox :checked="form.is_root" /> 设为管理员 (Root)
        </label>
      </view>

      <text class="label-text">分配可见工作流：</text>
      <checkbox-group @change="onWfChange">
        <label v-for="w in allWorkflows" :key="w.id" class="wf-item">
          <checkbox :value="w.id" :checked="form.workflows.includes(w.id)" />
          {{ w.name }}
        </label>
      </checkbox-group>

      <button @click="submitForm" class="btn" :class="{ 'btn-edit': !!editingId }">
        {{ editingId ? '保存修改' : '立即创建' }}
      </button>
    </view>
  </view>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';

/* 配置项 */
const BASE_URL = 'http://192.168.44.173:8000'; // 请确保与您的后端地址一致
const token = uni.getStorageSync('token');

/* 数据源 */
const users = ref([]);
const allWorkflows = ref([]);

/* 表单状态 */
const editingId = ref(0); // 0 表示新建模式，非 0 表示正在编辑的用户ID
const form = reactive({
  username: '',
  password: '',
  is_root: false,
  workflows: [] // 存放选中的工作流 ID 字符串数组
});

onMounted(() => {
  loadUsers();
  loadWorkflows();
});

/* 加载用户列表 */
function loadUsers() {
  uni.request({
    url: `${BASE_URL}/api/root/users`,
    header: { token },
    success: res => {
      // 确保后端 list_users 接口返回了 workflows 字段
      users.value = res.data; 
    },
    fail: () => uni.showToast({ title: '加载用户失败', icon: 'none' })
  });
}

/* 加载所有可用工作流 */
function loadWorkflows() {
  uni.request({
    url: `${BASE_URL}/api/workflows/list`,
    header: { token },
    success: res => {
      allWorkflows.value = res.data;
    }
  });
}

/* 监听多选框变化 */
function onWfChange(e) {
  form.workflows = e.detail.value;
}

/* 进入编辑模式 */
function editUser(u) {
  editingId.value = u.id;
  // 回填数据
  form.username = u.username;
  form.is_root = u.is_root;
  // 注意：后端需要返回 workflows 数组，否则这里为空
  form.workflows = u.workflows || []; 
  form.password = ''; // 密码清空，避免误操作
  
  // 滚动到底部方便操作
  uni.pageScrollTo({ scrollTop: 9999, duration: 300 });
}

/* 取消编辑 / 重置表单 */
function cancelEdit() {
  editingId.value = 0;
  form.username = '';
  form.password = '';
  form.is_root = false;
  form.workflows = [];
}

/* 提交表单（新建或更新） */
function submitForm() {
  // 基础校验
  if (!form.username) return uni.showToast({ title: '请输入用户名', icon: 'none' });
  if (!editingId.value && !form.password) return uni.showToast({ title: '新建用户必须设置密码', icon: 'none' });

  const isEdit = !!editingId.value;
  const url = isEdit 
    ? `${BASE_URL}/api/root/users/${editingId.value}` 
    : `${BASE_URL}/api/root/users`;
  const method = isEdit ? 'PUT' : 'POST';

  uni.request({
    url: url,
    method: method,
    header: { token },
    data: {
      username: form.username,
      password: form.password || null, // 编辑模式下若为空传null
      is_root: form.is_root,
      workflows: form.workflows
    },
    success: (res) => {
      if (res.statusCode >= 400) {
        return uni.showToast({ title: res.data.detail || '操作失败', icon: 'none' });
      }
      uni.showToast({ title: isEdit ? '修改成功' : '创建成功' });
      loadUsers();  // 刷新列表
      cancelEdit(); // 重置状态
    },
    fail: () => uni.showToast({ title: '请求失败', icon: 'none' })
  });
}


/* 重置密码（独立功能） */
function resetPwd(id) {
  uni.showModal({
    title: '重置密码',
    editable: true,
    placeholderText: '请输入新密码',
    success: res => {
      if (res.confirm && res.content) {
        uni.request({
          url: `${BASE_URL}/api/root/users/${id}/password`,
          method: 'PATCH',
          header: { token },
          data: { newPwd: res.content }, // 发送 JSON 数据
          success: (apiRes) => {
            // 【关键修改】必须检查状态码
            if (apiRes.statusCode !== 200) {
              return uni.showToast({ title: '重置失败，请查看后端日志', icon: 'none' });
            }
            uni.showToast({ title: '密码已重置' });
          },
          fail: () => uni.showToast({ title: '请求失败', icon: 'none' })
        });
      }
    }
  });
}

/* 删除用户 */
function del(id) {
  uni.showModal({
    title: '危险操作',
    content: '确定要删除该用户及其所有权限吗？',
    confirmColor: '#dd524d',
    success: res => {
      if (res.confirm) {
        uni.request({
          url: `${BASE_URL}/api/root/users/${id}`,
          method: 'DELETE',
          header: { token },
          success: () => {
            uni.showToast({ title: '已删除' });
            loadUsers();
            // 如果删除的是当前正在编辑的用户，重置表单
            if (editingId.value === id) cancelEdit();
          }
        });
      }
    }
  });
}
</script>

<style lang="scss">
.box { padding: 20px; padding-bottom: 50px; }
.title { font-size: 20px; font-weight: bold; margin-bottom: 20px; color: #333; }

/* 列表行样式 */
.row {
  display: flex; flex-direction: column; 
  background: #fff; border-radius: 8px; padding: 12px; margin-bottom: 12px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.05);
}
.user-info { display: flex; align-items: center; margin-bottom: 10px; }
.name { font-size: 16px; font-weight: 500; color: #333; margin-right: 8px; }
.tag-admin { font-size: 10px; color: #fff; background: #f0ad4e; padding: 2px 6px; border-radius: 4px; }

.btn-group { display: flex; justify-content: flex-end; gap: 8px; }
.mini-btn { margin: 0 !important; font-size: 12px; }

.divider { height: 1px; background: #eee; margin: 20px 0; }

/* 表单区域样式 */
.add-section { background: #f9f9f9; padding: 15px; border-radius: 10px; border: 1px solid #eee; }
.form-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; }
.sub-title { font-size: 16px; font-weight: bold; color: #555; }
.cancel-link { font-size: 14px; color: #007aff; }

.in { 
  border: 1px solid #ddd; background: #fff; padding: 10px; border-radius: 6px; margin-bottom: 12px; font-size: 14px; 
}
.in.disabled { background: #eee; color: #999; }

.chk-row { margin-bottom: 15px; }
.chk-label { display: flex; align-items: center; font-size: 14px; color: #333; }
.label-text { font-size: 14px; color: #666; margin-bottom: 8px; display: block; }
.wf-item { display: flex; align-items: center; margin-bottom: 8px; font-size: 14px; color: #333; }

.btn { 
  margin-top: 15px; background: #007aff; color: #fff; border-radius: 6px; font-size: 16px; 
  transition: background 0.3s;
}
.btn-edit { background: #4cd964; } /* 编辑保存时显示绿色 */
</style>