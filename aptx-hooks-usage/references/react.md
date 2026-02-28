# React Hooks 完整用法

## 创建 Hooks

```typescript
import { createReactQueryHooks } from '@aptx/api-query-react';

// 生成 hooks
const useAssignmentHooks = createReactQueryHooks(assignmentQueryDef);
```

## useAptxQuery API

```typescript
function useAptxQuery<TInput, TOutput, TError = Error>(
  input: TInput,
  options?: {
    query?: Omit<
      UseQueryOptions<TOutput, TError, TOutput, QueryKey>,
      "queryKey" | "queryFn"
    >;
  }
): UseQueryResult<TOutput, TError>
```

## 完整示例

```typescript
import { useAssignmentRequirementGetListQuery } from '@/api/assignment';

// 基本用法
const { data, isLoading, error } = useAssignmentRequirementGetListQuery({
  assignmentId: 123
});

// 带配置
const { data, isLoading, error, refetch } = useAssignmentRequirementGetListQuery(
  { assignmentId: 123 },
  {
    query: {
      enabled: true,
      staleTime: 5 * 60 * 1000, // 5分钟
      refetchOnWindowFocus: false,
      retry: 2,
    }
  }
);

// 条件查询
const { data, isLoading } = useAssignmentRequirementGetListQuery(
  { assignmentId: formData.id },
  {
    query: {
      enabled: !!formData.id, // 只有 id 存在时才执行
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
): UseMutationResult<TOutput, TError, TInput, unknown>
```

## Mutation 完整示例

```typescript
import { useAssignmentRequirementAddMutation } from '@/api/assignment';
import { useQueryClient } from '@tanstack/react-query';

function AddButton() {
  const queryClient = useQueryClient();

  const mutation = useAssignmentRequirementAddMutation({
    mutation: {
      onSuccess: (data) => {
        // 刷新列表
        queryClient.invalidateQueries({ queryKey: ['assignment-requirement'] });
        console.log('创建成功:', data);
      },
      onError: (error) => {
        console.error('创建失败:', error);
      },
      onSettled: () => {
        // 无论成功失败都执行
      },
    }
  });

  const handleSubmit = async () => {
    mutation.mutate({ name: 'New Item' });
  };

  return (
    <button onClick={handleSubmit} disabled={mutation.isPending}>
      {mutation.isPending ? '提交中...' : '提交'}
    </button>
  );
}
```

## 状态属性

### Query Result
- `data`: TOutput | undefined
- `error`: TError | null
- `isLoading`: boolean
- `isFetching`: boolean
- `isError`: boolean
- `isSuccess`: boolean
- `refetch()`: Promise<void>
- `remove()`: void

### Mutation Result
- `mutate(input)`: void
- `mutateAsync(input)`: Promise<TOutput>
- `data`: TOutput | undefined
- `error`: TError | null
- `isPending`: boolean
- `isSuccess`: boolean
- `isError`: boolean
- `reset()`: void
