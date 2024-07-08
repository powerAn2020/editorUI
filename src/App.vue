<template>
  <van-config-provider :theme="theme ? 'dark' : 'light'">
    <van-nav-bar title="Editor For KSU">
    </van-nav-bar>
    <van-space>
      <van-button type="primary" size="mini" icon="replay" plain @click="reload()">重载</van-button>
      <van-button type="primary" size="mini" icon="plus" plain @click="update()">保存</van-button>
      <van-button type="primary" size="mini" icon="todo-list-o" plain @click="reload()">加载日志</van-button>
      <van-button type="primary" size="mini" icon="replay" plain>卸载保留数据</van-button>
    </van-space>
    <Codemirror v-model:value="code" :options="cmOptions" border ref="cmRef" height="100vh" width="100%"
      @change="onChange" @input="onInput" @ready="onReady">
    </Codemirror>

  </van-config-provider>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import { exec } from 'kernelsu';
import "codemirror/mode/shell/shell"
import Codemirror from "codemirror-editor-vue3"

const theme = ref();
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

//读取定时任务信息
execCmd(`cat /data/adb/crond/root`).then(v => {

});
//读取执行日志
execCmd(`cat /data/adb/crond/run.log`).then(v => {

});
const code = ref(
  `0 */12 * * * echo 111
`
)
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
</script>

<style>
.van-theme-dark body {
  color: #f5f5f5;
  background-color: black;
}
</style>
