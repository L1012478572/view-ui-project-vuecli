<template>
    <div>
        <h1>云台参数设置</h1>
    </div>
    <div class="centered">
        <List>
            <ListItem>
                <div class="form-item">
                    <label for="class">云台类型:</label>
                    <input type="number" v-model="ptzClass" id="class" step="1" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="ip">云台IP地址:</label>
                    <input type="text" v-model="ip" id="ip" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="username">云台用户名:</label>
                    <input type="text" v-model="username" id="username" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="password">云台密码:</label>
                    <input type="text" v-model="password" id="password" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="need_ir_ip">是否需要红外设备IP地址:</label>
                    <input type="number" v-model="need_ir_ip" id="need_ir_ip" step="1" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="ir_ip">红外设备IP地址:</label>
                    <input type="text" v-model="ir_ip" id="ir_ip" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="username_ir">红外设备用户名:</label>
                    <input type="text" v-model="username_ir" id="username_ir" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="password_ir">红外设备密码:</label>
                    <input type="text" v-model="password_ir" id="password_ir" />
                </div>
            </ListItem>
            <ListItem>
                <Button type="info"  @click="saveSettings">保存</Button>
                <Button type="success"  @click="loadSettings">读取</Button>
            </ListItem>
            <ListItem>
                <p v-if="message" class="message">{{ message }}</p>
            </ListItem>
        </List>
    </div>
</template>

<script>
import axios from 'axios';
import { List, ListItem } from 'view-ui-plus';

export default {
    components: {
        List,
        ListItem
    },
    data() {
        return {
            ptzClass: 1,
            ip: '192.168.xx.xx',
            username: 'admin',
            password: 'Admin123',
            need_ir_ip: 1,
            ir_ip: '192.168.xx.xx',
            username_ir: 'admin',
            password_ir: 'Admin123',
            message: '' // 添加一个状态变量来存储提示消息
        };
    },
    methods: {
        async saveSettings() {
            console.log('saveSettings called');
            const settings = {
                ptzClass: this.ptzClass,
                ip: this.ip,
                username: this.username,
                password: this.password,
                need_ir_ip: this.need_ir_ip,
                ir_ip: this.ir_ip,
                username_ir: this.username_ir,
                password_ir: this.password_ir
            };
            console.log('Saving settings', settings);

            // 创建XML结构
            const xmlBuilder = new XMLSerializer();
            const xmlDoc = document.implementation.createDocument('', '', null);
            const root = xmlDoc.createElement('Yuntai');
            for (const key in settings) {
                const element = xmlDoc.createElement(key);
                element.textContent = settings[key];
                root.appendChild(element);
            }
            xmlDoc.appendChild(root);

            // 增加换行和缩进
            const xmlString = xmlBuilder.serializeToString(xmlDoc);
            const formattedXmlString = xmlString.replace(/(>)(<)(\/*)/g, '$1\n$2$3');

            try {
                const host = window.location.hostname;
                const port = '8010'; // 你的后端端口号
                const url = `http://${host}:${port}/api/param/ptzinfo`;
                console.log(`Sending PUT request to ${url}`);
                const response = await axios.put(url, formattedXmlString, {
                    headers: {
                        'Content-Type': 'application/xml'
                    }
                });
                console.log('Response received:', response);

                this.message = '设置已保存'; // 更新提示消息
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            } catch (error) {
                console.error('Error saving settings:', error);
                this.message = '保存设置失败'; // 更新提示消息
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            }
        },
        async loadSettings() {
            console.log('loadSettings called');
            try {
                const host = window.location.hostname;
                const port = '8010'; // 你的后端端口号
                const url = `http://${host}:${port}/api/param/ptzinfo`;
                console.log(`Sending request to ${url}`);
                const response = await axios.get(url);
                console.log('Response received:', response);

                const parser = new DOMParser();
                const xmlDoc = parser.parseFromString(response.data, 'application/xml');

                this.ptzClass = parseInt(xmlDoc.getElementsByTagName('class')[0].textContent);
                this.ip = xmlDoc.getElementsByTagName('ip')[0].textContent;
                this.username = xmlDoc.getElementsByTagName('username')[0].textContent;
                this.password = xmlDoc.getElementsByTagName('password')[0].textContent;
                this.need_ir_ip = parseInt(xmlDoc.getElementsByTagName('need_ir_ip')[0].textContent);
                this.ir_ip = xmlDoc.getElementsByTagName('ir_ip')[0].textContent;
                this.username_ir = xmlDoc.getElementsByTagName('username_ir')[0].textContent;
                this.password_ir = xmlDoc.getElementsByTagName('password_ir')[0].textContent;

                this.message = '设置已读取'; // 更新提示消息
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            } catch (error) {
                console.error('Error loading settings:', error);
                this.message = '读取设置失败'; // 更新提示消息
                setTimeout(() => this.message = '', 3000); // 3秒后清除提示消息
            }
        }
    },
    mounted() {
        this.loadSettings(); // 在组件挂载时调用loadSettings
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

.form-item {
    display: flex;
    align-items: center;
    margin-bottom: 10px;
}

label {
    width: 150px;
    text-align: right;
    margin-right: 10px;
}

input {
    flex: 1;
    user-select: text; /* 允许复制 */
}

button {
    margin-right: 10px;
}

.message {
    color: green;
    margin-top: 10px;
}
</style>