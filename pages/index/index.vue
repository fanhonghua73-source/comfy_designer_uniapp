<template>
	<view class="container">
		<view class="header">
			<text class="title">AI 设计助手 - {{ userName || '未登录' }}</text>
		</view>

		<view class="section">
			<text class="label">选择工作流模板</text>
			<picker @change="onWorkflowChange" :value="wfIndex" :range="workflowList" range-key="name">
				<view class="picker-box">
					{{ workflowList[wfIndex] ? workflowList[wfIndex].name : '请选择工作流' }}
				</view>
			</picker>
		</view>

		<view class="section" v-if="currentSchema">
			<view v-for="(item, index) in currentSchema.inputs" :key="index" class="form-item">
				<text class="item-label">{{ item.label }}</text>
				<view v-if="item.type === 'image'" class="upload-area" @click="chooseImage(item.key)">
					<image v-if="formData[item.key]" :src="formData[item.key]" mode="aspectFill"></image>
					<text v-else>+</text>
				</view>
				<input v-else v-model="formData[item.key]" class="input-box" placeholder="请输入内容" />
			</view>
		</view>

		<view class="section result-section" v-if="resultImageUrl">
			<text class="label">生成结果 (点击预览)</text>
			<image :src="resultImageUrl" mode="aspectFit" class="result-image" @click="previewImage"></image>
		</view>

		<view class="footer">
			<button class="submit-btn" :disabled="isRunning" @click="startGeneration">
				{{ isRunning ? '生成中 (' + progress + '%)' : '开始生成' }}
			</button>
			
			<view v-if="isRunning" class="progress-bar">
				<view class="progress-inner" :style="{width: progress + '%'}"></view>
			</view>
		</view>
	</view>
</template>

<script setup>
import { ref, onMounted, reactive } from 'vue';

// 【重要】请确保此 IP 是你后端电脑的真实局域网 IP
const BASE_URL = 'http://192.168.44.173:8000'; 
const userName = ref('');

const workflowList = ref([]);
const wfIndex = ref(-1);
const currentSchema = ref(null);
const formData = reactive({});
const imagePaths = reactive({});
const isRunning = ref(false);
const progress = ref(0);
const resultImageUrl = ref(''); // 存放最终生成的图片地址

// --- 初始化：获取工作流列表 ---
onMounted(() => {
	console.log("=== UniApp 生命周期：onMounted 启动 ===");
	const savedName = uni.getStorageSync('designer_name');
	if (savedName) {
		userName.value = savedName;
	} else {
		uni.showModal({
			title: '身份登记',
			editable: true,
			placeholderText: '请输入你的姓名',
			success: (res) => {
				if (res.confirm && res.content) {
					userName.value = res.content;
					uni.setStorageSync('designer_name', res.content);
				}
			}
		});
	}

	uni.request({
		url: `${BASE_URL}/api/workflows/list`,
		method: 'GET',
		success: (res) => {
			if (res.data && res.data.length > 0) {
				workflowList.value = res.data;
			}
		},
		fail: (err) => {
			console.error("无法连接后端:", err);
			uni.showToast({ title: '服务器连接失败', icon: 'none' });
		}
	});
});

// --- 选择工作流：加载表单结构 ---
const onWorkflowChange = (e) => {
	wfIndex.value = e.detail.value;
	const wf = workflowList.value[wfIndex.value];
	resultImageUrl.value = ''; // 切换模板时清空上次结果
	
	uni.request({
		url: `${BASE_URL}/api/workflows/${wf.id}/schema`,
		success: (res) => {
			currentSchema.value = res.data;
			// 重置表单数据
			Object.keys(formData).forEach(key => delete formData[key]);
			res.data.inputs.forEach(item => { 
				formData[item.key] = ''; 
			});
		}
	});
};

const chooseImage = (key) => {
	uni.chooseImage({
		count: 1,
		success: (res) => {
			formData[key] = res.tempFilePaths[0]; // 预览
			imagePaths[key] = res.tempFilePaths[0]; // 待上传
		}
	});
};

// --- 提交任务 ---
const startGeneration = () => {
	if (wfIndex.value === -1) return uni.showToast({ title: '请选择模板', icon: 'none' });
	
	// 简单校验：如果有图片输入项，必须选图
	const hasImageField = currentSchema.value.inputs.some(i => i.type === 'image');
	const imageKey = currentSchema.value.inputs.find(i => i.type === 'image')?.key;
	if (hasImageField && !imagePaths[imageKey]) {
		return uni.showToast({ title: '请先上传图片', icon: 'none' });
	}

	isRunning.value = true;
	progress.value = 0;
	resultImageUrl.value = '';

	const wfId = workflowList.value[wfIndex.value].id;
	const params = {};
	currentSchema.value.inputs.forEach(item => {
		if(item.type !== 'image') params[item.key] = formData[item.key];
	});

	uni.uploadFile({
		url: `${BASE_URL}/api/tasks/run/${wfId}`,
		filePath: imagePaths[imageKey] || '',
		name: 'files',
		formData: {
			'user': userName.value,
			'params': JSON.stringify(params)
		},
		success: (uploadRes) => {
			try {
				const data = JSON.parse(uploadRes.data);
				pollStatus(data.comfy_id);
			} catch (e) {
				isRunning.value = false;
				uni.showToast({ title: '后端响应异常', icon: 'none' });
			}
		},
		fail: () => {
			isRunning.value = false;
			uni.showToast({ title: '提交失败', icon: 'none' });
		}
	});
};

// --- 轮询状态与结果 ---
const pollStatus = (promptId) => {
	const timer = setInterval(() => {
		uni.request({
			url: `${BASE_URL}/api/tasks/status/${promptId}`,
			success: (res) => {
				if (res.data) {
					// 更新进度
					progress.value = res.data.progress || 0;
					
					if (res.data.status === 'success') {
						clearInterval(timer);
						isRunning.value = false;
						// 拼接完整的图片访问地址
						// 后端返回的 result_url 应该是 "outputs/xxx/output/xxx.png" 格式
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
};

// 预览图片
const previewImage = () => {
	uni.previewImage({
		urls: [resultImageUrl.value]
	});
};
</script>

<style lang="scss">
.container { padding: 20px; background-color: #f8f8f8; min-height: 100vh; }
.header { margin-bottom: 20px; .title { font-size: 18px; font-weight: bold; color: #333; } }
.section { background: white; padding: 15px; border-radius: 10px; margin-bottom: 15px; }
.label { font-size: 14px; color: #666; margin-bottom: 10px; display: block; }
.picker-box { padding: 10px; border: 1px solid #ddd; border-radius: 5px; height: 40px; display: flex; align-items: center; }
.form-item { margin-bottom: 15px; }
.item-label { font-size: 13px; color: #444; margin-bottom: 5px; display: block; }
.input-box { border: 1px solid #ddd; padding: 10px; border-radius: 5px; width: 100%; box-sizing: border-box; height: 40px; font-size: 14px; }
.upload-area { 
	width: 80px; height: 80px; border: 2px dashed #ddd; border-radius: 8px;
	display: flex; align-items: center; justify-content: center; font-size: 24px; color: #999;
	image { width: 100%; height: 100%; border-radius: 8px; }
}
.result-section {
	display: flex; flex-direction: column; align-items: center;
	.result-image { width: 100%; height: 260px; border-radius: 8px; margin-top: 5px; background-color: #eee; }
}
.footer { margin-top: 20px; padding-bottom: 40px; }
.submit-btn { background-color: #007aff; color: white; border-radius: 25px; height: 44px; line-height: 44px; font-size: 16px; }
.progress-bar { width: 100%; height: 6px; background: #eee; border-radius: 3px; margin-top: 15px; overflow: hidden; }
.progress-inner { height: 100%; background: #4cd964; transition: width 0.3s; }
</style>