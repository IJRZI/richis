# lambda, std::function, etc
## 1. 问题
之前写代码遇到了一些传递回调函数的需求, 例如:
```c++
receive_packet_and_handle([](const char* buf, int len){
    // do sth. with buf
});
```

这样 `receive_packet_and_handle` 有两种写法:
1.  ```c++
    template<typename F>
    void receive_packet_and_handle(F&& handler) {
        // ...
    }
    ```
2.  ```c++
    using handler_t = std::function<void(const char*, int)>;
    void receive_packet_and_handle(const handler_t& handler) {
        // ...
    }
    ```

想知道两种传参有没有性能上的不同

## 2. 结论

先说结论, `template` 风格性能通常比 `std::function` 风格好, 前者能 inline lambda, 后者通常不能 inline, 可能还需要构造 `std::function` 对象

参考 [can-be-stdfunction-inlined-or-should-i-use-different-approach](https://stackoverflow.com/questions/42856707/can-be-stdfunction-inlined-or-should-i-use-different-approach) 的回答
> `std::function` is **not** a zero-runtime-cost abstraction. It is a type-erased wrapper that has a virtual-call like cost when invoking operator() and could also potentially heap-allocate (which could mean a cache-miss per call).
>
> The compiler will **most likely not be able to inline it**.
>
> If you want to store your function objects in such a way that does not introduce additional overhead and that allows the compiler to inline it, you should use a template parameters. This is not always possible, but might fit your use case.

不过之前想用 `std::function` 的另一个原因是它能约束传入的 lambda 的参数及返回值, 用模板的话当传入的函数签名不符合预期时会有比较晦涩的报错, 而且函数使用者也很难直接从函数声明看出来到底要传入什么 lambda, 返回值如何, 可读性很差

对于这个问题, 查到的解决办法是用 `std::invocable`, 参考 [how-can-i-restrict-lambda-signatures-in-c17-template-arguments](https://stackoverflow.com/questions/64029445/how-can-i-restrict-lambda-signatures-in-c17-template-arguments)


## 3. 汇编
在 Compiler Explorer 上用 x86-64 gcc 12.2 测试如下代码

```c++
template<typename F>
void with_tempalte(F&& f) {
    f(10);
}

void with_function(std::function<void(int x)>&& f) {
    f(10);
}

static volatile int a;
void test() {
    // #1
    with_tempalte([](int x){
        a = x;
    });

    // #2
    with_function([](int x){
        a = x;
    });
}
```

### O0
#### 1. 调用 `with_template`
```
lea     rax, [rbp-65]
mov     rdi, rax
call    void with_tempalte<test()::{lambda(int)#1}>(test()::{lambda(int)#1}&&)
```
#### 2. 调用 `with_function`
```
lea     rdx, [rbp-17]
lea     rax, [rbp-64]
mov     rsi, rdx
mov     rdi, rax
call    std::function<void (int)>::function<test()::{lambda(int)#2}, void>(test()::{lambda(int)#2}&&)
lea     rax, [rbp-64]
mov     rdi, rax
call    with_function(std::function<void (int)>&&)
lea     rax, [rbp-64]
mov     rdi, rax
call    std::function<void (int)>::~function() [complete object destructor]
jmp     .L19
mov     rbx, rax
lea     rax, [rbp-64]
mov     rdi, rax
call    std::function<void (int)>::~function() [complete object destructor]
mov     rax, rbx
mov     rdi, rax
call    _Unwind_Resume
```

用 `std::function` 时会构造 `std::function` 对象, 后者显然更低效

### O1
#### 1.
lambda 被直接内联了, 汇编只有
```
mov     DWORD PTR a[rip], 10
```
#### 2.
```
mov     rdi, rsp
call    with_function(std::function<void (int)>&&)

with_function(std::function<void (int)>&&):
        sub     rsp, 24
        mov     DWORD PTR [rsp+12], 10
        cmp     QWORD PTR [rdi+16], 0
        je      .L9
        lea     rsi, [rsp+12]
        call    [QWORD PTR [rdi+24]]
        add     rsp, 24
        ret
.L9:
        call    std::__throw_bad_function_call()
```
仍然存在对 `std::function` 对象的调用, 没有内联

### O3
```
        mov     DWORD PTR a[rip], eax
        ret
with_function(std::function<void (int)>&&):
        sub     rsp, 24
        cmp     QWORD PTR [rdi+16], 0
        mov     DWORD PTR [rsp+12], 10
        je      .L10
        lea     rsi, [rsp+12]
        call    [QWORD PTR [rdi+24]]
        add     rsp, 24
        ret
.L10:
        call    std::__throw_bad_function_call()
test():
        mov     DWORD PTR a[rip], 10 # with_template
        mov     DWORD PTR a[rip], 10 # with_function
        ret
```
O3 情况下二者都被内联了, 但此时 `with_function` 仍然出现在了汇编结果里