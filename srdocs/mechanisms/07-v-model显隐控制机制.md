# 07 - v-model 显隐控制机制

## 概述

Element Plus 通过 `useModelToggle` hook 实现了统一的 v-model 显隐控制模式，被 Dialog、Drawer、Tooltip、Popover、Dropdown 等所有需要显隐切换的组件使用。

## 核心源码

**文件**: `packages/hooks/use-model-toggle/index.ts`

```typescript
export const useModelToggle = <T extends string>(name: T) => {
  const updateEventKeyRaw = `onUpdate:${name}`
  const updateEventKey = `update:${name}`

  return ({
    indicator,        // Ref<boolean> — 显隐状态
    toggleReason,     // Ref<Event|undefined> — 触发原因
    shouldHideWhenRouteChanges,  // 路由变化时是否隐藏
    shouldProceed,    // 是否允许操作
    onShow,           // 显示回调
    onHide,           // 隐藏回调
  }: ModelToggleParams) => {
    const instance = getCurrentInstance()!
    const { emit } = instance
    const props = instance.props

    // 判断是否有外部 v-model 绑定
    const hasUpdateHandler = computed(() =>
      isFunction(props[updateEventKeyRaw])
    )
    const isModelBindingAbsent = computed(() => props[name] === null)

    // 内部显示
    const doShow = (event?) => {
      if (indicator.value === true) return
      indicator.value = true
      if (toggleReason) toggleReason.value = event
      onShow?.(event)
    }

    // 内部隐藏
    const doHide = (event?) => {
      if (indicator.value === false) return
      indicator.value = false
      if (toggleReason) toggleReason.value = event
      onHide?.(event)
    }

    // 显示（考虑外部绑定）
    const show = (event?) => {
      if (props.disabled === true) return
      if (shouldProceed && !shouldProceed()) return

      const shouldEmit = hasUpdateHandler.value && isClient
      if (shouldEmit) {
        emit(updateEventKey, true)  // 通知外部更新
      }
      if (isModelBindingAbsent.value || !shouldEmit) {
        doShow(event)  // 无外部绑定时直接操作
      }
    }

    // 隐藏（考虑外部绑定）
    const hide = (event?) => {
      if (props.disabled === true) return
      const shouldEmit = hasUpdateHandler.value && isClient
      if (shouldEmit) emit(updateEventKey, false)
      if (isModelBindingAbsent.value || !shouldEmit) doHide(event)
    }

    // 监听外部 prop 变化
    const onChange = (val: boolean) => {
      if (!isBoolean(val)) return
      if (props.disabled && val) {
        if (hasUpdateHandler.value) emit(updateEventKey, false)
      } else if (indicator.value !== val) {
        val ? doShow() : doHide()
      }
    }

    const toggle = () => indicator.value ? hide() : show()

    watch(() => props[name], onChange)

    // 路由变化时自动隐藏
    if (shouldHideWhenRouteChanges && instance.appContext.config.globalProperties.$route) {
      watch(() => ({ ...instance.proxy.$route }), () => {
        if (shouldHideWhenRouteChanges.value && indicator.value) hide()
      })
    }

    onMounted(() => { onChange(props[name]) })

    return { hide, show, toggle, hasUpdateHandler }
  }
}
```

## 双模式工作

### 受控模式（有 v-model）

```vue
<el-dialog v-model="visible">
  <!-- visible 由外部管理 -->
  <!-- show() → emit('update:modelValue', true) -->
  <!-- 外部更新 visible → watch 触发 onChange → doShow() -->
</el-dialog>
```

### 非受控模式（无 v-model）

```vue
<el-dialog>
  <!-- 直接操作内部 indicator -->
  <!-- show() → doShow() → indicator = true -->
</el-dialog>
```

## 使用场景

| 组件 | v-model 属性 | 说明 |
|------|-------------|------|
| Dialog | `v-model` | 对话框显隐 |
| Drawer | `v-model` | 抽屉显隐 |
| Tooltip | `v-model:visible` | 提示显隐 |
| Popover | `v-model:visible` | 气泡卡片显隐 |
| Dropdown | `v-model:visible` | 下拉菜单显隐 |
| Popconfirm | `v-model:visible` | 气泡确认框显隐 |

## 设计要点

| 要点 | 说明 |
|------|------|
| 双模式兼容 | 同时支持受控和非受控使用 |
| 防重复操作 | `doShow`/`doHide` 内部检查状态是否已匹配 |
| 禁用态保护 | disabled 为 true 时 show/hide 无效 |
| 路由感知 | 可选路由变化时自动隐藏弹层 |
| 触发原因记录 | `toggleReason` 记录触发事件，便于调试 |
