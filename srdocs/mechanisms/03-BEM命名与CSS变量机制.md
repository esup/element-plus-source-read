# 03 - BEM 命名与 CSS 变量机制

## 概述

Element Plus 通过 `useNamespace` hook 实现了完整的 BEM 命名系统，同时支持 CSS 自定义属性（CSS Variables）的动态生成，是所有组件样式 class 的统一来源。

## 核心源码

**文件**: `packages/hooks/use-namespace/index.ts`

### BEM 核心函数

```typescript
const _bem = (
  namespace: string,
  block: string,
  blockSuffix: string,
  element: string,
  modifier: string
) => {
  let cls = `${namespace}-${block}`       // el-button
  if (blockSuffix) cls += `-${blockSuffix}` // el-button-group
  if (element) cls += `__${element}`        // el-button__text
  if (modifier) cls += `--${modifier}`      // el-button--primary
  return cls
}
```

### useNamespace 返回值

```typescript
export const useNamespace = (block: string, namespaceOverrides?) => {
  const namespace = useGetDerivedNamespace(namespaceOverrides)

  const b  = (blockSuffix = '') => _bem(namespace.value, block, blockSuffix, '', '')
  const e  = (element?: string) => element ? _bem(namespace.value, block, '', element, '') : ''
  const m  = (modifier?: string) => modifier ? _bem(namespace.value, block, '', '', modifier) : ''
  const be = (blockSuffix?, element?) => ...
  const em = (element?, modifier?) => ...
  const bm = (blockSuffix?, modifier?) => ...
  const bem = (blockSuffix?, element?, modifier?) => ...

  // 状态类名
  const is = (name: string, ...args) => {
    const state = args.length >= 1 ? args[0]! : true
    return name && state ? `is-${name}` : ''
  }

  // CSS 变量
  const cssVar = (object) => {
    // 生成 --el-xxx: value
  }
  const cssVarBlock = (object) => {
    // 生成 --el-button-xxx: value
  }
  const cssVarName = (name) => `--${namespace.value}-${name}`
  const cssVarBlockName = (name) => `--${namespace.value}-${block}-${name}`

  return { namespace, b, e, m, be, em, bm, bem, is, cssVar, cssVarName, cssVarBlock, cssVarBlockName }
}
```

## 命名空间继承

```typescript
export const useGetDerivedNamespace = (namespaceOverrides?) => {
  const derivedNamespace = namespaceOverrides ||
    (getCurrentInstance() ? inject(namespaceContextKey, ref(defaultNamespace)) : ref(defaultNamespace))
  return computed(() => unref(derivedNamespace) || defaultNamespace)  // 默认 'el'
}
```

命名空间可通过 `ConfigProvider` 的 `namespace` 属性全局配置，所有组件自动继承。

## 使用示例

```typescript
// 在组件中使用
const ns = useNamespace('button')

const buttonKls = computed(() => [
  ns.b(),                      // 'el-button'
  ns.m(_type.value),           // 'el-button--primary'
  ns.m(_size.value),           // 'el-button--large'
  ns.is('disabled', disabled), // 'is-disabled'
  ns.is('loading', loading),   // 'is-loading'
  ns.is('round', round),       // 'is-round'
  ns.is('circle', circle),     // 'is-circle'
])

// CSS 变量
const style = computed(() => ns.cssVar({ 'color-primary': '#409eff' }))
// => { '--el-color-primary': '#409eff' }
```

## BEM 命名规范

| 方法 | 输出格式 | 示例 |
|------|----------|------|
| `ns.b()` | `{ns}-{block}` | `el-button` |
| `ns.b('group')` | `{ns}-{block}-{suffix}` | `el-button-group` |
| `ns.e('text')` | `{ns}-{block}__{element}` | `el-button__text` |
| `ns.m('primary')` | `{ns}-{block}--{modifier}` | `el-button--primary` |
| `ns.em('text', 'expand')` | `{ns}-{block}__{element}--{modifier}` | `el-button__text--expand` |
| `ns.be('group', 'text')` | `{ns}-{block}-{suffix}__{element}` | `el-button-group__text` |
| `ns.bem('group', 'text', 'lg')` | `{ns}-{block}-{suffix}__{element}--{modifier}` | `el-button-group__text--lg` |
| `ns.is('loading')` | `is-{state}` | `is-loading` |

## CSS 变量系统

```scss
// 全局 CSS 变量（theme-chalk）
:root {
  --el-color-primary: #409eff;
  --el-color-success: #67c23a;
  --el-border-radius-base: 4px;
  --el-font-size-base: 14px;
}

// 组件级 CSS 变量
.el-button {
  --el-button-bg-color: var(--el-color-primary);
  --el-button-border-color: var(--el-color-primary);
}
```

## 设计要点

| 要点 | 说明 |
|------|------|
| 可配置命名空间 | 通过 ConfigProvider 可全局修改前缀（默认 `el`） |
| 类型安全 | 所有方法参数都有 TypeScript 类型约束 |
| 空值安全 | 参数为空时返回空字符串，不会产生无效 class |
| CSS 变量联动 | 支持全局变量和组件级变量的动态生成 |
| 状态前缀统一 | 所有状态类使用 `is-` 前缀，与 BEM 修饰符区分 |
