# CompletionStage

1. 产出型: 用上一个阶段的结果作为指定 (Bi)Function 函数的参数执行指定函数产生新的结果
2. 消费型: 消费上一个阶段的结果作为指定 (Bi)Consumer 函数的参数, 但不对Stage的结果产生影响
3. 不消费也不产出型(just run): 只要上个阶段完成, 就执行指定函数 (Runnable类型)

