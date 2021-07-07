# CPP 编译成动态链接库

##  1. 编译源代码，将输出source.o

```
gcc -c source.cpp
```

## 2. 编译一个动态链接库

```
gcc -shared -o mydll.dll source.o
```