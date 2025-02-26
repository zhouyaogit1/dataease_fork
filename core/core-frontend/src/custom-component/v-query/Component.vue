<script lang="ts" setup>
import eventBus from '@/utils/eventBus'
import { ElMessage } from 'element-plus-secondary'
import { snapshotStoreWithOut } from '@/store/modules/data-visualization/snapshot'
import QueryConditionConfiguration from './QueryConditionConfiguration.vue'
import type { ComponentInfo } from '@/api/chart'
import { infoFormat } from './options'
import {
  onBeforeUnmount,
  reactive,
  ref,
  toRefs,
  watch,
  computed,
  onMounted,
  CSSProperties
} from 'vue'
import { storeToRefs } from 'pinia'
import { useI18n } from '@/hooks/web/useI18n'
import { dvMainStoreWithOut } from '@/store/modules/data-visualization/dvMain'
import { comInfo } from './com-info'
import { useEmitt } from '@/hooks/web/useEmitt'
import StyleInject from './StyleInject.vue'
const props = defineProps({
  view: {
    type: Object,
    default() {
      return {
        customStyle: ''
      }
    }
  },
  element: {
    type: Object,
    default() {
      return {
        id: null,
        propValue: []
      }
    }
  },
  showPosition: {
    type: String,
    required: true,
    default: ''
  }
})
const { element, view } = toRefs(props)
const { t } = useI18n()
const dvMainStore = dvMainStoreWithOut()
const { curComponent, canvasViewInfo } = storeToRefs(dvMainStore)
const canEdit = ref(false)
const queryConfig = ref()
const defaultStyle = {
  border: '',
  background: '',
  text: '',
  layout: 'horizontal',
  btnList: ['sure'],
  titleLayout: 'left',
  titleShow: false,
  titleColor: '',
  textColorShow: false,
  labelColor: '',
  bgColorShow: false,
  borderShow: false,
  labelColorShow: false,
  title: ''
}
const customStyle = reactive({ ...defaultStyle })
const snapshotStore = snapshotStoreWithOut()

const curComponentView = computed(() => {
  return (canvasViewInfo.value[element.value.id] || {}).customStyle
})

const { datasetFieldList } = comInfo(element.value.id)

const setCustomStyle = val => {
  const {
    show,
    borderShow,
    borderColor,
    bgColorShow,
    btnList,
    titleLayout,
    labelColor,
    labelColorShow,
    text,
    bgColor,
    layout,
    titleShow,
    titleColor,
    textColorShow,
    title
  } = val
  if (!show) {
    Object.assign(customStyle, { ...defaultStyle })
    return
  }
  customStyle.background = bgColorShow ? bgColor || '' : ''
  customStyle.border = borderShow ? borderColor || '' : ''
  customStyle.btnList = [...btnList]
  customStyle.layout = layout
  customStyle.titleShow = titleShow
  customStyle.titleColor = titleColor
  customStyle.labelColor = labelColorShow ? labelColor || '' : ''
  customStyle.title = title
  customStyle.text = textColorShow ? text || '' : ''
  customStyle.titleLayout = titleLayout
}

watch(
  () => view.value,
  val => {
    if (!val?.customStyle?.component) return
    setCustomStyle(val?.customStyle?.component)
  },
  {
    deep: true,
    immediate: true
  }
)

watch(
  () => curComponentView.value,
  val => {
    if (!val?.component) return
    setCustomStyle(val?.component)
  },
  {
    deep: true,
    immediate: true
  }
)
const list = ref([])

watch(
  () => props.element.propValue,
  () => {
    list.value = [...props.element.propValue]
  },
  {
    immediate: true
  }
)
const onComponentClick = () => {
  if (curComponent.value.id !== element.value.id) {
    canEdit.value = false
  }
}

const { emitter } = useEmitt()

onBeforeUnmount(() => {
  emitter.off(`addQueryCriteria${element.value.id}`)
  emitter.off(`editQueryCriteria${element.value.id}`)
  emitter.off(`updateQueryCriteria${element.value.id}`)
  eventBus.off('componentClick', onComponentClick)
})

const updateQueryCriteria = () => {
  Array.isArray(element.value.propValue) &&
    element.value.propValue.forEach(ele => {
      if (ele.auto) {
        const componentInfo = {
          datasetId: ele.dataset.id,
          id: ele.field.id
        }
        const checkedFields = []
        const checkedFieldsMap = {}
        datasetFieldList.value.forEach(ele => {
          if (ele.tableId === componentInfo.datasetId) {
            checkedFields.push(ele.id)
            checkedFieldsMap[ele.id] = componentInfo.id
          }
        })
        ele.checkedFields = checkedFields
        ele.checkedFieldsMap = checkedFieldsMap
      }
    })
}

onMounted(() => {
  emitter.on(`addQueryCriteria${element.value.id}`, addCriteriaConfigOut)
  emitter.on(`editQueryCriteria${element.value.id}`, editQueryCriteria)
  emitter.on(`updateQueryCriteria${element.value.id}`, updateQueryCriteria)
  updateQueryCriteria()
})

const dragover = () => {
  // do
}

const drop = e => {
  const componentInfo: ComponentInfo = JSON.parse(e.dataTransfer.getData('dimension') || '{}')
  if (!componentInfo.id) return
  const checkedFields = []
  const checkedFieldsMap = {}
  datasetFieldList.value.forEach(ele => {
    if (ele.tableId === componentInfo.datasetId) {
      checkedFields.push(ele.id)
      checkedFieldsMap[ele.id] = componentInfo.id
    }
  })
  list.value.push({
    ...infoFormat(componentInfo),
    auto: true,
    optionValueSource: 1,
    checkedFields,
    checkedFieldsMap,
    displayType: `${componentInfo.deType}`
  })
  element.value.propValue = [...list.value]
  snapshotStore.recordSnapshotCache()
}

const editeQueryConfig = (queryId: string) => {
  queryConfig.value.setCondition(queryId)
}

const editQueryCriteria = () => {
  if (!list.value.length) {
    addCriteriaConfigOut()
    return
  }
  editeQueryConfig(list.value[0].id)
}

const addCriteriaConfigOut = () => {
  queryConfig.value.setConditionOut()
}

const delQueryConfig = index => {
  list.value.splice(index, 1)
  element.value.propValue = [...list.value]
  snapshotStore.recordSnapshotCache()
}

const resetData = () => {
  ;(list.value || []).reduce((pre, next) => {
    if (!next.defaultValueCheck) {
      next.defaultValue = next.multiple || +next.displayType === 7 ? [] : undefined
    }
    next.selectValue = Array.isArray(next.defaultValue) ? [...next.defaultValue] : next.defaultValue
    const keyList = Object.entries(next.checkedFieldsMap)
      .filter(ele => next.checkedFields.includes(ele[0]))
      .filter(ele => !!ele[1])
      .map(ele => ele[0])
    pre = [...new Set([...keyList, ...pre])]
    return pre
  }, [])
}

const clearData = () => {
  ;(list.value || []).reduce((pre, next) => {
    next.selectValue = next.multiple || +next.displayType === 7 ? [] : undefined
    const keyList = Object.entries(next.checkedFieldsMap)
      .filter(ele => next.checkedFields.includes(ele[0]))
      .filter(ele => !!ele[1])
      .map(ele => ele[0])
    pre = [...new Set([...keyList, ...pre])]
    return pre
  }, [])
}
const listVisible = computed(() => {
  return list.value.filter(itx => itx.visible)
})

const queryData = () => {
  let requiredName = ''
  const emitterList = (element.value.propValue || []).reduce((pre, next) => {
    if (next.required) {
      if (!next.defaultValueCheck) {
        requiredName = next.name
      }

      if (
        (Array.isArray(next.selectValue) && !next.selectValue.length) ||
        (next.selectValue !== 0 && !next.selectValue)
      ) {
        requiredName = next.name
      }
    }
    const keyList = Object.entries(next.checkedFieldsMap)
      .filter(ele => next.checkedFields.includes(ele[0]))
      .filter(ele => !!ele[1])
      .map(ele => ele[0])
    pre = [...new Set([...keyList, ...pre])]
    return pre
  }, [])
  if (!!requiredName) {
    ElMessage.error(`【${requiredName}】查询条件是必填项，请设置选项值后，再进行查询！`)
    return
  }
  if (!emitterList.length) return

  emitterList.forEach(ele => {
    emitter.emit(`query-data-${ele}`)
  })
}
const titleStyle = computed(() => {
  return {
    textAlign: customStyle.titleLayout || 'left',
    color: customStyle.titleColor || '#1f2329'
  } as CSSProperties
})

const labelStyle = computed(() => {
  return {
    color: customStyle.labelColor || '#1f2329'
  } as CSSProperties
})
const opacityStyle = computed(() => {
  return element.value?.style?.opacity
    ? ({
        opacity: element.value.style.opacity
      } as CSSProperties)
    : {}
})
</script>

<template>
  <div class="v-query-container" :style="opacityStyle">
    <p v-if="customStyle.titleShow" class="title" :style="titleStyle">
      {{ customStyle.title }}
    </p>
    <div
      :class="[
        'v-query',
        customStyle.layout,
        customStyle.titleShow && !!customStyle.title && 'title-show'
      ]"
      @dragover.prevent.stop="dragover"
      @drop.prevent.stop="drop"
    >
      <div v-if="!listVisible.length" class="no-list-label flex-align-center">
        <div class="container flex-align-center">
          将右侧的字段拖拽到这里 或 点击
          <el-button :disabled="showPosition === 'preview'" @click="addCriteriaConfigOut" text>
            添加查询条件
          </el-button>
        </div>
      </div>
      <div class="query-fields-container">
        <div class="query-item" :key="ele.id" v-for="(ele, index) in listVisible">
          <div class="query-field">
            <div class="label">
              <div class="label-wrapper">
                <div class="label-wrapper-text" :style="labelStyle">
                  <el-tooltip effect="dark" :content="ele.name" placement="top">
                    {{ ele.name }}
                  </el-tooltip>
                  <span v-if="ele.required" class="required">*</span>
                </div>
              </div>
              <div class="label-wrapper-tooltip" v-if="showPosition !== 'preview'">
                <el-tooltip effect="dark" content="设置过滤条件" placement="top">
                  <el-icon @click="editeQueryConfig(ele.id)">
                    <Icon name="icon_edit_outlined"></Icon>
                  </el-icon>
                </el-tooltip>
                <el-tooltip effect="dark" content="删除条件" placement="top">
                  <el-icon style="margin-left: 8px" @click="delQueryConfig(index)">
                    <Icon name="icon_delete-trash_outlined"></Icon>
                  </el-icon>
                </el-tooltip>
              </div>
            </div>
            <div class="query-select">
              <StyleInject :customStyle="customStyle" :config="ele"></StyleInject>
            </div>
          </div>
        </div>
        <div class="query-button" v-if="!!listVisible.length">
          <el-button @click.stop="clearData" v-if="customStyle.btnList.includes('clear')" secondary>
            {{ t('commons.clear') }}
          </el-button>
          <el-button @click.stop="resetData" v-if="customStyle.btnList.includes('reset')" secondary>
            {{ t('chart.reset') }}
          </el-button>
          <el-button
            @click.stop="queryData"
            style="margin-right: 7px"
            v-if="customStyle.btnList.includes('sure')"
            type="primary"
          >
            {{ t('commons.adv_search.search') }}
          </el-button>
        </div>
      </div>
    </div>
  </div>
  <Teleport to="body">
    <QueryConditionConfiguration
      :query-element="element"
      ref="queryConfig"
    ></QueryConditionConfiguration>
  </Teleport>
</template>

<style lang="less" scoped>
.v-query-container {
  width: 100%;
  height: 100%;
  overflow: auto;
  position: relative;

  .no-list-label {
    width: 100%;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 0;
    .container {
      width: 100%;
      justify-content: center;
      color: #646a73;
      text-align: center;
      font-family: AlibabaPuHuiTi;
      font-size: 16px;
      font-style: normal;
      font-weight: 400;
      line-height: 24px;
      .ed-button {
        font-size: 16px;
        font-style: normal;
        font-weight: 400;
        line-height: 24px;
      }
    }
  }
  .title {
    color: #1f2329;
    font-feature-settings: 'clig' off, 'liga' off;
    font-family: AlibabaPuHuiTi;
    font-size: 14px;
    font-style: normal;
    font-weight: 500;
    line-height: 22px;
    letter-spacing: -0.1px;
  }
}
.v-query {
  width: 100%;
  height: 100%;
  line-height: 1.5;
  color: rgba(0, 0, 0, 0.87);
  align-items: center;
  position: relative;
  display: flex;
  margin: auto 0;
  padding: 16px 4px 4px 4px;
  .query-fields-container {
    display: flex;
    flex-wrap: wrap;
    width: 100%;
    height: 100%;
  }
  .query-item {
    position: relative;
    line-height: 28px;
    margin: 5px 16px 5px 0;
    max-width: 100%;
    min-width: 60px;
    .query-field {
      position: relative;
      .label {
        display: flex;
        overflow: hidden;
        color: #1f2329;

        .label-wrapper {
          visibility: visible;
          pointer-events: auto;
          cursor: auto;
          line-height: 16px;
          color: #575757;
          box-sizing: border-box;
          margin: 0;
          padding: 0;
          display: flex;
          flex: 1 1 0;
          overflow: hidden;
        }
        .label-wrapper-text {
          position: relative;
          cursor: pointer;
          flex: 0 1 auto;
          max-width: 100%;
          overflow: hidden;
          text-overflow: ellipsis;
          white-space: nowrap;
          color: #1f2329;
          font-family: AlibabaPuHuiTi;
          font-size: 14px;
          font-style: normal;
          font-weight: 400;
          line-height: 22px;

          .required {
            font-size: 14px;
            color: #f54a45;
            margin-left: 3px;
            line-height: 22px;
          }
        }
        .label-wrapper-tooltip {
          align-items: center;
          background: #fff;
          border-radius: 2px;
          font-size: 16px;
          display: none;
          flex: 0 0 auto;
          height: 16px;
          line-height: 16px;
          color: #575757;
        }
      }

      .query-select {
        display: flex;
        flex-wrap: wrap;
        line-height: 28px;

        :deep(.ed-date-editor) {
          width: 227px;
          .ed-input__wrapper {
            width: 100%;
          }
        }

        :deep(.ed-select-v2) {
          min-width: 170px;
        }
      }
    }
  }
  .query-button {
    align-self: flex-end;
    line-height: 28px;
    margin: auto 0 5px auto;
    z-index: 0;
  }

  &.title-show {
    height: calc(100% - 22px);
  }

  &.vertical {
    .query-fields-container {
      .query-field {
        padding-top: 30px;

        &:hover {
          .label-wrapper-tooltip {
            display: inline-flex !important;
            cursor: pointer;
          }
        }
        .label {
          align-items: center;
          height: 16px;
          left: 0;
          line-height: 16px;
          max-width: 100%;
          position: absolute;
          right: 0;
          top: 0;
        }
      }
    }
  }

  &.horizontal {
    line-height: 1.5;
    color: rgba(0, 0, 0, 0.87);
    align-items: center;
    display: flex;
    margin: auto 0;
    .query-fields-container {
      align-items: flex-start;

      .query-field {
        align-items: center;
        display: flex;
        .label {
          visibility: visible;
          pointer-events: auto;
          cursor: auto;
          line-height: 28px;
          box-sizing: border-box;
          flex: 0 0 auto;
          margin-right: 8px;

          .label-wrapper {
            max-width: 200px;
          }
          .label-wrapper-tooltip {
            position: absolute;
            right: 0;
            top: -26px;
            z-index: 11;
            padding: 4px 8px;
            height: 26px;
            width: 58px;
            border-radius: 4px;
            border: 1px solid #dee0e3;
            background: #fff;
            box-shadow: 0px 4px 8px 0px rgba(31, 35, 41, 0.1);
          }
        }
        &:hover {
          .label-wrapper-tooltip {
            display: block;
            cursor: pointer;
          }
        }
      }
    }
  }
}
</style>
<style lang="less">
.load-select {
  .ed-select-dropdown__list {
    & > div {
      &:nth-child(1) {
        .ed-radio__inner::after {
          display: none !important;
        }
      }
    }
  }
  .ed-select-dropdown {
    &:nth-child(1) {
      .ed-checkbox__inner::after {
        display: none !important;
      }
    }
  }
}
</style>
