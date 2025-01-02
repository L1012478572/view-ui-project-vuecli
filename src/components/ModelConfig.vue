<template>
    <div class="centered">
        <List>
            <ListItem>
                <h2>模型文件管理</h2>
            </ListItem>
            <ListItem>
                <Upload :before-upload="handleFileUpload">
                    <Button type="primary">选择文件</Button>
                </Upload>
                <span v-if="selectedFile">{{ selectedFile.name }}</span>
            </ListItem>
            <ListItem>
                <Button type="info" @click="uploadModel">上传模型文件</Button>
                <Button type="success" @click="fetchModelList">读取模型列表</Button>
            </ListItem>
            <ListItem>
                <List>
                    <ListItem v-for="model in models" :key="model.id" class="model-item">
                        <div class="model-info">
                            <span>{{ model.file_name }} ({{ model.upload_date }})</span>
                            <div class="model-actions">
                                <Button type="info" @click="downloadModel(model.file_name)">下载</Button>
                                <Button type="error" @click="deleteModel(model.file_name)">删除</Button>
                            </div>
                        </div>
                    </ListItem>
                </List>
            </ListItem>
            <ListItem>
                <p v-if="message" class="message">{{ message }}</p>
            </ListItem>
            <ListItem v-if="downloadProgress > 0">
                <div class="progress-bar">
                    <div class="progress" :style="{ width: downloadProgress + '%' }"></div>
                </div>
            </ListItem>
        </List>
    </div>
</template>

<script>
import axios from 'axios';
import { Button, List, ListItem, Upload } from 'view-ui-plus';

export default {
    components: {
        List,
        ListItem,
        Button,
        Upload
    },
    data() {
        return {
            models: [],
            selectedFile: null,
            message: '', // 添加一个状态变量来存储提示消息
            downloadProgress: 0 // 添加一个状态变量来存储下载进度
        };
    },
    methods: {
        async fetchModelList() {
            try {
                const host = window.location.hostname;
                const port = '8010'; // 你的后端端口号
                const url_send = `http://${host}:${port}/api/model/list`;
                const response = await axios.get(url_send);
                const parser = new DOMParser();
                const xmlDoc = parser.parseFromString(response.data, 'application/xml');
                const items = xmlDoc.getElementsByTagName('Item');
                this.models = Array.from(items).map(item => ({
                    id: item.getAttribute('id'),
                    file_name: item.getAttribute('name')
                }));
                this.message = '模型文件列表已更新';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            } catch (error) {
                console.error('获取模型文件列表时出错:', error);
                this.message = '获取模型文件列表时出错';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            }
        },
        handleFileUpload(file) {
            this.selectedFile = file;
            return false; // 阻止默认的上传行为
        },
        async uploadModel() {
            if (!this.selectedFile) {
                this.message = '请选择一个文件';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
                return;
            }
            const formData = new FormData();
            formData.append('file', this.selectedFile);
            try {
                const host = window.location.hostname;
                const port = '8010'; // 你的后端端口号
                const url_send = `http://${host}:${port}/api/model/upload`;
                const response = await axios.post(url_send, formData, {
                    headers: {
                        'Content-Type': 'multipart/form-data'
                    }
                });
                console.log('上传响应:', response.data);
                this.message = '模型文件上传成功';
                this.selectedFile = null;
                this.fetchModelList(); // 上传完成后刷新模型文件列表
            } catch (error) {
                console.error('上传模型文件时出错:', error);
                this.message = '上传模型文件时出错';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            }
        },
        async downloadModel(fileName) {
            try {
                const host = window.location.hostname;
                const port = '8010'; // 你的后端端口号
                const url_send = `http://${host}:${port}/api/model/download`;
                const response = await axios.get(url_send, {
                    params: { file: fileName },
                    responseType: 'blob',
                    onDownloadProgress: (progressEvent) => {
                        if (progressEvent.lengthComputable) {
                            this.downloadProgress = Math.round((progressEvent.loaded * 100) / progressEvent.total);
                        }
                    }
                });
                const url = window.URL.createObjectURL(new Blob([response.data]));
                const link = document.createElement('a');
                link.href = url;
                link.setAttribute('download', fileName);
                document.body.appendChild(link);
                link.click();
                this.message = '模型文件下载成功';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
                this.downloadProgress = 0; // 重置进度条
            } catch (error) {
                console.error('下载模型文件时出错:', error);
                this.message = '下载模型文件时出错';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
                this.downloadProgress = 0; // 重置进度条
            }
        },
        async deleteModel(fileName) {
            try {
                const host = window.location.hostname;
                const port = '8010'; // 你的后端端口号
                const url_send = `http://${host}:${port}/api/model/delete`;
                const response = await axios.put(url_send, null, {
                    params: { file: fileName }
                });
                console.log('删除响应:', response.data);
                this.message = '模型文件删除成功';
                this.fetchModelList(); // 删除完成后刷新模型文件列表
            } catch (error) {
                console.error('删除模型文件时出错:', error);
                this.message = '删除模型文件时出错';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            }
        }
    },
    mounted() {
        this.fetchModelList(); // 在组件挂载时获取模型文件列表
    }
};
</script>

<style scoped>
.centered {
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
}

.model-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px;
    border-bottom: 1px solid #eee;
}

.model-info {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 100%;
}

.model-actions {
    display: flex;
    gap: 10px;
}

.message {
    color: green;
    margin-top: 10px;
}

.progress-bar {
    width: 100%;
    background-color: #f3f3f3;
    border: 1px solid #ccc;
    margin-top: 10px;
}

.progress {
    height: 20px;
    background-color: #4caf50;
}
</style>