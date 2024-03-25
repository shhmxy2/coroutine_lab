# 一、姓名，学号和班级
毛心怡，2022012730，计26

# 二、三个小节，分别记录你为了实现三个子任务所添加到代码，并对代码进行详细的注释；每个子任务都有实验报告的额外要求，请阅读上面的文本
## 子任务一
### 分析代码：
代码1：
```
    // TODO: Task 1
    // 在实验报告中分析以下代码
    // 对齐到 16 字节边界
    uint64_t rsp = (uint64_t)&stack[stack_size - 1];
    rsp = rsp - (rsp & 0xF);
```
分析1：把栈顶的地址以无符号64位整数的形式存储在寄存器rsp中；随后，将rsp的后四位清零，使其对齐到16字节边界
代码2：
```
// TODO: Task 1
// 在实验报告中分析以下代码
void coroutine_main(struct basic_context *context) {
  context->run();
  context->finished = true;
  coroutine_switch(context->callee_registers, context->caller_registers);

  // unreachable
  assert(false);
}
```
分析2：先执行context；然后设置finished为true，标志着进程已跑完；最后调用携程切换函数，将控制权从caller转移给callee。assert函数在运行不出错时不可能到达，一旦到达，断言失败，程序终止。

### 添加的代码：
1. context.S：
```
    # TODO: Task 1
    # 保存 callee-saved 寄存器到 %rdi 指向的上下文
    movq %rsp, 64(%rdi) # 保存栈指针 受计22陈罗曦提醒
    movq %rbp, 72(%rdi)
    movq %rbx, 80(%rdi)
    movq %r12, 88(%rdi)
    movq %r13, 96(%rdi)
    movq %r14, 104(%rdi)
    movq %r15, 112(%rdi)

    # 保存的上下文中 rip 指向 ret 指令的地址（.coroutine_ret）
    leaq .coroutine_ret(%rip), %rax
    movq %rax, 120(%rdi)

    # 从 %rsi 指向的上下文恢复 callee-saved 寄存器
    movq 64(%rsi), %rsp # 恢复栈指针，同样受计22陈罗曦提醒
    movq 72(%rsi), %rbp
    movq 80(%rsi), %rbx
    movq 88(%rsi), %r12
    movq 96(%rsi), %r13
    movq 104(%rsi), %r14
    movq 112(%rsi), %r15
    
    # 最后 jmpq 到上下文保存的 rip
    movq 120(%rsi), %rax
    jmpq *%rax
```
2. context.h：
```
  /**
   * @brief 恢复协程函数运行。
   * TODO: Task 1
   * 你需要保存 callee-saved 寄存器，并且设置协程函数栈帧，然后将 rip 恢复到协程
   * yield 之后所需要执行的指令地址。
   */
  virtual void resume() {
    // 调用 coroutine_switch
    coroutine_switch(this->caller_registers, this->callee_registers); //从caller切换到callee
    // 在汇编中保存 callee-saved 寄存器，设置协程函数栈帧，然后将 rip 恢复到协程 yield 之后所需要执行的指令地址。
  }
```
3. common.h：
```
/**
 * @brief yield函数
 *
 * TODO: Task 1
 * 协程主动暂停执行，保存协程的寄存器和栈帧。
 * 将上下文转换至 coroutine_pool.serial_execute_all() 中的上下文进行重新的
 * schedule 调用。
 */
void yield() {
  if (!g_pool->is_parallel) {
    // 从 g_pool 中获取当前协程状态
    auto context = g_pool->coroutines[g_pool->context_id];
    // 调用 coroutine_switch 切换到 coroutine_pool 上下文
    coroutine_switch(context->callee_registers, context->caller_registers); //从callee切换到caller
  }
}
```
4. coroutine_pool.h：
```
  /**
   * @brief 以协程执行的方式串行并同时执行所有协程函数
   * TODO: Task 1, Task 2
   * 在 Task 1 中，我们不需要考虑协程的 ready
   * 属性，即可以采用轮询的方式挑选一个未完成执行的协程函数进行继续执行的操作。
   * 在 Task 2 中，我们需要考虑 sleep 带来的 ready
   * 属性，需要对协程函数进行过滤，选择 ready 的协程函数进行执行。
   *
   * 当所有协程函数都执行完毕后，退出该函数。
   */
  void serial_execute_all() {
    is_parallel = false;
    g_pool = this;
    this->context_id = 0;
    int to_be_done = coroutines.size();
    int done = 0;
    for(int& i=g_pool->context_id; i<to_be_done; i == coroutines.size() - 1 ? i = 0 : i++) { //本行代码受计26 陈畅指教 int&使得i可更改，```i == coroutines.size() - 1 ? i = 0 : i++```使得i能反复从0开始循环
      if(coroutines[i]->finished){
        continue;
      }
      coroutines[i]->resume();
      if(coroutines[i]->finished){
        done++;
      }
      if(done==to_be_done){ // 假如所有进程都已结束，则break
        break;
      }
    }
    
    for (auto context : coroutines) {
      delete context;
    }
    coroutines.clear();
  }
```
### 1. 绘制出在协程切换时，栈的变化过程；
|      Coroutine      |
|:-------------------:|
|      Stack Top      |
|        ...          |
|    Caller's Stack   |
|          ↓          |
|    **Coroutine**    |
|      Stack Top      |
|        ...          |
|        %r15         |
|        %r14         |
|        %r13         |
|        %r12         |
|        %rbx         |
|        %rbp         |
|        %rsp         |
|        ...          |
|    Caller's Stack   |
|          ↓          |
|    **Coroutine**    |
|      Stack Top      |
|        ...          |
|    Return Address   |
|        %r15         |
|        %r14         |
|        %r13         |
|        %r12         |
|        %rbx         |
|        %rbp         |
|        %rsp         |
|        ...          |
|    Caller's Stack   |
|          ↓          |
|    **Coroutine**    |
|      Stack Top      |
|        ...          |
|    Caller's Stack   |

### 2. 并结合源代码，解释协程是如何开始执行的，包括 `coroutine_entry` 和 `coroutine_main` 函数以及初始的协程状态；
basic_context中，用r12寄存器寄存coroutine_main的地址，r13寄存器寄存coroutine_main的参数；  
coroutine_entry中，将coroutine_main的参数（r13寄存器中）赋给rdi寄存器，并调用r12寄存器中寄存的地址，进入coroutine_main；  
coroutine_main中，先执行context，然后设置finished为true，然后调用coroutine_switch把控制权从callee移交给caller；  
coroutine_switch中，先保存callee-saved的寄存器和栈指针，进阶炸保存指向coroutine_ret的地址，再恢复callee-saved的寄存器和栈指针，最后跳转到coroutine_ret，返回。

### 3. 目前的协程切换代码只考虑了通用寄存器，设计一下，如果要考虑浮点和向量寄存器，要怎么处理。
浮点和向量寄存器也需要在coroutine_switch中保存和恢复这些寄存器的状态，操作方式与通用寄存器相似，指令则尹寄存器类型而异地选择相应的mov指令。

## 子任务二
### 添加的代码：
1. common.h：
```
/**
 * @brief 完成 sleep 函数
 *
 * TODO: Task 2
 * 你需要完成 sleep 函数。
 * 此函数的作用为：
 *  1. 将协程置为不可用状态。
 *  2. yield 协程。
 *  3. 在至少 @param ms 毫秒之后将协程置为可用状态。
 */
void sleep(uint64_t ms) {
  if (g_pool->is_parallel) {
    auto cur = get_time();
    while (
        std::chrono::duration_cast<std::chrono::milliseconds>(get_time() - cur)
            .count() < ms)
      ;
  } else {
    // 从 g_pool 中获取当前协程状态
    auto context = g_pool->coroutines[g_pool->context_id];
    // 获取当前时间，更新 ready_func
    auto cur = get_time();
    context->ready = false; // 预先设置ready为false，即还不能运行
    // ready_func：检查当前时间，如果已经超时，则返回 true
    context->ready_func = [ms, cur]() {
      return std::chrono::duration_cast<std::chrono::milliseconds>(get_time() - cur).count() >= ms;
    };
    // 调用 coroutine_switch 切换到 coroutine_pool 上下文
    coroutine_switch(context->callee_registers, context->caller_registers);
  }
}

```
2. coroutine_pool.h：
```
      if(!coroutines[i]->ready){ // 如果已经准备好运行，直接向下执行，否则用redy_func更新ready的状态
        coroutines[i]->ready = coroutines[i]->ready_func();
        if(!coroutines[i]->ready){ // 如果仍未准备好运行，直接跳过
          continue;
        }
      }
```
### 1. 按照时间线，绘制出 `sleep_sort` 中不同协程的运行情况；
以所给范例为例：  
输入：  
```
5
1 3 4 5 2
```
时间线：  
start-s  
end-e  
|s1|s3|s4|s5|s2|e1|e2|e3|e4|e5
|-|-|-|-|-|-|-|-|-|-|

### 2. 目前的协程库实现方式是轮询 `ready_func` 是否等于 `true`，设计一下，能否有更加高效的方法。
可以用一个向量存储未ready的context，每次对向量中的context进行询问；  
在存储时，估计context准备完成所需的时间长短，并根据时间长短进行插入排序。

## 子任务三
### 添加的代码：
1. binary_search.cpp：
```
    // TODO: Task 3
    // 使用 __builtin_prefetch 预取容易产生缓存缺失的内存
    __builtin_prefetch(&table[probe], 1, 3);
    // 并调用 yield
    yield();
```
### 1. 汇报性能的提升效果。
对```__builtin_prefetch(&table[probe], 1, 3);```的第二、三个参数进行排列组合的测试后，发现选取1,3的效果最好，但在binary_search所给定的条件下，效果明显差于naive
1. 完全不改变任何参数：
```
Size: 4294967296
Loops: 1000000
Batch size: 16
Initialization done
naive: 2891.33 ns per search, 90.35 ns per access
coroutine batched: 1345.95 ns per search, 42.06 ns per access
``` 
但在后续测试中发现，这是一个极端的例子，不调整参数，大部分情况下仍然是naive性能好于coroutine batched。  
对于一系列测试数据，有如下表格：  
|index|l|m|b|naive|coroutine_batched|c_no_prefetch|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1|16|1000000|16|30.12, 1.88|694.26, 43.39|156.33, 9.77|
|2|30|1000000|16|829.04, 29.73|1265.10, 42.17|1506.15, 50.21|
|3|31|1000000|16|1008.69, 32.54|1297.95, 41.87|1601.68, 51.67|
|4|32|100000|16|1104.17, 34.51|1333.47, 41.67|1689.21, 52.79|
|5|32|10000000|16|1089.12, 34.04|1363.10, 42.60|1708.77, 53.40|
|6|32|1000000|8|1086.71, 33.96|1308.77, 40.90|1741.25, 54.41|
|7|32|1000000|32|1123.43, 35.11|2486.34, 77.70|2809.36, 87.79|

遗憾的是，相对于naive， coroutine_batched性能并没有显著的提升。不过在多组数据的对比中可以看出：  
当数据量较小时，coroutine_batched速度并不快，而随着数据量的增加，其速度减慢的程度相比naive，非常可观，可以推测在更大的数据量下，会体现出更好的性能；  
当查询数目变化时，性能的差异体现不大；  
当batch_size变化时，在一定范围内，随着batch_size的提升，coroutine_batched耗时增加并不多，但超出这个范围，耗时就会显著增加。  

另一方面，将coroutine_batched与没有prefetch的coroutine比较，可以看见前者的性能总是优于后者（虽然在1号数据点，即数据量极小的时候有一个反常的后者更优，推测是因为数据量较小，所以prefetch反而更消耗时间）。  
注：对于表格中的数据，为5次测试后去除最大值和最小值去平均值得出

# 三、记录你在完成本实验的过程中，与哪些同学进行了交流，查阅了哪些网站或者代码
与计22陈罗曦和计26陈畅有所交流，均已标注在代码与report中。

# 四、总结和感想
老师提供的作业要求非常细致，只要脚踏实地地照做就可以了。作业过程中基本没有遇到问题，大部分完成过程都很流畅。  
本次作业让我更深刻地体会和理解了汇编代码的写作与应用，同时复习锻炼了oop的能力。另一方面，在Task3中，也锻炼了数据分析的能力（虽然分析结果和预期不尽相同）。