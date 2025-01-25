<template>
    <div>
        <h1>网络设置</h1>
    </div>
    <div class="centered">
        <label>eth0</label>
        <List>
            <ListItem>
                <div class="form-item">
                    <label for="eth0_ip">eth0 IP地址:</label>
                    <input type="text" v-model="eth0_ip" id="eth0_ip" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="eth0_mask">eth0 子网掩码:</label>
                    <input type="text" v-model="eth0_mask" id="eth0_mask" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="eth0_gateway">eth0 网关:</label>
                    <input type="text" v-model="eth0_gateway" id="eth0_gateway" />
                </div>
            </ListItem>
            <ListItem>
                <div class="form-item">
                    <label for="eth0_dhcp">eth0 是否启用DHCP:</label>
                    <input type="text" v-model="eth0_dhcp" id="eth0_dhcp" />
                </div>
            </ListItem>
        </List>
        <br>
        <label>eth1</label>
        <List>
            <ListItem>
                <label>eth1 IP地址:</label>
                <input type="text" v-model="eth1_ip" id="eth1_ip" />
            </ListItem>
            <ListItem>
                <label>eth1 子网掩码:</label>
                <input type="text" v-model="eth1_mask" id="eth1_mask" />
            </ListItem>
            <ListItem>
                <label>eth1 网关:</label>
                <input type="text" v-model="eth1_gateway" id="eth1_gateway" />
            </ListItem>
            <ListItem>
                <label>eth1 是否启用DHCP:</label>
                <input type="text" v-model="eth1_dhcp" id="eth1_dhcp" />
            </ListItem>
        </List>
        <br>
        <List>
            <ListItem>
                <Button type="success" @click="fetchNetWorkSet">读取</Button>
                <Button type="info" @click="saveNetWorkSet">保存</Button>
            </ListItem>
        </List>
        <List>
            <ListItem>
                <p v-if="message" class="message">{{ message }}</p>
            </ListItem>
        </List>
    </div>
</template>

<script>
import axios from 'axios';
import {Input, Select, Button, List, ListItem } from 'view-ui-plus';

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
            eth0_ip: '',
            eth0_mask: '',
            eth0_gateway: '',
            eth0_dhcp: '',
            eth1_ip: '',
            eth1_mask: '',
            eth1_gateway: '',
            eth1_dhcp: '',
            message: '',
        }
    },
    mounted() {
        this.fetchNetWorkSet();
    },
    methods: {
        async fetchNetWorkSet() {
            try {
                const host = window.location.hostname;
                const port = '8010';
                const url_send = `http://${host}:${port}/api/network/getpar`;
                const response = await axios.get(url_send);
                
                // 处理eth0数据
                if (response.data.eth0 && response.data.eth0[0]) {
                    const eth0 = response.data.eth0[0];
                    this.eth0_ip = eth0.ip_address;
                    this.eth0_mask = eth0.netmask;
                    this.eth0_gateway = eth0.gateway;
                    this.eth0_dhcp = eth0.mode;
                }
                
                // 处理eth1数据
                if (response.data.eth1 && response.data.eth1[0]) {
                    const eth1 = response.data.eth1[0];
                    this.eth1_ip = eth1.ip_address;
                    this.eth1_mask = eth1.netmask;
                    this.eth1_gateway = eth1.gateway;
                    this.eth1_dhcp = eth1.mode;
                }
                this.message = '网络设置读取成功';
            } catch (error) {
                console.error('获取网络设置失败:', error);
                this.message = '获取网络设置失败: ' + error.message;
            }
        },
        async saveNetWorkSet() {
            try {
                const host = window.location.hostname;
                const port = '8010';
                const url_send = `http://${host}:${port}/api/network/setpar`;
                
                const networkData = {
                    eth0: [{
                        ip_address: this.eth0_ip,
                        netmask: this.eth0_mask,
                        gateway: this.eth0_gateway,
                        mode: this.eth0_dhcp
                    }],
                    eth1: [{
                        ip_address: this.eth1_ip,
                        netmask: this.eth1_mask,
                        gateway: this.eth1_gateway,
                        mode: this.eth1_dhcp
                    }]
                };

                await axios.post(url_send, networkData);
                this.message = '网络设置保存成功';
            } catch (error) {
                console.error('保存网络设置失败:', error);
                this.message = '保存网络设置失败: ' + error.message;
            }
        }
    }
}
</script>

<style lang="less" scoped>
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

.message {
    margin-top: 10px;
    padding: 10px;
    border-radius: 4px;
    background-color: #f0f9eb;
    color: #67c23a;
    text-align: center;
    width: 100%;
}
</style>
