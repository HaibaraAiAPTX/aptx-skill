# Vue Hooks 完整用法

## 创建 Hooks

```typescript
import { createVueQueryHooks } from '@aptx/api-query-vue';

// 生成 hooks
const useAssignmentHooks = createVueQueryHooks(assignmentQueryDef);
```

## useAptxQuery API

```typescript
function useAptxQuery<TInput, TOutput, TError = Error>(
  input: MaybeRefOrGetter<TInput>,
  options?: {
    query?: Omit<
      UseQueryOptions<TOutput, TError, TOutput, QueryKey>,
      "queryKey" | "queryFn"
    >;
  }
): UseQueryReturnType<TOutput, TError>
```

注意: `input` 支持 ref 或 getter，适合响应式场景。

## 完整示例

```typescript
import { ref } from 'vue';
import { useAssignmentRequirementGetListQuery } from '@/api/assignment';

// 基本用法
const input = ref({ assignmentId: 123 });
const query = useAssignmentRequirementGetListQuery(input);

// 使用 getter
const query = useAssignmentRequirementGetListQuery(
  () => ({ assignmentId: formData.value?.id })
);

// 带配置
const { data, isLoading, error, refetch } = useAssignmentRequirementGetListQuery(
  input,
  {
    query: {
      enabled: true,
      staleTime: 5000,
    }
  }
);

// 条件查询 - Vue 中可使用 getter 实现响应式
const { data } = useAssignmentRequirementGetListQuery(
  () => formData.value?.id ? { assignmentId: formData.value.id } : { assignmentId: -1 },
  {
    query: {
      enabled: () => !!formData.value?.id, // 响应式判断
    }
  }
);
```

## useAptxMutation API

```typescript
function useAptxMutation<TInput, TOutput, TError = Error>(
  options?: {
    mutation?: Omit<UseMutationOptions<TOutput, TError, TInput, unknown>, "mutationFn">;
  }
): UseMutationReturnType<TOutput, TError, TInput, unknown>
```

## Mutation 完整示例

```typescript
import { useAssignmentRequirementAddMutation } from '@/api/assignment';
import { useQueryClient } from '@tanstack/vue-query';

function AddButton() {
  const queryClient = useQueryClient();

  const mutation = useAssignmentRequirementAddMutation({
    mutation: {
      onSuccess: (data) => {
        queryClient.invalidateQueries({ queryKey: ['assignment-requirement'] });
      },
      onError: (error) => {
        console.error(error);
      },
    }
  });

  const handleSubmit = async () => {
    mutation.mutate({ name: 'New Item' });
  };

  return <button onClick={handleSubmit}>提交</button>;
}
```

## 状态属性

### Query Return
- `data`: Ref<TOutput | undefined>
- `error`: Ref<TError | null>
- `isLoading`: ComputedRef<boolean>
- `isFetching`: ComputedRef<boolean>
- `isError`: ComputedRef<boolean>
- `isSuccess`: ComputedRef<boolean>
- `failureCount`: ComputedRef<number>
- `refetch()`: () => Promise<void>

### Mutation Return
- `mutate(input)`: void
- `mutateAsync(input)`: Promise<TOutput>
- `data`: Ref<TOutput | undefined>
- `error`: Ref<TError | null>
- `isPending`: ComputedRef<boolean>
- `isSuccess`: ComputedRef<boolean>
- `isError`: ComputedRef<boolean>
- `reset()`: void
