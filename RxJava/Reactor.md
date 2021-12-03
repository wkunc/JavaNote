# Reactor Core 

## Flux, an Asynchronous Sequence of O~N Items
一个异步的多个元素的序列.

Flux<T> 是标准的 Publisher<T> (代表0~N个元素的异步序列)
可选的终结通过 completion signal 或者 error.

与 Reactive Stream 规范, 这三种信号转换为对 downstream 订阅者的
onNext, onComplete, onError 方法的调用.


