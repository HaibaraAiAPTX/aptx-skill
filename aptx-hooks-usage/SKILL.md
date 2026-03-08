---
name: aptx-hooks-usage
description: "使用代码生成的 React Query/Vue Query hooks 时使用。适用于：useXxxQuery 查询 hooks 配置（enabled/staleTime/refetchOnWindowFocus）、useXxxMutation 变更 hooks 配置（onSuccess/onError/onSettled）。关键：配置必须放在 query 或 mutation 命名空间内。当代码出现 useXxxQuery、useXxxMutation 模式的 hooks 且需要配置选项时应触发。"
---

# Aptx Hooks Usage

## 核心原则

aptx 生成的 hooks 使用**命名空间**封装配置对象，LLM 必须正确使用：

- **查询 (Query)**: 配置必须放在 `query` 属性内
- **变更 (Mutation)**: 配置必须放在 `mutation` 属性内

## React Hooks 用法

### useXxxQuery (查询)

```typescript
// ✅ 正确写法
const query = useAssignmentRequirementGetListQuery(input, {
  query: {
    enabled: true,
    staleTime: 5000,
    refetchOnWindowFocus: false,
  }
})

// ❌ 错误写法 - LLM 常见错误
const query = useAssignmentRequirementGetListQuery(input, {
  enabled: true,  // 直接写在外面会丢失！
})
```

### useXxxMutation (变更)

```typescript
// ✅ 正确写法
const mutation = useAssignmentRequirementAddMutation({
  mutation: {
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['assignment'] })
    },
    onError: (error) => {
      console.error(error)
    }
  }
})

// ❌ 错误写法
const mutation = useAssignmentRequirementAddMutation({
  onSuccess: () => {},  // 丢失！
})
```

### 常用配置选项

**Query 配置** (`options.query`):
- `enabled`: boolean - 是否自动执行查询
- `staleTime`: number - 数据新鲜时间（毫秒）
- `refetchOnWindowFocus`: boolean - 窗口聚焦时重新获取
- `retry`: number | boolean - 重试次数
- `initialData`: TOutput - 初始数据

**Mutation 配置** (`options.mutation`):
- `onSuccess`: (data) => void - 成功回调
- `onError`: (error) => void - 错误回调
- `onSettled`: () => void - 完成回调（无论成功失败）
- `retry`: number | boolean - 突变重试

## Vue Hooks 用法

### useXxxQuery (查询)

```typescript
// ✅ 正确写法
const query = useAssignmentRequirementGetListQuery(input, {
  query: {
    enabled: true,
    staleTime: 5000,
  }
})
```

### useXxxMutation (变更)

```typescript
// ✅ 正确写法
const mutation = useAssignmentRequirementAddMutation({
  mutation: {
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['assignment'] })
    }
  }
})
```

## 常见错误清单

| 错误类型 | 错误写法 | 正确写法 |
|---------|---------|---------|
| 直接传配置 | `{ enabled: false }` | `{ query: { enabled: false } }` |
| 缺少命名空间 | `{ staleTime: 3000 }` | `{ query: { staleTime: 3000 } }` |
| Mutation 配置 | `{ onSuccess: fn }` | `{ mutation: { onSuccess: fn } }` |

## 参考资料

- React 完整用法: See [references/react.md](references/react.md)
- Vue 完整用法: See [references/vue.md](references/vue.md)
