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
</template>

<script>
import axios from 'axios';
import { Input, Select, Button, List, ListItem } from 'view-ui-plus';

export default {
    components: {
        List,
        ListItem,
        Button,
        Select,
        Input,
    },
    data() {
        return {
            presetList: [],
            Preset_Select: null,    // 选择预制信息 Select组件
            Preset_Input: null,     // 选择预制信息 Input组件
        };
    },
    mounted() {
        this.fetchPresetList()
        this.initIframe('mppstream');
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
                setTimeout(() => this.message = '', 3000); // 3��后清除提示消息
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
        PTZ_Up() {
            console.log('PTZ_Up');
        },
        PTZ_Left() {
            console.log('PTZ_Left');
        },
        PTZ_Stop() {
            console.log('PTZ_Stop');
        },
        PTZ_Right() {
            console.log('PTZ_Right');
        },
        PTZ_Down() {
            console.log('PTZ_Down');
        },
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
