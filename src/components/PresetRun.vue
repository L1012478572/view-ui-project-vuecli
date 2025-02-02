<template>
<div>
    <Grid square :col=3>
        <GridItem>
            <iframe id="webrtc-iframe" :src="iframeSrc" frameborder="0" allowfullscreen></iframe>
        </GridItem>
        <GridItem>
            <div>
                <label>选择预制信息:</label>
                <Select name="PresetSelect" v-model="Preset_Select" @on-change="PresetDataChange_Select">
                    <Option v-for="preset in presetList" :key="preset.id" :value="preset.id">{{ preset.preset_name }}</Option>
                </Select>
                <label>预制信息编号</label>
                <Input name="PresetInput" type="number" v-model="Preset_Input" @on-change="PresetDataChange_Input" />
                <br>
                <Button type="info" @click="runPreset">运行</Button>
                <Button type="warning" @click="pausePreset">暂停</Button>
                <br>
                <label>任务进度:</label>
                <Progress :percent="task_progress" :stroke-width="20" text-inside />
                <br>
                <label>任务状态:</label>
                <label>{{ task_status }}</label>
                <br>
                <label>任务测温数量:</label>
                <label>{{ task_thermal_num }}</label>
                <br>
                <label>实时最高温度:</label>
                <label>{{ real_max_temp }}</label>
                <br>
                <label>实时最低温度:</label>
                <label>{{ real_min_temp }}</label>
                <br>
                <label>实时平均温度:</label>
                <label>{{ real_avg_temp }}</label>
                <br>
                <br>
                <Button type="info" @click="readTaskResult">读取任务结果</Button>
                <br>
                <br>
                <Button type="info" @click="on_capture">拍照</Button>
                <div>   
                    <br>
                    <Button type="info" @click="startRecording">开始录像</Button>
                    <Button type="info" @click="stopRecording">停止录像</Button>
                </div>
            </div>
        </GridItem>
        <GridItem>
            <div>
                <br>
                <br>
                <div id="temperature-chart" style="width: 100%; height: 400px;"></div>
            </div>
        </GridItem>
        <GridItem>
            <div>
                <Table height="200" :columns="columns" :data="tab_data"></Table>
            </div>
        </GridItem>
        <GridItem>
            <div class="image-container">
                <div class="image-grid">
                    <img v-for="(image, index) in images" :key="index" :src="image.src" class="grid-image" @click="showLargeImage(image)">
                </div>
            </div>
        </GridItem>
    </Grid>
</div>
</template>

<script>
import axios from 'axios';
import {
    Slider,
    Input,
    Select,
    Button,
    List,
    ListItem
} from 'view-ui-plus';
import {
    Table,
    Progress,
    Grid,
    GridItem
} from 'view-ui-plus';
import * as echarts from 'echarts';

export default {
    name: 'PresetRun',
    components: {
        Slider,
        Input,
        Select,
        Button,
        List,
        ListItem,
        Grid,
        GridItem,
        Progress,
        Table
    },
    data() {
        return {
            isSaveVideo: false, // 是否正在保存视频
            presetList: [],
            Preset_Select: null, // 选择预制信息 Select组件
            Preset_Input: null, // 选择预制信息 Input组件
            ptz_speed: 25, // 云台速度
            message: '', // 提示消息
            iframeSrc: '', // iframe的src webRTC流地址
            task_progress: 0, // 任务进度
            task_status: '未知', // 任务状态
            task_thermal_num: 0, // 任务测温数量
            real_max_temp: 0.1, // 实时最高温度
            real_min_temp: 0.1, // 实时最低温度
            real_avg_temp: 0.1, // 实时平均温度
            list_maxThermals: [35.1, 35.2, 35.3, 35.4], // 最大温度列表
            list_minThermals: [24.1, 24.2, 24.3, 24.4], // 最小温度列表
            list_avgThermals: [14.6, 14.7, 14.8, 14.9], // 平均温度列表
            maxThermals_max: 0.1, // 最大温度最大值
            maxThermals_min: 0.1, // 最大温度最小值
            maxThermals_avg: 0.1, // 最大温度平均值
            minThermals_max: 0.1, // 最小温度最大值
            minThermals_min: 0.1, // 最小温度最小值
            minThermals_avg: 0.1, // 最小温度平均值
            avgThermals_max: 0.1, // 平均温度最大值
            avgThermals_min: 0.1, // 平均温度最小值
            avgThermals_avg: 0.1, // 平均温度平均值
            columns: [{
                    title: 'Type',
                    key: 'type'
                },
                {
                    title: 'Max',
                    key: 'max'
                },
                {
                    title: 'Min',
                    key: 'min'
                },
                {
                    title: 'Avg',
                    key: 'avg'
                }
            ],
            tab_data: [{
                    type: '最大温度',
                    max: this.maxThermals_max,
                    min: this.maxThermals_min,
                    avg: this.maxThermals_avg
                },
                {
                    type: '最小温度',
                    max: this.minThermals_max,
                    min: this.minThermals_min,
                    avg: this.minThermals_avg
                },
                {
                    type: '平均温度',
                    max: this.avgThermals_max,
                    min: this.avgThermals_min,
                    avg: this.avgThermals_avg
                }
            ],
            images: Array(20).fill().map((_, i) => ({
                src: '', // 这里填入实际的图片路径
                id: i
            })),
            scrollPosition: 0,
            scrollValue: 0,
            maxScroll: 0,
            timer: null,
            timer_heartbeat: null,
        };
    },
    mounted() {
        this.fetchPresetList();
        this.initIframe('mppstream');
        this.timer = setInterval(this.readTaskProgress, 2000);
        this.timer_heartbeat = setInterval(this.sendSaveHeartbeat, 5000);
        this.initChart();
        this.calculateMaxScroll();
    },
    beforeDestroy() {
        clearInterval(this.timer);
        clearInterval(this.timer_heartbeat);
    },
    methods: {
        // 初始化iframe
        initIframe(stream) {
            const host = window.location.hostname;
            this.iframeSrc = `http://${host}:8889/${stream}`;
            const iframe = document.getElementById('webrtc-iframe');
            iframe.style.width = '100%';
            iframe.style.height = '100%';
        },
        // 获取预制信息列表
        async fetchPresetList() {
            try {
                const host = window.location.hostname;
                const port = '8010'; // 你的后端端口号
                const url_send = `http://${host}:${port}/api/param/presetlist`;
                const response = await axios.get(url_send);
                const parser = new DOMParser();
                const xmlDoc = parser.parseFromString(response.data, 'application/xml');
                const items = xmlDoc.getElementsByTagName('Item');
                this.presetList = Array.from(items).map(item => ({
                    id: item.getAttribute('id'),
                    preset_name: item.getAttribute('preset_name'),
                    modle_name: item.getAttribute('model_name'),
                    ir_model_name: item.getAttribute('ir_model_name'),
                    cla_id: item.getAttribute('cla_id'),
                    cla_name: item.getAttribute('cla_name'),
                    score_val: item.getAttribute('score_val'),
                    object_id: item.getAttribute('object_id'),
                    ir_object_id: item.getAttribute('ir_object_id'),
                    location: item.getAttribute('location'),
                    image_channel: item.getAttribute('image_channel'),
                    isRunIrModel: item.getAttribute('isRunIrModel'),
                    isControlPtz: item.getAttribute('isControlPtz'),
                    thermal_num: item.getAttribute('thermal_num')
                }));
                // 按照标号排序
                this.presetList.sort((a, b) => parseInt(a.id) - parseInt(b.id));
                this.message = '预制信息列表已更新';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            } catch (error) {
                console.error('获取预制信息列表时出错:', error);
                this.message = '获取预制信息列表时出错';
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            }
        },
        // 选择预制信息编号
        PresetDataChange_Input() {
            // console.log(this.Preset_Input);
            // 判断是否超过预制信息列表的长度
            if (this.Preset_Input > this.presetList.length) {
                this.$Message.error('预制信息编号超出范围');
                return;
            }
            // 设置PresetSelect的值
            this.Preset_Select = this.Preset_Input;
            // console.log(this.Preset_Select);
        },
        // 选择预制信息
        PresetDataChange_Select() {
            // console.log(this.Preset_Select);
            this.Preset_Input = parseInt(this.Preset_Select);
            // console.log(this.Preset_Input);
        },
        // 运行预制信息
        async runPreset() {
            if (this.Preset_Select === null) {
                this.$Message.error('请选择预制信息');
                return;
            }
            const preset = this.presetList.find(preset => preset.id === this.Preset_Select);
            if (preset === undefined) {
                this.$Message.error('预制信息不存在');
                return;
            }
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/task/execute?preset_id=${preset.id}`;
            // post
            try {
                const response = await axios.put(url_send);
                console.log(response.data);
                const responseData = JSON.stringify(response.data);
                if (responseData.indexOf('Task executed successfully') !== -1) {
                    this.$Message.success('预制信息已发送');
                } else {
                    this.$Message.error('预制信息发送失败');
                }
            } catch (error) {
                console.error('发送预制信息时出错:', error);
                this.$Message.error('发送预制信息时出错');
            }
        },
        // 暂停预制信息
        async pausePreset() {
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/task/pause`;
            // post
            try {
                const response = await axios.put(url_send);
                console.log(response.data);
                const responseData = JSON.stringify(response.data);
                if (responseData.indexOf('success') !== -1) {
                    this.$Message.success('预制信息已暂停');
                } else {
                    this.$Message.error('预制信息暂停失败');
                }
            } catch (error) {
                console.error('暂停预制信息时出错:', error);
                this.$Message.error('暂停预制信息时出错');
            }
        },
        // 读任务进度
        async readTaskProgress() {
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/task/progress`;
            try {
                const response = await axios.get(url_send);
                const responseData = response.data;
                if (responseData.status) {
                    switch (responseData.status) {
                        case 'idle':
                            this.task_status = '空闲';
                            break;
                        case 'running':
                            this.task_status = '运行中';
                            break;
                        default:
                            this.task_status = '未知';
                    }
                }
                if (responseData.progress) {
                    this.task_progress = parseInt(responseData.progress);
                }
                if (responseData.thermal_num) {
                    this.task_thermal_num = parseInt(responseData.thermal_num)
                }
                if (responseData.realtime_max_thermal) {
                    this.real_max_temp = parseFloat(responseData.realtime_max_thermal)
                }
                if (responseData.realtime_min_thermal) {
                    this.real_min_temp = parseFloat(responseData.realtime_min_thermal)
                }
                if (responseData.realtime_avg_thermal) {
                    this.real_avg_temp = parseFloat(responseData.realtime_avg_thermal)
                }
            } catch (error) {
                console.error('获取任务进度时出错:', error);
            }
        },
        // 读取任务结果
        async readTaskResult() {
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/task/result`;
            try {
                const response = await axios.get(url_send);
                const responseData = response.data;
                if (responseData.status === 'success') {
                    this.$Message.success('任务结果已更新');

                } else {
                    this.$Message.error('任务仍在运行中, 展示部分结果');
                }
                this.list_maxThermals = responseData.list_maxThermals;
                this.list_minThermals = responseData.list_minThermals;
                this.list_avgThermals = responseData.list_avgThermals;
                this.maxThermals_max = responseData.maxThermals_max;
                this.maxThermals_min = responseData.maxThermals_min;
                this.maxThermals_avg = responseData.maxThermals_avg;
                this.minThermals_max = responseData.minThermals_max;
                this.minThermals_min = responseData.minThermals_min;
                this.minThermals_avg = responseData.minThermals_avg;
                this.avgThermals_max = responseData.avgThermals_max;
                this.avgThermals_min = responseData.avgThermals_min;
                this.avgThermals_avg = responseData.avgThermals_avg;
                this.tab_data = [{
                        type: '最大温度',
                        max: parseFloat(responseData.maxThermals_max),
                        min: parseFloat(responseData.maxThermals_min),
                        avg: parseFloat(responseData.maxThermals_avg)
                    },
                    {
                        type: '最小温度',
                        max: parseFloat(responseData.minThermals_max),
                        min: parseFloat(responseData.minThermals_min),
                        avg: parseFloat(responseData.minThermals_avg)
                    },
                    {
                        type: '平均温度',
                        max: parseFloat(responseData.avgThermals_max),
                        min: parseFloat(responseData.avgThermals_min),
                        avg: parseFloat(responseData.avgThermals_avg)
                    }
                ];
                this.initChart(); // 更新图表
            } catch (error) {
                console.error('读取任务结果时出错:', error);
                this.$Message.error('读取任务结果时出错');
            }
            // 读取测温过程图像
            for (let i = 0; i < 20; i++) {
                this.readThermalImage(i);
            }
        },
        // 拍照
        async on_capture() {
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/camera/capture`;
            try {
                const response = await axios.get(url_send, {
                    responseType: 'arraybuffer'
                });
                const contentType = response.headers['content-type'];
                console.log(contentType)
                if (contentType && contentType.includes('image/jpeg')) {
                    console.log('测试预制信息成功');
                    const blob = new Blob([response.data], {
                        type: 'image/jpeg'
                    });
                    const link = document.createElement('a');
                    link.href = URL.createObjectURL(blob);
                    const timestamp = new Date().toISOString().replace(/[-:]/g, '_').replace(/\..+/, '');
                    link.download = `${timestamp}_capture.jpg`;
                    link.click();
                    // console.log(blob);
                    // const imageUrl = URL.createObjectURL(blob);
                    // console.log(imageUrl);
                    // this.$refs.showResultImage.src = imageUrl;
                    // console.log(this.$refs.showResultImage.src);
                } else {
                    this.$Message.error('测试错误，无图像！');
                }
            } catch (error) {
                console.error('测试预制信息时出错:', error);
                this.$Message.error('无图像！');
            }
        },
        // 初始化温度曲线图
        initChart() {
            const chartDom = document.getElementById('temperature-chart');
            const myChart = echarts.init(chartDom);
            const option = {
                title: {
                    text: '温度曲线图'
                },
                tooltip: {
                    trigger: 'axis'
                },
                legend: {
                    data: ['最大温度', '最小温度', '平均温度']
                },
                xAxis: {
                    type: 'category',
                    data: Array.from({
                        length: this.list_maxThermals.length
                    }, (_, i) => i + 1)
                },
                yAxis: {
                    type: 'value',
                    axisLabel: {
                        formatter: '{value} °C'
                    }
                },
                series: [{
                        name: '最大温度',
                        type: 'line',
                        data: this.list_maxThermals
                    },
                    {
                        name: '最小温度',
                        type: 'line',
                        data: this.list_minThermals
                    },
                    {
                        name: '平均温度',
                        type: 'line',
                        data: this.list_avgThermals
                    }
                ]
            };
            myChart.setOption(option);
        },
        // 滚动条滚动
        handleScroll(value) {
            this.scrollPosition = value;
        },
        // 显示大图
        showLargeImage(image) {
            // 这里可以添加点击图片后的放大显示逻辑
            console.log('显示图片:', image.id);
        },
        // 计算滚动条最大值
        calculateMaxScroll() {
            const containerHeight = 400; // 容器高度
            const totalImagesHeight = Math.ceil(this.images.length / 2) * 200; // 每行2张图，每张图高200px
            this.maxScroll = Math.max(0, totalImagesHeight - containerHeight);
        },
        // 读取测温过程图像
        async readThermalImage(id) {
            if (id < 0 || id > 19) {
                this.$Message.error('图像编号超出范围');
                return;
            }

            const host = window.location.hostname;
            const port = '8010';
            const url_send = `http://${host}:${port}/api/task/thermal_image?id=${id}`;

            try {
                const response = await axios.get(url_send, {
                    responseType: 'arraybuffer'
                });
                const contentType = response.headers['content-type'];

                if (contentType && contentType.includes('image/jpeg')) {
                    const blob = new Blob([response.data], {
                        type: 'image/jpeg'
                    });
                    const imageUrl = URL.createObjectURL(blob);
                    this.images[id].src = imageUrl;
                } else {
                    this.$Message.error(`获取图像${id}失败`);
                }
            } catch (error) {
                console.error('读取测温过程图像时出错:', error);
                this.$Message.error(`读取图像${id}时出错`);
            }
        },
        // 开始录像
        async startRecording() {
            const host = window.location.hostname;
            const port = '8010';
            const url_send = `http://${host}:${port}/api/camera/start_recording`;

            try {
                const response = await axios.put(url_send);
                console.log(response.data);
                if (response.data.status === 'success') {
                    this.$Message.success('开始录像');
                    this.isSaveVideo = true;
                    this.message = '开始录像';
                    setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
                } else {
                    this.$Message.error('开始录像失败');
                }
            } catch (error) {
                console.error('开始录像时出错:', error);
                this.$Message.error('开始录像时出错');
            }
        },
        // 停止录像
        async stopRecording() {
            const host = window.location.hostname;
            const port = '8010';
            const url_send = `http://${host}:${port}/api/camera/stop_recording`;
            this.isSaveVideo = false;

            try {
                const response = await axios.put(url_send);
                console.log(response.data);
                if (response.data.status === 'success') {
                    this.$Message.success('停止录像');
                    this.message = '停止录像,视频名称: ' + response.data.file_name;
                    setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
                } else {
                    this.$Message.error('停止录像失败');
                    this.isSaveVideo = false;
                }
            } catch (error) {
                console.error('停止录像时出错:', error);
                this.$Message.error('停止录像时出错');
            }
        },
        // 发送保存心跳
        async sendSaveHeartbeat() {
            if (this.isSaveVideo) {
                const host = window.location.hostname;
                const port = '8010';
                const url_send = `http://${host}:${port}/api/camera/recording_heartbeat`;
                try {
                    const response = await axios.put(url_send);
                    // console.log(response.data);
                } catch (error) {
                    console.error('发送保存心跳时出错:', error);
                    this.$Message.error('发送保存心跳时出错');
                }
            }
        }
    }
}
</script>

<style lang="less" scoped>
#temperature-chart {
    width: 100%;
    height: 400px;
}

.iframe-container {
    position: relative;
    width: 100%;
    padding-bottom: 56.25%;
    /* 16:9 aspect ratio */
    height: 0;
    overflow: hidden;
}

iframe {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    border: none;
}

.image-container {
    position: relative;
    height: 400px;
    overflow: auto;
}

.image-grid {
    display: flex;
    flex-wrap: wrap;
    width: 100%;
}

.grid-image {
    width: calc(50% - 10px);
    height: 190px;
    margin: 5px;
    object-fit: cover;
    cursor: pointer;
}

.scroll-bar {
    position: absolute;
    right: 0;
    top: 10px;
    width: 20px;
    height: calc(100% - 20px);
    display: flex;
    align-items: center;
    justify-content: center;
}
</style>
