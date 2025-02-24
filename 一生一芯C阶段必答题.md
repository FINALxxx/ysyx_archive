# 一生一芯C阶段考核

## 1.程序是个状态机

- 一个死循环
  - 每次执行一条指令
    - 取指：利用当前pc，在内存数组M中取出一条指令，放入this.inst中
    - 译码与执行：在switch中对this.inst进行译码，并执行对应效果（当没有结果时发生op invalid，并设置halt=1）
    - 更新pc：由于没有j/b类型的语句，pc固定赋值为pc+1
  - 若halt=1则跳出循环，输出加法结果

## 2. RTFSC理解指令执行的过程

- 结构体cpu：包含当前pc、静态下一跳snpc等信息
- execute(执行次数)
  - exec_once：执行一条命令
  - device_update：对device进行更新
- exec_once(当前pc,指令解码对象s)
  - 更新s->pc为当前pc
  - 执行一次isa_exec_once
  - 将执行后更新的dnpc，赋值给s->dnpc
- isa_exec_once(s)
  - 取指令：执行一次inst_fetch,并将解析后的指令，赋值给s->isa.inst.val，此时s指令准备完毕
  - 译码与执行：执行一次decode_exec



- inst_fetch(s)
  - 定义在include/cpu/iftech.h，比较难找
  - 调用vaddr_ifetch,进而调用paddr_read(pc)，得到指令的uint32_t值
  - s->pc+=指令长度（riscv中为4）
  - 返回指令

- paddr_read(addr,len)

  - ==判断是否有外设==

    - 无外设时：

      1. 计算指令对应的pc是否在mem中，若过界，则直接使用out_bound进行assert

      2. 若未过界，则通过pmem_read来获取对应地址的指令（它会将地址转为指针，并返回指针指向位置的数据）

    - 有外设时：

      1. 计算指令对应的pc是否在mem中，若过界，则直接调用mmio_read(pc)（假设nemu使用mmio形式）
      2. 同上，调用pmem_read

- pmem_read(addr,len)

  - 计算访问地址，调用==host_read==读取内容并返回

- mmio_read(addr,len)

  - 使用fetch_mmio_map(addr)，查找addr对应的外设map
  - 调用map_read(addr,len,map)

- map_read(addr,len,map)

  - check_bound(addr)检查addr是否过界，若过界，则直接assert
    - 根据外设map找到其对应回调函数map->callback()，全部传进invoke_callback进行调用
    - 调用==host_read==读取内容并返回




- decode_exec(s)
  - 通过INSTPAT宏，对s->isa.inst.val进行按位匹配，匹配对应指令类型，并执行对应操作。



- ==对于地址访问的总结==

  - 无外设：

    - 过界：报错
    - 不过界：paddr_xxx->pmem_xxx->host_xxx

  - 有外设

    - 过界：paddr_xxx->mmio_xxx->map_xxx->host_xxx
    - 不过界：paddr_xxx->pmem_xxx->host_xxx

    

## 3. NEMU游戏是怎么运行的

- main
  - 执行ioe_init，初始化io扩展
  - 执行video_init，初始化画面
  - 使用io_read读取AM_TIMER_UPTIME，以毫秒为单位，并计算出1frame的时间
  - 当时间小于1frame时，执行game_logic_update，进行游戏更新
  - 一个死循环
    - 其中使用io_read读取AM_INPUT_KEYBRD，使用check_hit检查输入是否命中掉落字符
    - 当时间大于rendered时，执行render
- ioe_init()==与架构相关==
  - 有一个lut，可以查找对应的AM抽象的设备寄存器
  - 检查AM设备寄存器是否存在
  - 进行\_\_am_gpu_init()、\_\_am_gpu_init()、\_\_am_gpu_init()
- 以gpu为例，\_\_am_gpu_init()
  - 使用outl，对SYNC_ADDR地址，输入一个1
- render
  - 执行使用io_write向AM_GPU_FBDRAW写入画面绘制的数据



- io_read()
  - 是一个klib-macros.h中的宏定义
  - 输入一个寄存器名（reg），宏会自动拼接出对应的“设备存储器结构体”名（reg_T），并实例化（reg_T obj）
  - 调用ioe_read(reg，结构体对象obj)，将reg中的值读取到obj中，然后将obj作为宏的返回值
- io_write()
  - 是一个klib-macros.h中的宏定义
  - 输入一个寄存器名（reg），宏会自动拼接出对应的“设备存储器结构体”名（reg_T），实例化后赋值（reg_T obj={...}）
  - 调用ioe_write(reg，结构体对象obj)，将obj中的值赋值给reg



- “设备存储器结构体”的定义
  - 存放在amdev.h中，使用了AM_DEVREG进行拼接，并创建对应的结构体



- ioe_read()==与架构相关==
  - 存放在对应架构的ioe.c中
  - 接收一个reg和buf，根据reg，在lut（一个指针函数表）中查找到控制设备寄存器的**指针函数**(dev_func=lut[reg])
  - 最终利用typedef，得到返回handler_t类型的函数，即(handler_t)dev_func()
  - 不同设备寄存器有不同的控制函数，将dev中的值提取到buf中并返回
- ioe_write()==与架构相关==
  - 存放在对应架构的ioe.c中
  - 同ioe_read。
  - 不同设备有不同的控制函数，将buf中的值赋值到dev中并返回



- ==总结==
  - 上述程序最终会变为字节码，存放到nemu的指令内存中
  - ioe_read()语句最终会执行paddr_read()、ioe_write()最终会执行paddr_write()
  - 那如何区分ioe的读写和普通读写呢？
    - 在am中规定了不同架构的外设地址，所以ioe的地址范围一定在外设地址内，普通读写一定在非外设地址内。



## 4. 编译与链接

- static变量
  - 在头文件中不要使用static变量，因为static限制可见域为本文件内。
  - 如果在头文件中声明static变量A，那么每一个引用头文件的文件都会存在一个“属于自己的A”，而不是“共同使用的静态变量A”
- static函数
  - 在头文件中不要使用static变量，因为static限制可见域为本文件内。
  - 如果在头文件中声明static函数A，那么每一个引用头文件的文件都会存在一个“属于自己的A函数定义”
  - 此时，如果要所有文件需要调用A，就必须**每一个文件实现一次A函数**
- static inline
  - 在头文件中可以使用static inline。
  - 这时，每一个引用头文件的文件都会存在一个“属于自己的**A函数展开**的定义”
  - 然而此时并不需要提供A的实现，因为inline是在调用点直接展开成函数本体，而不是通过引用的方式来调用函数。
  - ==是一种以空间换时间的方式==
    - 好处：inline减少了函数调用时的花销
    - 坏处：每个文件都需要存放一份“属于自己的**A函数展开**的定义”

## 5. 了解Makefile

- hello处的Makefile
  - 提供程序名hello和文件名hello.c
  - include (AM_HOME)/Makefile
- $(AM_HOME)/Makefile
  - 检查编译环境：正确的AM_HOME、受支持的ARCHS、利用ARCHS计算出完整ISA和PLATFORM字符串
  - 计算编译目标路径与名称
    - 确定工作目录与生成目录
    - 计算出IMAGE、ARCHIVE
    - 计算出OBJS、LINKAGE
  - 规定交叉编译器、编译参数
  - 与架构相关的配置
    - -include \$(AM_HOME)/scripts/\$(ARCHS).mk（-include的意思是：若不存在mk也不会报错）
  - 编译
    - 目标文件生成（.c->.o/.cc->.o/.cpp->.o/.S->.o）
    - 库文件生成（.a）
    - 目标文件与库文件链接（.o+.a->.elf）
    - 库文件生成（.o->.a）
    - 处理依赖关系：编译过程中.h会产生.d的依赖文件，需要将.d添加到gcc的处理中（一般是自动寻找并添加的）
- \$(AM_HOME)/scripts/riscv32e-nemu.mk（以riscv32e-nemu为例）
  - include $(AM_HOME)/scripts/isa/riscv.mk：对编译参数进行第一次修改
  - include $(AM_HOME)/scripts/platform/nemu.mk：对编译参数进行第二次修改
  - 特定的架构还需要override一些特定的编译参数：对编译参数进行第三次修改
  - 将riscv对应的拓展内容加入到AM_SRCS中，才能一起进行编译
- riscv.mk
  - 修改编译参数
  - 将riscv.h包含进ARCH_H，才能一起进行编译
- nemu.mk
  - 修改编译参数
  - 将nemu对应的拓展内容加入到AM_SRCS中，才能一起进行编译
  - ==规定了run规则，当我们使用make ARCH=riscv32e-nemu run时，就会调用这条规则==





## ”理解上下文结构体的前世今生“与”理解穿越时空的旅程“

- yield-os中的main函数
  - cte_init，将schedule函数传递进去，作为cte的handler函数
  - 定义两个context
  - 执行yield()，内部触发了ecall中断：“asm volatile("li a7, -1; ecall");”
- cte_init()
  - 将__am_asm_trap的入口地址传给了mtvec，即当作了异常入口
  - 将传入的handler作为全局的user_handler
- ecall中断流程
  - 当触发中断后，会跳转进异常入口，即init时设置的__am_asm_trap（被放置在trap.S中）
  - trap.S中首先在栈中开辟一个空间作为上下文（地址为[sp-CONTEXT_SIZE,sp]，因为riscv的栈中地址是向下生长的）
  - 将寄存器内容保存到上下文中（即**用软件来保存环境**）
  - 然后trap.S会跳转进__am_irq_handle
    - 前面的”mv a0, sp“是传参的意思（见riscv调用函数的约定）
    - 进行handle处理，实际上内部是传入一个上下文1，然后返回一个上下文2（以实现上下文切换）
    - 后面的”mv sp, a0“是将返回值赋值给sp的意思
  - 用软件来恢复环境
- __am_irq_handle()
  - trap.S调用该函数，并将旧上下文作为参数传入，对应形参为c。
  - 若user_handler存在，则识别c的事件类型，并调用user_handler处理（即cte_init传入的schedule函数）
  - schedule函数会进行上下文的切换，并返回。
  - 于是执行__am_irq_handle，等同于调用schedule函数，实现了上下文切换

