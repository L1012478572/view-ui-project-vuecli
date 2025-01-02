<template>
    <div class="modal">
        <div class="modal-content">
            <span class="close" @click="$emit('close')">&times;</span>
            <h2>修改预制信息</h2>
            <List>
                <ListItem>
                    <p>ID: {{ presetId }}</p>
                </ListItem>
                <ListItem>
                    <div class="form-item">
                        <label>预制名称:</label>
                        <Input v-model="presetInfo.preset_name" type="text" />
                    </div>
                </ListItem>
                <ListItem>
                    <div class="form-item">
                        <label>模型名称:</label>
                        <Select v-model="presetInfo.model_name">
                            <Option v-for="model in modelList" :key="model.id" :value="model.file_name">{{ model.file_name }}</Option>
                        </Select>
                    </div>
                </ListItem>
                <ListItem>
                    <div class="form-item">
                        <label>红外模型名称:</label>
                        <Select v-model="presetInfo.ir_model_name">
                            <Option v-for="model in modelList" :key="model.id" :value="model.file_name">{{ model.file_name }}</Option>
                        </Select>
                    </div>
                </ListItem>
                <ListItem v-for="(item, index) in presetInfo.classes" :key="index">
                    <div class="form-item" style="display: flex; align-items: center;">
                        <div style="display: flex; align-items: center; margin-right: 20px;">
                            <label style="margin-right: 10px;">分类名称:</label>
                            <Input v-model="item.value" type="text" />
                        </div>
                        <div style="display: flex; align-items: center; margin-right: 20px;">
                            <label style="margin-right: 10px;">分类ID:</label>
                            <Input v-model="item.id" type="number" />
                        </div>
                        <div style="display: flex; align-items: center;">
                            <label style="margin-right: 10px;">分数:</label>
                            <Input v-model="presetInfo.score[index].value" type="number" step="0.1" />
                        </div>
                    </div>
                </ListItem>
                <ListItem>
                    <Button type="info" @click="addClass">增加分类</Button>
                    <Button type="error" @click="removeClass">删除分类</Button>
                </ListItem>
                <ListItem>
                    <div class="form-item" style="display: flex; align-items: center;">
                        <div style="display: flex; align-items: center; margin-right: 20px;">
                            <label style="margin-right: 10px;">位置X:</label>
                            <Input v-model="presetInfo.location.x" type="number" />
                        </div>
                        <div style="display: flex; align-items: center;">
                            <label style="margin-right: 10px;">位置Y:</label>
                            <Input v-model="presetInfo.location.y" type="number" />
                        </div>
                    </div>
                </ListItem>
                <ListItem>
                    <div class="form-item">
                        <label>图像通道:</label>
                        <Select v-model="presetInfo.image_channel">
                            <Option value="0">高清</Option>
                            <Option value="1">红外</Option>
                        </Select>
                    </div>
                </ListItem>
                <ListItem>
                    <div class="form-item">
                        <label>是否运行红外模型:</label>
                        <Select v-model="presetInfo.isRunIrModel">
                            <Option value="0">不开启</Option>
                            <Option value="1">开启</Option>
                        </Select>
                    </div>
                </ListItem>
                <ListItem>
                    <div class="form-item">
                        <label>是否控制云台:</label>
                        <Select v-model="presetInfo.isControlPtz">
                            <Option value="0">不开启</Option>
                            <Option value="1">开启</Option>
                        </Select>
                    </div>
                </ListItem>
                <ListItem>
                    <div class="form-item">
                        <label>测温次数:</label>
                        <Input v-model="presetInfo.tempture_num" type="number" />
                    </div>
                </ListItem>
                <ListItem>
                    <div class="form-item">
                        <label>识别目标ID:</label>
                        <Input v-model="presetInfo.object_id" type="number" />
                        <label>测温目标ID:</label>
                        <Input v-model="presetInfo.ir_object_id" type="number" />
                    </div>
                </ListItem>
                <ListItem>
                    <Button type="info" @click="applyChanges">应用</Button>
                    <Button type="success" @click="addNewPreset">新增</Button>
                </ListItem>
                <ListItem>
                    <p v-if="message" class="message">{{ message }}</p>
                </ListItem>
            </List>
        </div>
    </div>
</template>

<script>
import axios from 'axios';
import {Input, Select, Option, Button, List, ListItem } from 'view-ui-plus';

export default {
    components: {
        List,
        ListItem,
        Button
    },
    props: {
        presetId: {
            type: Number,
            required: true
        }
    },
    data() {
        return {
            modelList: [],
            presetInfo: {
                preset_name: '',
                model_name: '',
                ir_model_name: '',
                classes: [],
                score: [],
                location: { x: 0, y: 0 },
                image_channel: 0,
                isRunIrModel: 0,
                isControlPtz: 0,
                tempture_num: 0,
                object_id: 0,
                ir_object_id: 0
            },
            message: '' // 添加一个状态变量来存储提示消息
        };
    },
    methods: {
        fetchModelList() {
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/model/list`;
            fetch(url_send)
                .then(response => response.text())
                .then(xml => {
                    console.log(xml);
                    const parser = new DOMParser();
                    const xmlDoc = parser.parseFromString(xml, 'text/xml');
                    const items = xmlDoc.getElementsByTagName('Item');
                    this.modelList = Array.from(items).map(item => ({
                        id: item.getAttribute('id'),
                        file_name: item.getAttribute('name')
                    }));
                    console.log(this.modelList)
                });
        },
        fetchPresetInfo() {
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/param/presetInfo?id=${this.presetId}`;
            fetch(url_send)
                .then(response => response.text())
                .then(xml => {
                    console.log(xml);
                    const parser = new DOMParser();
                    const xmlDoc = parser.parseFromString(xml, 'text/xml');
                    const preset = xmlDoc.getElementsByTagName('Preset')[0];
                    this.presetInfo.preset_name = preset.getElementsByTagName('preset_name')[0].textContent;
                    this.presetInfo.model_name = preset.getElementsByTagName('model_name')[0].textContent;
                    this.presetInfo.ir_model_name = preset.getElementsByTagName('ir_model_name')[0].textContent;
                    this.presetInfo.classes = Array.from(preset.getElementsByTagName('classes')[0].getElementsByTagName('Item')).map(item => ({
                        id: item.getAttribute('id'),
                        value: item.getAttribute('value')
                    }));
                    console.log(this.presetInfo.classes);
                    this.presetInfo.score = Array.from(preset.getElementsByTagName('score')[0].getElementsByTagName('Item')).map(item => ({
                        value: item.getAttribute('value')
                    }));
                    const location = preset.getElementsByTagName('location')[0].getElementsByTagName('Item')[0];
                    this.presetInfo.location.x = location.getAttribute('x');
                    this.presetInfo.location.y = location.getAttribute('y');
                    this.presetInfo.image_channel = preset.getElementsByTagName('image_channel')[0].textContent;
                    this.presetInfo.isRunIrModel = preset.getElementsByTagName('isRunIrModel')[0].textContent;
                    this.presetInfo.isControlPtz = preset.getElementsByTagName('isControlPtz')[0].textContent;
                    this.presetInfo.tempture_num = preset.getElementsByTagName('tempture_num')[0].textContent;
                    this.presetInfo.object_id = preset.getElementsByTagName('object_id')[0].textContent;
                    this.presetInfo.ir_object_id = preset.getElementsByTagName('ir_object_id')[0].textContent;
                });
        },
        addClass() {
            this.presetInfo.classes.push({ id: '', value: '' });
            this.presetInfo.score.push({ value: '' });
        },
        removeClass() {
            if (this.presetInfo.classes.length > 0) {
                this.presetInfo.classes.pop();
                this.presetInfo.score.pop();
            }
        },
        editPreset(id) {
            // 修改预制信息功能暂留
            console.log('修改预制信息', id);
            this.$emit('open-modal', id);
        },
        applyChanges() {
            // 应用按钮��接口
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/param/presetlist/update`;

            const xmlData = `<?xml version="1.0" encoding="UTF-8"?>
                <Preset>
                    <preset_name>${this.presetInfo.preset_name}</preset_name>
                    <model_path>${this.presetInfo.model_name}</model_path>
                    <ir_model_path>${this.presetInfo.ir_model_name}</ir_model_path>
                    <classes>
                        ${this.presetInfo.classes.map(item => `<Item id="${item.id}" value="${item.value}" />`).join('')}
                    </classes>
                    <score>
                        ${this.presetInfo.score.map(item => `<Item value="${item.value}" />`).join('')}
                    </score>
                    <location>
                        <Item x="${this.presetInfo.location.x}" y="${this.presetInfo.location.y}" />
                    </location>
                    <image_channel>${this.presetInfo.image_channel}</image_channel>
                    <isRunIrModel>${this.presetInfo.isRunIrModel}</isRunIrModel>
                    <isControlPtz>${this.presetInfo.isControlPtz}</isControlPtz>
                    <tempture_num>${this.presetInfo.tempture_num}</tempture_num>
                    <object_id>${this.presetInfo.object_id}</object_id>
                    <ir_object_id>${this.presetInfo.ir_object_id}</ir_object_id>
                </Preset>
            `;

            axios.put(url_send, xmlData, {
                headers: {
                    'Content-Type': 'application/xml'
                },
                params: {
                    id: this.presetId
                }
            })
            .then(response => {
                console.log('预制信息更新成功', response);
                this.message = '预制信息更新成功'; // 更新提示消息
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            })
            .catch(error => {
                console.error('预制信息更新失败', error);
                this.message = '预制信息更新失败'; // 更新提示消息
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            });
        },
        addNewPreset() {
            // 新增按钮的接口
            const host = window.location.hostname;
            const port = '8010'; // 你的后端端口号
            const url_send = `http://${host}:${port}/api/param/presetlist/add`;

            const xmlData = `<?xml version="1.0" encoding="UTF-8"?>
                <Preset>
                    <preset_name>${this.presetInfo.preset_name}</preset_name>
                    <model_path>${this.presetInfo.model_name}</model_path>
                    <ir_model_path>${this.presetInfo.ir_model_name}</ir_model_path>
                    <classes>
                        ${this.presetInfo.classes.map(item => `<Item id="${item.id}" value="${item.value}" />`).join('')}
                    </classes>
                    <score>
                        ${this.presetInfo.score.map(item => `<Item value="${item.value}" />`).join('')}
                    </score>
                    <location>
                        <Item x="${this.presetInfo.location.x}" y="${this.presetInfo.location.y}" />
                    </location>
                    <image_channel>${this.presetInfo.image_channel}</image_channel>
                    <isRunIrModel>${this.presetInfo.isRunIrModel}</isRunIrModel>
                    <isControlPtz>${this.presetInfo.isControlPtz}</isControlPtz>
                    <tempture_num>${this.presetInfo.tempture_num}</tempture_num>
                    <object_id>${this.presetInfo.object_id}</object_id>
                    <ir_object_id>${this.presetInfo.ir_object_id}</ir_object_id>
                </Preset>
            `;

            axios.put(url_send, xmlData, {
                headers: {
                    'Content-Type': 'application/xml'
                }
            })
            .then(response => {
                console.log('预制信息添加成功', response);
                this.message = '预制信息添加成功'; // 更新提示消息
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
                this.$emit('close'); // 关闭当前弹窗
            })
            .catch(error => {
                console.error('预制信息添加失败', error);
                this.message = '预制信息添加失败'; // 更新提示消息
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            });
        }
    },
    mounted() {
        this.fetchModelList();
        this.fetchPresetInfo();
    }
};
</script>

<style scoped>
.modal {
    display: flex;
    justify-content: center;
    align-items: center;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
}
.modal-content {
    border: 1px solid #4710109d;
    background-color: white;
    padding: 20px;
    border-radius: 5px;
    width: 800px;
}
.close {
    float: right;
    font-size: 24px;
    cursor: pointer;
}
.modal-footer {
    display: flex;
    justify-content: flex-end;
    margin-top: 20px;
}
.modal-footer button {
    margin-left: 10px;
}
.message {
    color: green;
    margin-top: 10px;
}
</style>