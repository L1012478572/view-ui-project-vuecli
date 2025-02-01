<template>
    <div>
        <h1>视频管理</h1>
    </div>
    <div class="centered">
        <List>
            <ListItem>
                <h2>视频文件管理</h2>
            </ListItem>
            <ListItem>
                <Button type="success" @click="fetchVideoList">读取视频列表</Button>
            </ListItem>
            <ListItem>
                <List>
                    <ListItem v-for="video in videos" :key="video.id" class="video-item">
                        <div class="video-info">
                            <span>{{ video.file_name }} ({{ video.upload_date }})</span>
                            <div class="video-actions">
                                <Button type="info" @click="downloadVideo(video.file_name)">下载</Button>
                                <Button type="error" @click="deleteVideo(video.file_name)">删除</Button>
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
            videos: [],
            selectedFile: null,
            message: '',
            downloadProgress: 0
        };
    },
    methods: {
        // 获取视频文件列表
        async fetchVideoList() {
            try {
                const host = window.location.hostname;
                const port = '8010';
                const url_send = `http://${host}:${port}/api/camera/read_videolist`;
                const response = await axios.get(url_send);
                this.videos = response.data.file_list.map((fileName, index) => ({
                    id: index,
                    file_name: fileName
                }));
                this.message = '视频文件列表已更新';
                setTimeout(() => this.message = '', 3000);
            } catch (error) {
                console.error('获取视频文件列表时出错:', error);
                this.message = '获取视频文件列表时出错';
                setTimeout(() => this.message = '', 3000);
            }
        },
        // 下载视频文件
        async downloadVideo(fileName) {
            try {
                const host = window.location.hostname;
                const port = '8010';
                const url_send = `http://${host}:${port}/api/camera/download_video`;
                const response = await axios.get(url_send, {
                    params: { file_name: fileName },
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
                this.message = '视频文件下载成功';
                setTimeout(() => this.message = '', 3000);
                this.downloadProgress = 0;
            } catch (error) {
                console.error('下载视频文件时出错:', error);
                this.message = '下载视频文件时出错';
                setTimeout(() => this.message = '', 3000);
                this.downloadProgress = 0;
            }
        },
        // 删除视频文件
        async deleteVideo(fileName) {
            try {
                const host = window.location.hostname;
                const port = '8010';
                const url_send = `http://${host}:${port}/api/camera/delete_video`;
                const response = await axios.put(url_send, null, {
                    params: { file_name: fileName }
                });
                console.log('删除响应:', response.data);
                this.message = '视频文件删除成功';
                this.fetchVideoList();
            } catch (error) {
                console.error('删除视频文件时出错:', error);
                this.message = '删除视频文件时出错';
                setTimeout(() => this.message = '', 3000);
            }
        }
    },
    mounted() {
        this.fetchVideoList();
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

.video-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px;
    border-bottom: 1px solid #eee;
}

.video-info {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 100%;
}

.video-actions {
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
