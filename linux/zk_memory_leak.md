###### 现象
- 线上 nginx + php-fpm来实时处理请求, php处理请求时需加载我们写的扩展;
- 发现每次请求处理完都有少量的内存泄漏, 因为是线上实时服务, 长时间运行的话此内存泄漏不可忽视;
###### 使用 valgrind 排查
- 执行命令: `valgrind --tool=memcheck  --leak-check=full --log-file=./v.log php  test.php`, 其中`test.php`模拟线上单次请求处理;
- log分析:
```
==15320== 30 (24 direct, 6 indirect) bytes in 1 blocks are definitely lost in loss record 4 of 18  
==15320==    at 0x4A04A28: calloc (vg_replace_malloc.c:467)                                        
==15320==    by 0xB571B1B: deserialize_String_vector (zookeeper.jute.c:245)                        
==15320==    by 0xB571C69: deserialize_GetChildrenResponse (zookeeper.jute.c:874)                  
==15320==    by 0xB56C3A4: zookeeper_process (zookeeper.c:1904)                                    
==15320==    by 0xB5733CE: do_io (mt_adaptor.c:439)                                                
==15320==    by 0x38CC607A50: start_thread (in /lib64/libpthread-2.12.so)                          
==15320==    by 0xC5A06FF: ???                                                                     
==15320==                                                                                          
==15320== LEAK SUMMARY:                                                                            
==15320==    definitely lost: 24 bytes in 1 blocks                                                 
==15320==    indirectly lost: 6 bytes in 3 blocks                                                  
==15320==      possibly lost: 0 bytes in 0 blocks                                                  
==15320==    still reachable: 21,762 bytes in 29 blocks                                            
==15320==         suppressed: 0 bytes in 0 blocks                                                  
==15320== Reachable blocks (those to which a pointer was found) are not shown.                     
==15320== To see them, rerun with: --leak-check=full --show-reachable=yes                          
==15320==                                                                                          
==15320== For counts of detected and suppressed errors, rerun with: -v                             
==15320== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 6 from 6)     
```
可以看到 `definitely lost: 24 bytes in 1 blocks `
###### 解决
- 按 valgrind的log查过去, 应该是调用zk的`zoo_get_children`所至, 代码如下:
```
String_vector children;
 if (ZOK == zoo_get_children(zk_handle_,                                                                                                                                                                
                            node_path.c_str(),                                                                                                                                                                     
                            0,                                                                                                                                                                                     
                            &children)) {
}
```
- 我们来看下 `String_vector`(在generated/zookeeper.jute.h中)的结构:
```
struct String_vector {
    int32_t count;
    char * *data;
};
```
实际上表示一个字符串数组, `count`:包含的字符串个数,`data`: 字符串数组的指针, 那么问题就很明显了,`zoo_get_children`中分配了`data`数组的内存, 又分配了`data`里包含的每个字符串的内存, 但没有释放;
- 使用 `deallocate_String_vector`(在generated/zookeeper.jute.h中)来释放内存, 再次运行 ``valgrind --tool=memcheck  --leak-check=full --log-file=./v.log php  test.php`:
```
==13244== Memcheck, a memory error detector                                                                                                                                                                        
==13244== Copyright (C) 2002-2010, and GNU GPL'd, by Julian Seward et al.                                                                                                                                          
==13244== Using Valgrind-3.6.0 and LibVEX; rerun with -h for copyright info                                                                                                                                        
==13244== Command: php tt.php                                                                                                                                                                                      
==13244== Parent PID: 10470                                                                                                                                                                                        
==13244==                                                                                                                                                                                                          
==13244==                                                                                                                                                                                                          
==13244== HEAP SUMMARY:                                                                                                                                                                                            
==13244==     in use at exit: 21,762 bytes in 29 blocks                                                                                                                                                            
==13244==   total heap usage: 8,779 allocs, 8,750 frees, 3,134,735 bytes allocated                                                                                                                                 
==13244==                                                                                                                                                                                                          
==13244== LEAK SUMMARY:                                                                                                                                                                                            
==13244==    definitely lost: 0 bytes in 0 blocks                                                                                                                                                                  
==13244==    indirectly lost: 0 bytes in 0 blocks                                                                                                                                                                  
==13244==      possibly lost: 0 bytes in 0 blocks                                                                                                                                                                  
==13244==    still reachable: 21,762 bytes in 29 blocks                                                                                                                                                            
==13244==         suppressed: 0 bytes in 0 blocks                                                                                                                                                                  
==13244== Reachable blocks (those to which a pointer was found) are not shown.                                                                                                                                     
==13244== To see them, rerun with: --leak-check=full --show-reachable=yes                                                                                                                                          
==13244==                                                                                                                                                                                                          
==13244== For counts of detected and suppressed errors, rerun with: -v                                                                                                                                             
==13244== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 6 from 6) 
```
问题解决~~~