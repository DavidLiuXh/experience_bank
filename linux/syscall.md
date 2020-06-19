* 所谓的系统调用,简单讲就是kernel提供给用户空间的一组统一的对设备和资源操作的接口, 用来user层和kernel交互, 完成相应的功能, 同时也对kernel层提供了一定的保护
* 用户空间通常不会直接使用系统调用, linux上的C库对所有的系统调用都作了封装,  调用系统调用,需要从用户态切换到内核态, 不同体系结构的系统陷入内核态的方法不同, C库封装了这层差异,这也是推荐直接使用C库的原因;
* 以x86为例, 使用C库来调用系统调用时, 会先通过`int 0x80`软中断,来跳转到相应的中断处理服务例程,即系统调用服务程序`system_call`, `systeml_call`根据系统调用号查找系统调用获取到系统调用服务例程地址并调用之.
* 这样就很清楚了, 如果要增加一个系统调用, 我们只需要:
  1. 先给要增加的系统调用定个名字;
  2. 按linux kernel的规范定义系统调用服务例程;
  3. 要系统调用表里添加系统调用号和系统调用的对应关系;
  4. 重新编译内核;
* 我们心linux kernel 4.14.11为例, 实操一下, 首先需要要相应的内核源码
---
###### 声明系统调用服务例程
* 假设我们新添加的系统调用名字为`hello`
* 打开源码下 `include/linux/syscalls.h`文件, 添加声明:
```
asmlinkage long sys_hello(const char __user *name);
```
其中 
  1. `asmlinkage`即为`extens C`, 按 c的编译方式
  2. 返回值必须是1`long`;
  3. 函数名以`sys_`为前缀;
  4. `__user`表示是从用户空间传递来的参数;
###### 定义系统调用服务例程
* 按理说我们应该提供单独的c文件来写这个系统调用对应的服务例程, 增加新文件,需要更改相应的Makefile文件,我们这里简单处理,借用已有的文件;
* 打开源码下 `kernel/sys.c`, 添加定义:
```
asmlinkage long sys_hello(const char __user *name) {         
        char *name_kd;                                       
        long ret;                                            
                                                             
        name_kd = strndup_user(name, PAGE_SIZE);             
        if (IS_ERR(name_kd)) {                               
             ret = PTR_ERR(name);                            
             goto error;                                     
        }                                                    
                                                             
        printk("Hello, %s!\n", name_kd);                     
        ret = 0;                                             
error:                                                       
        return ret;                                          
}                                                            
```
###### 添加系统调用号
* 打开源码下`arch/x86/entry/syscalls/syscall_64.tbl`, 添加调用号333(根据自己的源码,可自定义):
```
333     64      hello                   sys_hello
```
###### 编译安装新内核并使用新内核重启
* 可参考 [linux-4.14.11 编译](https://www.jianshu.com/p/ea39eba24845)
##### 测试新的系统调用
* 测试代码 test_syscall.c
```
#include <stdio.h>

int main(int argc, char *argv[]) {
   long ret = syscall(333, "lw");
   printf("ret: %d\n", ret);
   return 0;
}
```
* 编译: `gcc -o test_syscall test_syscall.c`
* 运行: ./test_syscall lw
* 使用`dmesg`命令查看,在结尾会有类似下面的显示:
```
[  402.829360] Hello, lw!
```