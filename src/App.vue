<template>
  <Menu mode="horizontal" :theme="theme" active-name="1">
    <MenuItem name="1">
          <!-- <Icon type="ios-paper" /> -->
          运行
    </MenuItem>
    <MenuItem name="2" @click="currentView = 'PresetTest'">
        测试
    </MenuItem>
    <Submenu name="3">
        <template #title>
            <Icon type="ios-stats" />
            参数配置
        </template>
        <MenuGroup title="算法设置">
            <MenuItem name="2-1" @click="currentView = 'PresetConfig'">预制信息设置</MenuItem>
            <MenuItem name="2-2" @click="currentView = 'ModelConfig'">模型管理</MenuItem>
        </MenuGroup>
        <MenuGroup title="系统设置">
            <MenuItem name="2-3" @click="currentView = 'PTZConfig'">云台设置</MenuItem>
            <MenuItem name="2-4" @click="currentView = 'ImageOffset'">图像偏移设置</MenuItem>
            <MenuItem name="2-5" @click="currentView = 'PTZMoveConfig'">云台控制参数设置</MenuItem>
        </MenuGroup>
    </Submenu>
  </Menu>
  <br>
  <!-- <p>Change theme</p> -->
  <!-- <RadioGroup v-model="theme">
      <Radio label="light"></Radio>
      <Radio label="dark"></Radio>
      <Radio label="primary"></Radio>
  </RadioGroup> -->
  <component :is="currentViewComponent"></component>
</template>

<script>
import { MenuItem } from 'view-ui-plus';
import PTZConfig from './components/PTZConfig.vue';
import ImageOffset from './components/ImageOffset.vue';
import PTZMoveConfig from './components/PTZMoveConfig.vue';
import ModelConfig from './components/ModelConfig.vue';
import PresetConfig from './components/PresetConfig.vue';
import PresetTest from './components/PresetTest.vue';

export default {
  name: 'App',
  components: {
        ImageOffset,
        PTZConfig,
        PTZMoveConfig,
        ModelConfig,
        PresetConfig,
        PresetTest,
        // WebrtcPlayer
    },
  data () {
    return {
      theme: 'primary',
      currentView: 'PTZConfig'
    }
  },
  computed: {
        currentViewComponent() {
            switch (this.currentView) {
                case 'ImageOffset':
                    return 'ImageOffset';
                case 'PTZConfig':
                    return 'PTZConfig';
                case 'PTZMoveConfig':
                    return 'PTZMoveConfig';
                case 'ModelConfig':
                    return 'ModelConfig';
                case 'PresetConfig':
                    return 'PresetConfig';
                case 'PresetTest':
                    return 'PresetTest';
                default:
                    return null;
            }
        }
    }
}
</script>
