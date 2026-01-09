<template>
  <view class="box">
    <view class="title">Root 管理 - 用户列表</view>

    <!-- 用户列表 -->
    <view v-for="u in users" :key="u.id" class="row">
      <text class="name">{{ u.username }}  {{ u.is_root?'(管理员)':'' }}</text>
      <button size="mini" @click="resetPwd(u.id)">重置密码</button>
      <button size="mini" type="warn" @click="del(u.id)">删除</button>
    </view>

    <!-- 新建用户 -->
    <view class="add">
      <input v-model="add.username" placeholder="新用户名" class="in"/>
      <input v-model="add.password" placeholder="新密码" type="password" class="in"/>
      <view class="chk">
        <checkbox :checked="add.is_root" @click="add.is_root=!add.is_root"/>管理员
      </view>
      <text class="sub">可见工作流（多选）：</text>
      <checkbox-group @change="onWfChange">
        <label v-for="w in allWorkflows" :key="w.id" class="wf-item">
          <checkbox :value="w.id"/>{{ w.name }}
        </label>
      </checkbox-group>
      <button @click="create" class="btn">新建用户</button>
    </view>
  </view>
</template>

<script setup>
import { ref, onMounted } from 'vue';
const BASE_URL = 'http://192.168.44.173:8000';   // ←换成你的后端IP
const token = uni.getStorageSync('token');

const users = ref([]);
const allWorkflows = ref([]);
const add = ref({ username: '', password: '', is_root: false, workflows: [] });

onMounted(() => {
  loadUsers();
  loadWorkflows();
});

/* 用户列表 */
function loadUsers() {
  uni.request({
    url: `${BASE_URL}/api/root/users`,
    header: { token },
    success: res => users.value = res.data
  });
}

/* 全部工作流（root 看全部） */
function loadWorkflows() {
  uni.request({
    url: `${BASE_URL}/api/workflows/list`,
    header: { token },
    success: res => allWorkflows.value = res.data
  });
}

/* 多选工作流 */
function onWfChange(e) {
  add.value.workflows = e.detail.value;
}

/* 新建用户 */
function create() {
  uni.request({
    url: `${BASE_URL}/api/root/users`,
    method: 'POST',
    header: { token },
    data: add.value,
    success: () => {
      loadUsers();
      add.value = { username: '', password: '', is_root: false, workflows: [] };
    }
  });
}

/* 重置密码 */
function resetPwd(id) {
  uni.showModal({
    title: '重置密码',
    editable: true,
    placeholderText: '新密码',
    success: res => {
      if (res.confirm && res.content) {
        uni.request({
          url: `${BASE_URL}/api/root/users/${id}/password`,
          method: 'PATCH',
          header: { token },
          data: { newPwd: res.content },
          success: () => uni.showToast({ title: '已重置' })
        });
      }
    }
  });
}

/* 删除用户 */
function del(id) {
  uni.showModal({
    title: '确认',
    content: '确定删除该用户？',
    success: res => {
      if (res.confirm) {
        uni.request({
          url: `${BASE_URL}/api/root/users/${id}`,
          method: 'DELETE',
          header: { token },
          success: () => loadUsers()
        });
      }
    }
  });
}
</script>

<style lang="scss">
.box{ padding: 20px; }
.title{ font-size: 18px; font-weight: bold; margin-bottom: 15px; }
.row{ display: flex; align-items: center; justify-content: space-between; margin-bottom: 10px; }
.name{ flex: 1; }
.add{ margin-top: 30px; }
.in{ border: 1px solid #ddd; padding: 8px; border-radius: 4px; margin-bottom: 10px; }
.chk{ margin-bottom: 10px; }
.sub{ font-size: 14px; color: #666; margin: 10px 0 6px; }
.wf-item{ display: block; margin-bottom: 4px; }
.btn{ background: #007aff; color: #fff; border-radius: 4px; }
</style>