## MMAP 从0到放弃

### 储备知识

#### 寻址



#### 虚拟地址空间

### 内存空间分类

#### 内核空间

是操作系统内核访问的区域，独立于普通的应用程序，是受保护的内存空间

#### 用户空间

是普通应用程序可访问的内存区域

#### 内核地址映射模型

内存映射文件的方法

### MMAP

#### 函数原型

`void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);`

