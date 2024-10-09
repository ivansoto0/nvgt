# fetch_and
Atomically replaces the current value with the result of bitwise ANDing the value of this atomic object and `arg`. The operation is a read-modify-write operation. Memory is affected according to the value of `order`.

```nvgt
T fetch_and(T arg, memory_order order = MEMORY_ORDER_SEQ_CST);
```

## Parameters:
* `T arg`: the right-hand side of the bitwise AND operation.
* `memory_order order`: which memory order SHALL govern this operation.

## Returns:
The prior value of this atomic object.

## Remarks:
This function is only available on integral atomic types.

Within the above function signature, `T` is used as a placeholder for the actual type. For example, if this object is an `atomic_int`, then `T` SHALL be `int`.