<template>
  <div ref="tableWrapperInstance" class="plus-table">
    <PlusTableTitleBar
      v-if="titleBar"
      :columns="filterColumns"
      :default-size="size"
      :columns-is-change="columnsIsChange"
      :title-bar="titleBar"
      @click-density="handleClickDensity"
      @filter-table="handleFilterTableConfirm"
      @refresh="handleRefresh"
    >
      <template #title>
        <slot name="title" />
      </template>

      <template #toolbar>
        <slot name="toolbar" />
      </template>

      <!-- 表格拖拽行 和 列设置里拖拽 icon -->
      <template v-if="$slots['drag-sort-icon']" #drag-sort-icon>
        <slot name="drag-sort-icon" />
      </template>

      <!-- 表格表头 列设置 icon   -->
      <template v-if="$slots['column-settings-icon']" #column-settings-icon>
        <slot name="column-settings-icon" />
      </template>

      <!-- 表表格表头 密度 icon  -->
      <template v-if="$slots['density-icon']" #density-icon>
        <slot name="density-icon" />
      </template>
    </PlusTableTitleBar>

    <el-table
      ref="tableInstance"
      v-loading="loadingStatus"
      :reserve-selection="true"
      :data="__tableData"
      :border="true"
      :height="height"
      :header-cell-style="headerCellStyle"
      :size="size"
      :row-key="rowKey"
      highlight-current-row
      scrollbar-always-on
      v-bind="$attrs"
      @cell-click="handleClickCell"
      @cell-dblclick="handleDoubleClickCell"
    >
      <!-- 默认插槽 -->
      <template #default>
        <slot name="default">
          <!-- 选择栏 -->
          <el-table-column
            v-if="isSelection"
            key="selection"
            type="selection"
            v-bind="selectionTableColumnProps"
          />

          <!-- 序号栏 -->
          <PlusTableTableColumnIndex
            v-if="hasIndexColumn"
            :index-content-style="indexContentStyle"
            :index-table-column-props="indexTableColumnProps"
            :page-info="(pagination as PlusPaginationProps)?.modelValue"
          />

          <!-- 拖拽行 -->
          <PlusTableColumnDragSort
            v-if="dragSortable"
            :sortable="dragSortable"
            :drag-sortable-table-column-props="dragSortableTableColumnProps"
            :table-instance="tableInstance"
            @dragSortEnd="handleDragSortEnd"
          >
            <template v-if="$slots['drag-sort-icon']" #drag-sort-icon>
              <slot name="drag-sort-icon" />
            </template>
          </PlusTableColumnDragSort>

          <!-- 展开行 -->
          <el-table-column v-if="hasExpand" type="expand" v-bind="expandTableColumnProps">
            <template #default="scoped">
              <div class="plus-table-expand-col">
                <slot name="expand" :index="scoped.$index" v-bind="scoped" />
              </div>
            </template>
          </el-table-column>

          <!--配置渲染栏  -->
          <PlusTableColumn
            :columns="subColumns"
            :editable="editable"
            @formChange="handleFormChange"
          >
            <!--表格单元格表头的插槽 -->
            <template v-for="(_, key) in headerSlots" :key="key" #[key]="data">
              <slot :name="key" v-bind="data" />
            </template>

            <!--表格单元格的插槽 -->
            <template v-for="(_, key) in cellSlots" :key="key" #[key]="data">
              <slot :name="key" v-bind="data" />
            </template>

            <!--表单单项的插槽 -->
            <template v-for="(_, key) in fieldSlots" :key="key" #[key]="data">
              <slot :name="key" v-bind="data" />
            </template>

            <!-- 表单el-form-item 下一行额外的内容 的插槽 -->
            <template v-for="(_, key) in extraSlots" :key="key" #[key]="data">
              <slot :name="key" v-bind="data" />
            </template>

            <!-- tooltip-icon  插槽 -->
            <template v-if="$slots['tooltip-icon']" #tooltip-icon>
              <slot name="tooltip-icon" />
            </template>

            <!--表格单元格编辑的插槽 -->
            <template v-if="$slots['edit-icon']" #edit-icon>
              <slot name="edit-icon" />
            </template>
          </PlusTableColumn>

          <!-- 操作栏 -->
          <PlusTableActionBar
            v-if="actionBar"
            v-bind="actionBar"
            @clickAction="handleAction"
            @clickActionConfirmCancel="handleClickActionConfirmCancel"
          >
            <!-- 操作栏更多icon插槽 -->
            <template v-if="$slots['action-bar-more-icon']" #action-bar-more-icon>
              <slot name="action-bar-more-icon" />
            </template>
          </PlusTableActionBar>
        </slot>
      </template>

      <!-- 插入至表格最后一行之后的内容 -->
      <template #append>
        <slot name="append" />
      </template>

      <!-- 当数据为空时自定义的内容 -->
      <template #empty>
        <slot name="empty" />
      </template>
    </el-table>

    <!-- 分页 -->
    <PlusPagination
      v-if="pagination"
      ref="paginationInstance"
      v-model="subPageInfo"
      v-bind="pagination"
      @change="handlePaginationChange"
    >
      <template v-if="$slots['pagination-left']" #pagination-left>
        <slot name="pagination-left" />
      </template>
      <template v-if="$slots['pagination-right']" #pagination-right>
        <slot name="pagination-right" />
      </template>
    </PlusPagination>
  </div>
</template>

<script lang="ts" setup>
import {
  nextTick,
  reactive,
  toRefs,
  watch,
  ref,
  provide,
  shallowRef,
  useSlots,
  unref,
  computed,
  onMounted,
  onBeforeUnmount
} from 'vue'
import type {
  PlusPaginationProps,
  PlusPaginationInstance
} from '@plus-pro-components/components/pagination'
import { PlusPagination } from '@plus-pro-components/components/pagination'
import {
  DefaultPageInfo,
  TableFormRefInjectionKey,
  TableFormFieldRefInjectionKey
} from '@plus-pro-components/constants'
import type { Ref, ComputedRef } from 'vue'
import type { ComponentSize } from 'element-plus/es/constants'
import type { TableInstance } from 'element-plus'
import { ElTable, ElTableColumn, vLoading } from 'element-plus'
import type {
  PageInfo,
  PlusColumn,
  RecordType,
  FormFieldRefsType
} from '@plus-pro-components/types'
import {
  getTableCellSlotName,
  getTableHeaderSlotName,
  getFieldSlotName,
  getExtraSlotName,
  filterSlots,
  isSVGElement,
  isPlainObject
} from '@plus-pro-components/components/utils'
import { cloneDeep, debounce } from 'lodash-es'
import PlusTableActionBar from './table-action-bar.vue'
import PlusTableColumn from './table-column.vue'
import PlusTableTableColumnIndex from './table-column-index.vue'
import PlusTableColumnDragSort from './table-column-drag-sort.vue'
import PlusTableTitleBar from './table-title-bar.vue'
import type {
  ButtonsCallBackParams,
  PlusTableState,
  PlusTableSelfProps as PlusTableProps,
  PlusTableEmits,
  TableFormRefRow,
  FormChangeCallBackParams
} from './type'

defineOptions({
  name: 'PlusTable',
  inheritAttrs: false
})

const props = withDefaults(defineProps<PlusTableProps>(), {
  defaultSize: 'default',
  pagination: false,
  actionBar: false,
  hasIndexColumn: false,
  titleBar: true,
  isSelection: false,
  hasExpand: false,
  loadingStatus: false,
  tableData: () => [],
  data: () => [],
  columns: () => [],
  headerCellStyle: () => ({
    'background-color': 'var(--el-fill-color-light)'
  }),
  rowKey: 'id',
  dragSortable: false,
  dragSortableTableColumnProps: () => ({}),
  indexTableColumnProps: () => ({}),
  indexContentStyle: () => ({}),
  selectionTableColumnProps: () => ({
    width: 40
  }),
  expandTableColumnProps: () => ({}),
  editable: false,
  adaptive: false
})
const emit = defineEmits<PlusTableEmits>()

const subColumns: Ref<PlusColumn[]> = ref([])
const columnsIsChange: Ref<boolean> = ref(false)
const filterColumns: Ref<PlusColumn[]> = ref([])
const tableInstance = shallowRef<TableInstance | null>(null)
const tableWrapperInstance = ref<HTMLDivElement | null>(null)
const paginationInstance = ref<PlusPaginationInstance | null>(null)
const state = reactive<PlusTableState>({
  subPageInfo: {
    ...(((props.pagination as PlusPaginationProps)?.modelValue || DefaultPageInfo) as PageInfo)
  },
  size: props.defaultSize
})
const __tableData: ComputedRef<RecordType[]> = computed(() =>
  props.tableData?.length ? props.tableData : props.data
)

const hasAdaptive = computed(() => typeof props.height === 'undefined' && props.adaptive)

const slots = useSlots()

/**
 * 表格单元格的插槽
 */
const cellSlots = filterSlots(slots, getTableCellSlotName())

/**
 * 表格单元格表头的插槽
 */
const headerSlots = filterSlots(slots, getTableHeaderSlotName())

/**
 * 表单单项的插槽
 */
const fieldSlots = filterSlots(slots, getFieldSlotName())

/**
 * el-form-item 下一行额外的内容 的插槽
 */
const extraSlots = filterSlots(slots, getExtraSlotName())

/**
 * 表单的ref
 */
const formRefs = shallowRef<Record<string | number, TableFormRefRow[]>>({})
provide(TableFormRefInjectionKey, formRefs)
/**
 * 表单Field的ref
 */
const formFieldRefs = shallowRef<FormFieldRefsType>({})
provide(TableFormFieldRefInjectionKey, formFieldRefs)

// 监听配置更改
watch(
  () => props.columns,
  val => {
    subColumns.value = val.filter(item => unref(item.hideInTable) !== true)
    filterColumns.value = cloneDeep(subColumns.value)
    columnsIsChange.value = !columnsIsChange.value
  },
  {
    deep: true,
    immediate: true
  }
)

// 发分页改变事件
const handlePaginationChange = () => {
  emit('paginationChange', { ...state.subPageInfo })
}

const handleAction = (callbackParams: ButtonsCallBackParams) => {
  emit('clickAction', callbackParams)
}

const handleClickActionConfirmCancel = (callbackParams: ButtonsCallBackParams) => {
  emit('clickActionConfirmCancel', callbackParams)
}

const handleFilterTableConfirm = (_columns: PlusColumn[]) => {
  filterColumns.value = _columns
  subColumns.value = _columns.filter(
    item => unref(item.hideInTable) !== true && item.__selfHideInTable !== true
  )
}

// 密度
const handleClickDensity = (size: ComponentSize) => {
  state.size = size
}

const handleDragSortEnd = (newIndex: number, oldIndex: number) => {
  emit('dragSortEnd', newIndex, oldIndex)
}

// 刷新
const handleRefresh = () => {
  emit('refresh')
}

const handleFormChange = (data: FormChangeCallBackParams) => {
  emit('formChange', data)
}

// 保存活动的表单
const currentForm = ref()

const handleCellEdit = (row: RecordType, column: PlusColumn, type: 'click' | 'dblclick') => {
  const rowIndex = __tableData.value.indexOf(row)
  const columnIndex = column.index
  const columnConfig = subColumns.value[column.index]

  // 不是可编辑行，如操作栏
  if (!columnConfig) return

  if (props.editable === type) {
    document.addEventListener('click', handleStopEditClick)

    const currentCellForm = formRefs.value[rowIndex][columnIndex]

    // 停止上一个表单的编辑状态
    if (currentForm.value) {
      currentForm.value?.stopCellEdit()
    }
    currentForm.value = currentCellForm

    // 开启当前点击的单元格的编辑
    currentCellForm.startCellEdit()

    // 当表单初始化完成
    const unwatch = watch(
      () => formFieldRefs.value.valueIsReady,
      val => {
        if (
          val?.value &&
          formFieldRefs.value?.fieldInstance?.focus &&
          (props.editable === 'click' || props.editable === 'dblclick')
        ) {
          formFieldRefs.value.fieldInstance.focus()
          // 销毁监听
          unwatch()
        }
      }
    )
  }
}

const handleClickCell = (
  row: RecordType,
  column: PlusColumn,
  cell: HTMLTableCellElement,
  event: Event
) => {
  handleCellEdit(row, column, 'click')
  emit('cell-click', row, column, cell, event)
}

const handleDoubleClickCell = (
  row: RecordType,
  column: PlusColumn,
  cell: HTMLTableCellElement,
  event: Event
) => {
  handleCellEdit(row, column, 'dblclick')
  emit('cell-dblclick', row, column, cell, event)
}

// 退出编辑状态
const handleStopEditClick = (e: MouseEvent) => {
  if (tableWrapperInstance.value && currentForm.value) {
    const wrapperClass = '.el-table__body-wrapper'
    // eslint-disable-next-line @typescript-eslint/no-non-null-assertion
    const tbody = tableWrapperInstance.value.querySelector(wrapperClass)!
    const target = e?.target as HTMLElement
    const cls = Array.from(target.classList).join('.')
    const tempCls = cls ? `.${cls}` : ''
    const contains = tempCls && tbody.querySelector(tempCls)
    if (!contains && !isSVGElement(target)) {
      currentForm.value?.stopCellEdit()
      emit('edited')
      document.removeEventListener('click', handleStopEditClick)
    }
  }
}

const setAdaptive = async () => {
  await nextTick()
  if (!tableInstance.value) return
  const tableWrapper = tableInstance.value.$el
  let offsetBottom = 20
  let paginationHeight = 0

  if (isPlainObject(props.adaptive)) {
    offsetBottom = props.adaptive.offsetBottom ?? offsetBottom
  }
  if (paginationInstance.value && props.pagination) {
    paginationHeight = paginationInstance.value.$el.offsetHeight
  }

  tableWrapper.style.height = `${
    window.innerHeight - tableWrapper.getBoundingClientRect().top - offsetBottom - paginationHeight
  }px`
}

const debounceSetAdaptive = debounce(
  setAdaptive,
  isPlainObject(props.adaptive) ? props.adaptive.timeout ?? 60 : 60
)

onMounted(() => {
  if (hasAdaptive.value) {
    setAdaptive()
    window.addEventListener('resize', debounceSetAdaptive)
  }
})

onBeforeUnmount(() => {
  if (hasAdaptive.value) {
    window.removeEventListener('resize', debounceSetAdaptive)
  }
})

const { subPageInfo, size } = toRefs(state)

defineExpose({
  formRefs,
  tableInstance
})
</script>
