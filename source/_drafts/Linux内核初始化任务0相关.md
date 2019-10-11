### move_to_user_mode

1. 将堆栈指针保存到eax。
2. 将堆栈段选择符入栈，将堆栈段指针入栈。
3. 将标志寄存器eflags内容入栈。
4. 将内核代码段选择符cs入栈，l1的偏移地址eip入栈。
5. 



```c
//// 切换到用户模式运行。
// 该函数利用iret 指令实现从内核模式切换到用户模式（初始任务0）。
#define move_to_user_mode() \
_asm { \
	_asm mov eax,esp /* 保存堆栈指针esp 到eax 寄存器中。*/\
	_asm push 00000017h /* 首先将堆栈段选择符(SS)入栈。*/\
	_asm push eax /* 然后将保存的堆栈指针值(esp)入栈。*/\
	_asm pushfd /* 将标志寄存器(eflags)内容入栈。*/\
	_asm push 0000000fh /* 将内核代码段选择符(cs)入栈。*/\
	_asm push offset l1 /* 将下面标号l1 的偏移地址(eip)入栈。*/\
	_asm iretd /* 执行中断返回指令，则会跳转到下面标号1 处。*/\
_asm l1: mov eax,17h /* 此时开始执行任务0，*/\
	_asm mov ds,ax /* 初始化段寄存器指向本局部表的数据段。*/\
	_asm mov es,ax \
	_asm mov fs,ax \
	_asm mov gs,ax \
}
```

### fork



```c
#define _syscall0(type,name) \
type name(void) \
{ \
	volatile long __res; \
	_asm {  /* 输入为系统中断调用号__NR_name*/\
		_asm mov eax,__NR_##name\
		_asm int 80h /* 调用系统中断0x80。*/\
		_asm mov __res,eax /* 返回值??eax(__res)*/\
	} \
    if (__res >= 0) 		/* 如果返回值>=0，则直接返回该值。*/\
		return (type) __res; \
	errno = -__res; 	/* 否则置出错号，并返回-1。*/\
	return -1; \
}
```

### init函数

#### setup函数

