<template>
  <van-config-provider :theme="light">
    <van-sticky>
    <van-nav-bar title="Editor For KSU">
    </van-nav-bar>
    <van-cell center title="操作">
      <template #right-icon>
          <van-button type="primary" size="mini" icon="replay" plain @click="reload()">重载配置</van-button>
          <van-button type="primary" size="mini" icon="plus" plain @click="update"
            :disabled="disabled">保存配置</van-button>
          <van-button type="primary" size="mini" icon="todo-list-o" plain @click="loadLog">加载日志</van-button>
          <van-button type="primary" size="mini" icon="delete-o" plain @click="clearLog">清空日志</van-button>
      </template>

    </van-cell>

    <van-cell center title="卸载保留数据">
      <template #right-icon>
        <van-switch v-model="checked" size="22px" :loading="loading" />
      </template>
    </van-cell>
  </van-sticky>

    <Codemirror v-model:value="code" :options="cmOptions" border ref="cmRef" height="100vh" width="100%"
      @change="onChange" @input="onInput" @ready="onReady">
    </Codemirror>

  </van-config-provider>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue';
import { exec } from 'kernelsu';
import "codemirror/mode/shell/shell"
import Codemirror from "codemirror-editor-vue3";
import { Buffer } from 'buffer'

const theme = ref(true);
const checked = ref(false);
const disabled = ref(false);
const loading = ref(false);
const code = ref(
  ``
)
const logFile = '/data/adb/crond/run.log'
const rootCronFile = '/data/adb/crond/root'
const KEEP_ON_UNINSTALL = '/data/adb/crond/KEEP_ON_UNINSTALL'

let intervalId = null;

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
// execCmd('settings get secure ui_night_mode').then(v => {
//   // 0 表示跟随系统设置，即当前模式与系统设置的主题模式相匹配。
//   // 1 表示开启了 Dark Mode（夜间模式）。
//   // 2 表示关闭了 Dark Mode（白天模式）。
//   if (v == '1') {
//     theme.value = true;
//   } else if (v == '2') {
//     theme.value = false;
//   }
// });


const cmRef = ref()

const cmOptions = {
  mode: "text/x-sh"
}
const onChange = (val, cm) => {
  console.log(val)
  console.log(cm.getValue())
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
const reload = (filePath) => {
  console.log(filePath)
  if (typeof filePath == 'undefined') {
    filePath = rootCronFile;
    cmOptions.mode = "text/x-sh"
    cmOptions.readOnly = false;
    if (intervalId) {
      console.log('清除计时器')
      clearInterval(intervalId);
    }
    disabled.value = false;
  } else {
    disabled.value = true;
    cmOptions.mode = "log"
    cmOptions.readOnly = true;
  }
  cmRef.value?.cminstance.refresh()
  code.value = ""
  // //读取定时任务信息
  readFile(`${filePath}`).then(value => {
    code.value = value;
  }).catch(() => {
    showToast(`文件读取失败:${filePath}`);
  })

}
//更新配置
const update = () => {
  // /(((\d+,)+\d+|(\d+(\/|-)\d+)|\d+|\*) ?){5}/.test('*/5 4 * * 1-6')
  showToast(`尝试保存文件:${rootCronFile}`);
  //保存定时任务信息
  saveFile(code.value, `${rootCronFile}`).then(value => {
    showToast(`保存成功，重新加载配置文件`);
    reload()
  })
}
//读取配置文件
const loadLog = () => {
  reload(logFile)
  showToast(`每3s重新加载一次日志`);
  if (intervalId) {
    console.log('清除计时器')
    clearInterval(intervalId);
  }
  intervalId = setInterval(() => {
    reload(logFile)
  }, 3000);
}
const clearLog = () => {
  saveFile('', `${logFile}`).then(v=>{
    showToast(`清理日志完成`);
  })
}
//开关切换
watch(checked, (value, preValue) => {
  loading.value = true;
  let cmd = `[ -f ${KEEP_ON_UNINSTALL} ] && rm -f ${KEEP_ON_UNINSTALL}`
  if (typeof value != 'undefined' && value) {
    cmd = `touch ${KEEP_ON_UNINSTALL}`
  }
  setTimeout(() => {
    loading.value = false;
  }, 1000);
  execCmd(cmd).then(v => {
    console.log(v)
  });
})
reload()
execCmd(`if [ -f ${KEEP_ON_UNINSTALL} ];then echo 0;else echo 1;fi`).then(v => {
  if (v == '0') {
    checked.value = true;
  } else {
    checked.value = false;
  }
})

</script>

<style>
.van-theme-dark body {
  color: #f5f5f5;
  background-color: black;
}
</style>
