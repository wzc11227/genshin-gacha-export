<template>
  <div class="flex justify-between">
    <div>
      <el-button type="primary" :icon="state.status === 'init' ? 'el-icon-sugar': 'el-icon-refresh-right'" class="focus:outline-none" :disabled="!allowClick()" plain size="small" @click="fetchData" :loading="state.status === 'loading'">{{state.status === 'init' ? '加载数据': '更新数据'}}</el-button>
      <el-button icon="el-icon-folder-opened" @click="saveExcel" class="focus:outline-none" :disabled="!gachaData" size="small" type="success" plain>导出Excel</el-button>
      <el-tooltip v-if="detail" content="从其它账号导出数据" placement="bottom">
        <el-button @click="newUser()" plain icon="el-icon-plus" size="small" class="focus:outline-none"></el-button>
      </el-tooltip>
    </div>
    <el-select v-if="state.dataMap && (state.dataMap.size > 1 || (state.dataMap.size === 1 && state.current === 0))" class="w-32" size="small" @change="changeCurrent" v-model="uidSelectText" placeholder="请选择">
      <el-option
        v-for="item of state.dataMap"
        :key="item[0]"
        :label="maskUid(item[0])"
        :value="item[0]">
      </el-option>
    </el-select>
  </div>
  <p class="text-gray-400 my-2 text-xs">{{hint}}</p>
  <div v-if="detail" class="gap-4 grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 2xl:grid-cols-4">
    <div class="mb-4" v-for="(item, i) of detail" :key="i">
      <p class="text-center text-gray-600 my-2">{{typeMap.get(item[0])}}</p>
      <pie-chart :data="item" :typeMap="typeMap"></pie-chart>
      <gacha-detail :data="item" :typeMap="typeMap"></gacha-detail>
    </div>
  </div>
</template>

<script setup>
const { ipcRenderer } = require('electron')
import { reactive, computed, watch, onMounted } from 'vue'
import PieChart from './components/PieChart.vue'
import GachaDetail from './components/GachaDetail.vue'
import gachaDetail from './gachaDetail'
import { version } from '../../package.json'

const state = reactive({
  status: 'init',
  log: '',
  data: null,
  dataMap: new Map(),
  current: 0
})

const gachaData = computed(() => {
  return state.dataMap.get(state.current)
})

const uidSelectText = computed(() => {
  if (state.current === 0) {
    return '新用户'
  } else {
    return state.current
  }
})

const allowClick = () => {
  const data = state.dataMap.get(state.current)
  if (!data) return true
  if (Date.now() - data.time < 1000 * 60) {
    return false
  }
  return true
}

const hint = computed(() => {
  const data = state.dataMap.get(state.current)
  if (state.status === 'init') {
    return '请先在游戏里打开任意一个抽卡记录后再点击“加载数据”按钮'
  } else if (state.status === 'loaded') {
    return `上次数据更新时间为：${new Date(data.time).toLocaleString()}`
  } else if (state.status === 'loading') {
    return state.log || 'Loading...'
  } else if (state.status === 'failed') {
    return state.log + ' - 操作失败'
  }
  return '　'
})

const detail = computed(() => {
  const data = state.dataMap.get(state.current)
  if (data) {
    return gachaDetail(data.result)
  }
})

const typeMap = computed(() => {
  const data = state.dataMap.get(state.current)
  return data.typeMap
})

const fetchData = async () => {
  state.status = 'loading'
  const data = await ipcRenderer.invoke('FETCH_DATA')
  if (data) {
    state.dataMap = data.dataMap
    state.current = data.current
    state.status = 'loaded'
  } else {
    state.status = 'failed'
  }
}

const readData = async () => {
  const data = await ipcRenderer.invoke('READ_DATA')
  if (data) {
    state.dataMap = data.dataMap
    state.current = data.current
    if (data.dataMap.get(data.current)) {
      state.status = 'loaded'
    }
  }
}

const saveExcel = async () => {
  await ipcRenderer.invoke('SAVE_EXCEL')
}

const changeCurrent = async (uid) => {
  state.current = uid
  await ipcRenderer.invoke('CHANGE_UID', uid)
}

const newUser = async () => {
  state.status = 'init'
  state.current = 0
  await changeCurrent(0)
}

const maskUid = (uid) => {
  return `${uid}`.replace(/(.+)(.{3})$/, '******$2')
}

onMounted(() => {
  readData()

  ipcRenderer.on('LOAD_DATA_STATUS', (event, message) => {
    state.log = message
  })

  ipcRenderer.on('ERROR', (event, err) => {
    console.error(err)
  })

  document.title = `原神抽卡记录导出工具 - v${version}`
})
</script>