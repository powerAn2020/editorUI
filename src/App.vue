<template>
  <van-config-provider :theme="theme ? 'dark' : 'light'">
    <van-nav-bar title="Editor For KSU">
    </van-nav-bar>
    <van-space>
      <van-button type="primary" size="mini" icon="replay" plain @click="reload()">重载配置</van-button>
      <van-button type="primary" size="mini" icon="plus" plain @click="update()">保存配置</van-button>
      <van-button type="primary" size="mini" icon="todo-list-o" plain @click="loadLog()">加载日志</van-button>
      <!-- <van-button type="primary" size="mini" icon="replay" plain>卸载保留数据</van-button> -->
      <van-cell center title="卸载保留数据">
        <template #right-icon>
          <van-switch v-model="checked" size="22px" :loading="loading" />
        </template>
      </van-cell>
    </van-space>
    <Codemirror v-model:value="code" :options="cmOptions" border ref="cmRef" height="100vh" width="100%"
      @change="onChange" @input="onInput" @ready="onReady">
    </Codemirror>

  </van-config-provider>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue';
import { exec } from 'kernelsu';
import "codemirror/mode/shell/shell"
import Codemirror from "codemirror-editor-vue3"
import {Buffer} from 'buffer'

const theme = ref();
const checked = ref(false);
const loading = ref(false);
const code = ref(`
ls la
`)
const logFile='/data/adb/crond/run.log'
const rootCronFile='/data/adb/crond/root'
const execCmd = async (cmd) => {
  console.info(cmd)
  const { errno, stdout, stderr } = await exec(cmd, { cwd: '/' });
  if (errno === 0) {
    // success
    console.log(stdout);
    return stdout;
  } else {
    console.info(stderr)
  }
}
const readFile = async (filePath) => {
  return Buffer.from(await execCmd(`base64 -w 0 ${filePath}`), "base64").toString('utf-8')
}
const saveFile = (content, filePath) => {
  return execCmd(`echo ${Buffer.from(content).toString("base64")} | base64 -d > ${filePath}`)
}
execCmd('settings get secure ui_night_mode').then(v => {
  // 0 表示跟随系统设置，即当前模式与系统设置的主题模式相匹配。
  // 1 表示开启了 Dark Mode（夜间模式）。
  // 2 表示关闭了 Dark Mode（白天模式）。
  if (v == '1') {
    theme.value = true;
  } else if (v == '2') {
    theme.value = false;
  }
});


const cmRef = ref()
const cmOptions = {
  mode: "text/x-sh"
}
const onChange = (val, cm) => {
  console.log(val)
  console.log(cm.getValue())
  // /(((\d+,)+\d+|(\d+(\/|-)\d+)|\d+|\*) ?){5}/.test('*/5 4 * * 1-6')
}

const onInput = (val) => {
  console.log(val)
}

const onReady = (cm) => {
  console.log(cm.focus())
}

onMounted(() => {
  setTimeout(() => {
    cmRef.value?.refresh()
  }, 1000)

  // setTimeout(() => {
  //   cmRef.value?.resize('100%', '100%')
  // }, 2000)

  setTimeout(() => {
    cmRef.value?.cminstance.isClean()
  }, 3000)
})

onUnmounted(() => {
  cmRef.value?.destroy()
})
//重新加载配置
const reload = () => {
//读取定时任务信息
code.value=readFile(`${rootCronFile}`)
}
//更新配置
const update = () => {
//读取定时任务信息
saveFile(code.value,`${rootCronFile}`)
}
//读取配置文件
const loadLog = () => {
  code.value=readFile(`${logFile}`)

}
//开关切换
watch(checked, (theme, prevtheme) => {
  loading.value = true;
  let cmd = "touch /data/adb/crond/KEEP_ON_UNINSTALL"
  if (theme) {
    cmd = "rm -f /data/adb/crond/KEEP_ON_UNINSTALL"
  }
  // execCmd(cmd).then(v => {
  //   console.log(v)
  // });
  setTimeout(() => {
    loading.value = false;
  }, 1000);
}, {
  immediate: true
})
reload()
</script>

<style>
.van-theme-dark body {
  color: #f5f5f5;
  background-color: black;
}
</style>
