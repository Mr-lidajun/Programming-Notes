# 指针Pointer

## void指针和NULL指针

### void指针

* void指针我们把它称之为通用指针，就是可以指向任意类型的数据。也就是说，任何类型的指针都可以赋值给void指针。

```c
#include <stdio.h>

int main() {
    int num = 1024;
    int *pi = &num;
    char *ps = "FishC";
    void *pv;

    pv = pi;
    printf("pi:%p, pv:%p\n", pi, pv);
    printf("*pv:%d\n", *(int *)pv);

//    ps = (char *)pv;

    pv = ps;
    printf("ps:%p, pv:%p\n", ps, pv);
    printf("*pv:%s\n", (char *)pv);

    return 0;
}
```

### NULL指针

> #define NULL ((void *)0)

**养成良好的编程习惯**

* 当你还不清楚要将指针初始化为什么地址时，请将它初始化NULL；在对指针进行解引用时，先检查该指针是否为NULL。这种策略可以为你今后编写大型程序节省大量的调试时间。

**NULL不是NUL**
NULL用于指针和对象，表示指向一个不被使用的地址；而 '\0' (NUL)表示字符串的结尾

```c
#include <stdio.h>
int main() {
    int *p1;
    int *p2 = NULL;

    printf("%d\n", *p1);
    printf("%d\n", *p2);

    return 0;
}
```

## 指向指针的指针

## 指针数组和指向指针的指针

* 至少有两个好处：
  * 避免重复分配内存
  * 只需要进行一次修改
* 代码的灵活性和安全性都有了显著地提高！