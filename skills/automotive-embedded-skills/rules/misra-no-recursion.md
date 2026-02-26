---
title: No Recursion in Embedded Context
impact: CRITICAL
impactDescription: prevents stack overflow
tags: misra, recursion, stack-overflow, iterative, embedded, safety
---

## No Recursion in Embedded Context

Recursion makes stack usage unpredictable. In embedded systems with limited stack, this can cause hard faults. Replace all recursive algorithms with iterative equivalents using explicit stacks.

**Incorrect (recursive tree traversal):**

```c
uint32_t SumTree(const TreeNode_t *node)
{
    if (node == NULL) { return 0U; }
    return node->value + SumTree(node->left) + SumTree(node->right);
}
```

**Correct (iterative with explicit stack):**

```c
uint32_t SumTree(const TreeNode_t *root)
{
    uint32_t sum = 0U;
    const TreeNode_t *stack[MAX_TREE_DEPTH];
    int16_t top = -1;

    if (root != NULL)
    {
        stack[++top] = root;
    }

    while (top >= 0)
    {
        const TreeNode_t *node = stack[top--];
        sum += node->value;

        if ((node->right != NULL) && (top < (MAX_TREE_DEPTH - 1)))
        {
            stack[++top] = node->right;
        }
        if ((node->left != NULL) && (top < (MAX_TREE_DEPTH - 1)))
        {
            stack[++top] = node->left;
        }
    }
    return sum;
}
```

Reference: MISRA C:2012 Rule 17.2 — Functions shall not call themselves, either directly or indirectly.
