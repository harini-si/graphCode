<template>
  <div class="editorBox" :class="{ hide: hide }">
    <Drag v-if="show" :number="editorItemList.length" :dir="dir">
      <Chat> </Chat>

      <DragItem
        v-for="(item, index) in editorItemList"
        :key="item.title"
        :index="index"
        :disabled="item.disableDrag"
        :showTouchBar="item.showTouchBar"
        :title="item.showTitle ? item.title : ''"
      >
        <EditorItem
          :key="component_key"
          :title="item.title"
          :language="item.language"
          :codeTheme="codeTheme"
          :codeFontSize="codeFontSize"
          :content="item.content"
          :preprocessorList="preprocessorListMap[item.title]"
          :showAddBtn="item.showAddBtn"
          :dir="dir"
          :showAllAddResourcesBtn="['vue2', 'vue3'].includes(item.language)"
          :showHeader="showHeader"
          :readOnly="readOnly"
          @code-change="
            code => {
              codeChange(item, code)
            }
          "
          @preprocessor-change="
            p => {
              preprocessorChange(item, p)
            }
          "
          @add-resource="
            languageType => {
              addResource(languageType || item.title)
            }
          "
          @add-importmap="addImportmap(item)"
          @space-change="
            noSpace => {
              item.showTitle = noSpace
            }
          "
          @create-code-img="
            editor => {
              showCreateCodeImg(editor, item)
            }
          "
        ></EditorItem>
      </DragItem>
    </Drag>

    <EditAssets ref="EditAssetsComp"></EditAssets>

    <CodeToImg
      ref="CodeToImgComp"
      :getThemeData="getThemeData"
      :codeTheme="codeTheme"
      :codeFontSize="codeFontSize"
    ></CodeToImg>

    <EditImportMap
      :codeTheme="codeTheme"
      :codeFontSize="codeFontSize"
    ></EditImportMap>
  </div>
</template>

<script setup>
let component_key = 0

import {
  ref,
  defineProps,
  onMounted,
  computed,
  getCurrentInstance,
  watch,
  onUnmounted
} from 'vue'
import { useStore } from 'vuex'
import EditorItem from '@/components/EditorItem.vue'
import Drag from './Drag.vue'
import DragItem from './DragItem.vue'
import Chat from './Chat.vue'
import { defaultEditorMap, preprocessorListMap } from '@/config/constants'
import { ElMessage } from 'element-plus'
import { codeThemeList } from '@/config/codeThemeList'
import { base } from '@/config'
import * as monaco from 'monaco-editor/esm/vs/editor/editor.api'
import CodeToImg from '@/components/CodeToImg.vue'
import EditImportMap from './EditImportMap.vue'
import EditAssets from './EditAssets.vue'

// props
const props = defineProps({
  hide: {
    type: Boolean,
    default: false
  },

  dir: {
    type: String,
    default: 'h'
  },

  showList: {
    type: Array,
    default() {
      return ['HTML', 'CSS', 'JS']
    }
  },
  showHeader: {
    type: Boolean,
    default: true
  },
  notRunCode: {
    type: Boolean,
    default: false
  },
  readOnly: {
    type: Boolean,
    default: false
  },
  showChat: {
    type: Boolean,
    default: true
  }
})

const useInit = () => {
  const store = useStore()
  return {
    store,
    proxy: getCurrentInstance().proxy,
    editData: computed(() => store.state.editData), // 数据
    codeTheme: computed(() => store.state.editData.config.codeTheme), // 代码主题
    codeFontSize: computed(() => store.state.editData.config.codeFontSize) // 代码字号
  }
}

const useInitEditorList = ({ props, editData }) => {
  let show = ref(false)

  let editorItemList = ref([])

  const initEditorItemList = () => {
    editorItemList = ref(
      props.showList.map(item => {
        if (typeof item === 'string') {
          return {
            ...defaultEditorMap[item],
            showTitle: false
          }
        } else {
          return {
            ...defaultEditorMap[item.title],
            ...item,
            showTitle: false
          }
        }
      })
    )
  }
  initEditorItemList()

  watch(
    () => {
      return props.showList
    },
    initEditorItemList,
    {
      deep: true
    }
  )

  const setInitData = () => {
    const code = editData.value.code
    Object.keys(code).forEach(type => {
      let index = getIndexByType(type)
      if (index === -1) {
        return
      }
      editorItemList.value[index].content = code[type].content
      editorItemList.value[index].language = code[type].language
    })
  }

  return {
    show,
    editorItemList,
    setInitData
  }
}

// 处理主题
const useTheme = ({ codeTheme, proxy }) => {
  let themeData = null
  // 加载主题
  const loadTheme = async () => {
    try {
      if (!codeTheme.value) {
        return
      }
      let item = codeThemeList.find(item => {
        return item.value === codeTheme.value
      })
      if (!item) {
        return
      }
      if (item.custom) {
        if (item.loaded) {
          themeData = item.cache
        } else {
          themeData = await (
            await fetch(`${base}themes/${codeTheme.value}.json`)
          ).json()
          item.loaded = true
          item.cache = themeData
        }
        monaco.editor.defineTheme(codeTheme.value, themeData)
      }
      monaco.editor.setTheme(codeTheme.value)
      proxy.$eventEmitter.emit('set-theme', themeData)
    } catch (error) {
      console.log(error)
      ElMessage.error('主题加载失败，请重试')
    }
  }

  const getThemeData = () => {
    return themeData
  }

  watch(codeTheme, () => {
    loadTheme()
  })

  return {
    loadTheme,
    getThemeData
  }
}

const useRunCode = ({ store, proxy }) => {
  const layout = computed(() => {
    return store.state.editData.config.layout
  })

  const runCode = () => {
    if (props.notRunCode) return
    proxy.$eventEmitter.emit('run')
    if (layout.value === 'newWindowPreview') {
      proxy.$eventEmitter.emit('preview_window_run')
    }
  }

  proxy.$eventEmitter.on('run-code', runCode)

  const openAlmightyConsole = computed(() => {
    return store.state.editData.config.openAlmightyConsole
  })
  watch(
    () => {
      return openAlmightyConsole.value
    },
    () => {
      runCode()
    }
  )

  return {
    runCode
  }
}

const useEditorChange = ({
  setInitData,
  store,
  editorItemList,
  autoRun,
  runCode,
  editData,
  proxy
}) => {
  const resetCode = () => {
    setInitData()
    runCode()
  }
  proxy.$eventEmitter.on('reset_code', resetCode)
  onUnmounted(() => {
    proxy.$eventEmitter.off('reset_code', resetCode)
  })

  const codeChange = (item, code) => {
    store.commit('setCodeContent', {
      type: item.title,
      code
    })
    autoRun()
  }

  const getIndexByType = type => {
    return editorItemList.value.findIndex(item => {
      return item.title === type
    })
  }

  const preprocessorChange = (item, p) => {
    let index = getIndexByType(item.title)
    editorItemList.value[index].language = p
    editorItemList.value[index].content =
      editData.value.code[item.title].content
    store.commit('setCodePreprocessor', {
      type: item.title,
      preprocessor: p
    })
    runCode()
  }

  return {
    codeChange,
    getIndexByType,
    preprocessorChange
  }
}

const useAutoRun = ({ store, runCode }) => {
  let autoRunTimer = null
  const isAutoRun = computed(() => {
    return store.state.editData.config.autoRun
  })
  const autoRun = () => {
    if (!isAutoRun.value) {
      return
    }
    clearTimeout(autoRunTimer)
    autoRunTimer = setTimeout(() => {
      runCode()
    }, 1000)
  }
  return {
    autoRun
  }
}

const CodeToImgComp = ref(null)
const useCodeToImg = () => {
  const showCreateCodeImg = (_editor, item) => {
    CodeToImgComp.value.showCreateCodeImg(_editor, item)
  }

  return {
    showCreateCodeImg
  }
}

const EditAssetsComp = ref(null)
const useAssets = () => {
  const addResource = (...args) => {
    EditAssetsComp.value.addResource(...args)
  }
  const addImportmap = (...args) => {
    EditAssetsComp.value.addImportmap(...args)
  }

  return {
    addResource,
    addImportmap
  }
}

// created部分
const { store, editData, codeTheme, proxy, codeFontSize } = useInit()
const { show, editorItemList, setInitData } = useInitEditorList({
  props,
  editData
})
const { loadTheme, getThemeData } = useTheme({ codeTheme, proxy })
const { runCode } = useRunCode({ store, proxy })
const { autoRun } = useAutoRun({ store, runCode })
const { getIndexByType, preprocessorChange, codeChange } = useEditorChange({
  setInitData,
  store,
  editorItemList,
  autoRun,
  runCode,
  editData,
  proxy
})
const { showCreateCodeImg } = useCodeToImg()
const { addResource, addImportmap } = useAssets()
onMounted(async () => {
  await loadTheme()
  setInitData()
  show.value = true
  runCode()
})

// Openai api stuff

// import dotenv from 'dotenv';
// dotenv.config();
</script>

<style lang="less" scoped>
.editorBox {
  width: 100%;
  height: 100%;
  display: flex;

  &.hide {
    display: none;
  }
}

/deep/ .el-dialog__body {
  padding: 20px;
}
</style>
