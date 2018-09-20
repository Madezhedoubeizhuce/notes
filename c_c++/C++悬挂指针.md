# C++悬挂指针

C++ delete指针后，只会把内存标记为可使用，内存在未被重新使用之前也许还能正常访问呢，delete而且不负责把指针设置为空，所以指针依旧可以访问到那段内存。

```c++
char *str = new char[100];
delete[] str;
strcpy(str,"nihao");
cout<<str<<endl;
```


运行结果：

    nihao

