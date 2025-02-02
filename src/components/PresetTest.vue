<template>
    <Grid square :col=2>
        <GridItem>
            <iframe id="webrtc-iframe" :src="iframeSrc" frameborder="0" allowfullscreen></iframe>
        </GridItem>
        <GridItem>
            <div>
                <Grid :border="border" :col=2>
                    <GridItem>
                        <div>
                            <label>选择预制信息:</label>
                            <Select name="PresetSelect" v-model="Preset_Select" @on-change="PresetDataChange_Select">
                                <Option v-for="preset in presetList" :key="preset.id" :value="preset.id">{{ preset.preset_name }}</Option>
                            </Select>
                        </div>
                        <div>
                            <label>预制信息编号</label>
                            <Input name="PresetInput" type="number" v-model="Preset_Input" @on-change="PresetDataChange_Input" />
                        </div>
                        <div>
                            <!-- 增加换行的间隙 -->
                            <br>
                            <Button type="info" @click="testPreset">测试预制信息</Button>
                        </div>
                        <div>
                            <br>
                            <Button type="info" @click="on_capture">拍照</Button>
                        </div>
                        <div>
                            <br>
                            <Button type="info" @click="startRecording">开始录像</Button>
                            <Button type="info" @click="stopRecording">停止录像</Button>
                        </div>
                    </GridItem>
                    <GridItem>
                        <div>
                            <Grid border="false" square :col=3>
                                <GridItem></GridItem>
                                <GridItem>
                                    <Button type="info" @mousedown="PTZ_Up" @mouseup="PTZ_Stop">上</Button>
                                </GridItem>
                                <GridItem></GridItem>
                                <GridItem>
                                    <Button type="info" @mousedown="PTZ_Left" @mouseup="PTZ_Stop">左</Button>
                                </GridItem>
                                <GridItem>
                                    <Button type="info" @click="PTZ_Stop">停</Button>
                                </GridItem>
                                <GridItem>
                                    <Button type="info" @mousedown="PTZ_Right" @mouseup="PTZ_Stop">右</Button>
                                </GridItem>
                                <GridItem></GridItem>
                                <GridItem>
                                    <Button type="info" @mousedown="PTZ_Down" @mouseup="PTZ_Stop">下</Button>
                                </GridItem>
                                <GridItem></GridItem>
                            </Grid>
                        </div>
                        <div>
                            <br>
                            <label>云台速度:</label>
                            <Slider v-model="ptz_speed" show-input></Slider>
                        </div>
                        <div>
                            <label>垂直位置:</label>
                            <Input type="number" v-model="elevation" placeholder="云台高度" style="display: inline-block; width: 120px; margin-right: 10px;" />
                            <label>水平位置:</label>
                            <Input type="number" v-model="azimuth" placeholder="云台方位" style="display: inline-block; width: 120px;" />
                            <br>
                            <label>变焦:</label>
                            <Input type="number" v-model="absoluteZoom" placeholder="绝对变焦" style="display: inline-block; width: 120px; margin-right: 10px;" />
                            <label>焦距:</label>
                            <Input type="number" v-model="focus" placeholder="绝对焦距" style="display: inline-block; width: 120px; margin-right: 10px;" />
                            <br>
                            <label>水平速度:</label>
                            <Input type="number" v-model="horizontalSpeed" placeholder="水平速度" style="display: inline-block; width: 120px; margin-right: 10px;" />
                            <label>垂直速度:</label>
                            <Input type="number" v-model="verticalSpeed" placeholder="垂直速度" style="display: inline-block; width: 120px;" />
                            <br>
                            <Button type="info" @click="PTZ_Set">设置云台</Button>
                        </div>
                    </GridItem>
                </Grid>
            </div>
        </GridItem>
        <GridItem>
            <div>
                <label>测试结果</label>
            </div>
            <div>
                <img ref="showResultImage" name="showResultImage_Image" style="width: 200%; height: auto;" />
            </div>
        </GridItem>
    </Grid>
    <FooterToolbar>
        <label>当前云台位置:</label>
        <label>高度: {{ ptzCurrentElevation }}</label>
        <label>方位: {{ ptzCurrentAzimuth }}</label>
        <label>变焦: {{ ptzCurrentAbsoluteZoom }}</label>
        <label>焦距: {{ ptzCurrentFocus }}</label>
        <label>焦距: {{ ptzCurrentFocalLen }}</label>
    </FooterToolbar>
</template>

<script>
import axios from 'axios';
import {Slider, Input, Select, Button, List, ListItem, FooterToolbar } from 'view-ui-plus';

export default {
    components: {
        List,
        ListItem,
        Button,
        Select,
        Input,
        FooterToolbar,
    },
    data() {
        return {
            presetList: [],
            Preset_Select: null,    // 选择预制信息 Select组件
            Preset_Input: null,     // 选择预制信息 Input组件
            ptz_speed: 25,          // 云台速度
            elevation: 0,           // 云台高度
            azimuth: 0,            // 云台方位
            absoluteZoom: 1.0,     // 绝对变焦
            focus: 27000,          // 绝对焦距
            focalLen: 22,          // 焦距
            horizontalSpeed: 25,  // 水平速度
            verticalSpeed: 25,    // 垂直速度
            ptzCurrentElevation: 0, // 当前云台高度
            ptzCurrentAzimuth: 0,   // 当前云台方位
            ptzCurrentAbsoluteZoom: 1.0, // 当前绝对变焦
            ptzCurrentFocus: 27000, // 当前绝对焦距
            ptzCurrentFocalLen: 22, // 当前焦距
            timer: null, // 添加定时器变量
            timer_heartbeat: null,  // 添加心跳定时器变量
            isSaveVideo: false, // 是否保存录像
        };
    },
    mounted() {
        this.fetchPresetList()
        this.initIframe('mppstream');
        // 启动定时获取云台位置
        this.timer = setInterval(this.PTZ_Get, 2000); // 每秒更新一次
        this.timer_heartbeat = setInterval(this.sendSaveHeartbeat, 3000);
    },
    beforeDestroy() {
        // 组件销毁前清除定时器
        if (this.timer) {
            clearInterval(this.timer);
        }
        if (this.timer_heartbeat) {
            clearInterval(this.timer_heartbeat);
        }
    },
    methods: {
        initIframe(stream) {
            const host = window.location.hostname;
            this.iframeSrc = `http://${host}:8889/${stream}`;
            const iframe = document.getElementById('webrtc-iframe');
            iframe.style.width = '100%';
            iframe.style.height = '100%';
        },
        switchStream(stream) {
            this.initIframe(stream);
        },
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
        PresetDataChange_Select() {
            // console.log(this.Preset_Select);
            this.Preset_Input = parseInt(this.Preset_Select);
            // console.log(this.Preset_Input);
        },
        async testPreset(){
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/task/test?preset_id=${this.Preset_Input}`;
            try {
                const response = await axios.get(url_send, { responseType: 'arraybuffer' });
                const contentType = response.headers['content-type'];
                console.log(contentType)
                if (contentType && contentType.includes('image/jpeg')) {
                    console.log('测试预制信息成功');
                    const blob = new Blob([response.data], { type: 'image/jpeg' });
                    console.log(blob);
                    const imageUrl = URL.createObjectURL(blob);
                    console.log(imageUrl);
                    this.$refs.showResultImage.src = imageUrl;
                    console.log(this.$refs.showResultImage.src);
                } else {
                    this.$Message.error('测试错误，无图像！');
                }
            } catch (error) {
                console.error('测试预制信息时出错:', error);
                this.$Message.error('测试错误，无图像！');
            }
        },
        async PTZ_Up() {
            console.log('PTZ_Up');
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/ptz/up?speed=${this.ptz_speed}`;
            try {
                const response = await axios.put(url_send);
                console.log(response.data);
            } catch (error) {
                console.error('Error:', error);
            }
        },
        async PTZ_Left() {
            console.log('PTZ_Left');
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/ptz/left?speed=${this.ptz_speed}`;
            try {
                const response = await axios.put(url_send);
                console.log(response.data);
            } catch (error) {
                console.error('Error:', error);
            }
        },
        async PTZ_Stop() {
            console.log('PTZ_Stop');
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/ptz/stop`;
            try {
                const response = await axios.put(url_send);
                console.log(response.data);
            } catch (error) {
                console.error('Error:', error);
            }
        },
        async PTZ_Right() {
            console.log('PTZ_Right');
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/ptz/right?speed=${this.ptz_speed}`;
            try {
                const response = await axios.put(url_send);
                console.log(response.data);
            } catch (error) {
                console.error('Error:', error);
            }
        },
        async PTZ_Down() {
            console.log('PTZ_Down');
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/ptz/down?speed=${this.ptz_speed}`;
            try {
                const response = await axios.put(url_send);
                console.log(response.data);
            } catch (error) {
                console.error('Error:', error);
            }
        },
        async PTZ_Set() {
            console.log('PTZ_Set');
            const host = window.location.hostname;
            const port = '8010';
            const url_send = `http://${host}:${port}/api/ptz/set_position`;
            
            const params = {
                elevation: this.elevation,
                azimuth: this.azimuth,
                absoluteZoom: this.absoluteZoom,
                focus: this.focus,
                focalLen: this.focalLen,
                horizontalSpeed: this.horizontalSpeed,
                verticalSpeed: this.verticalSpeed
            };

            try {
                const response = await axios.put(url_send, params);
                console.log(response.data);
                this.$Message.success('云台位置设置成功');
            } catch (error) {
                console.error('Error:', error);
                this.$Message.error('云台位置设置失败');
            }
        },
        async PTZ_Get() {
            const host = window.location.hostname;
            const port = '8010';
            const url_send = `http://${host}:${port}/api/ptz/get_position`;
            
            try {
                const response = await axios.get(url_send);
                // 更新云台位置数据
                this.ptzCurrentElevation = response.data.elevation;
                this.ptzCurrentAzimuth = response.data.azimuth;
                this.ptzCurrentAbsoluteZoom = response.data.absoluteZoom;
                this.ptzCurrentFocus = response.data.focus;
                this.ptzCurrentFocalLen = response.data.focalLen;
            } catch (error) {
                console.error('获取云台位置失败:', error);
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
    },
}
</script>

<style lang="less" scoped>

    .iframe-container {
    position: relative;
    width: 100%;
    padding-bottom: 56.25%; /* 16:9 aspect ratio */
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
</style>
