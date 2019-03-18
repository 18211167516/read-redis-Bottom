## 前言：本文是对redis 简单字符串数据结构SDS读后解读

1. 定义
```
struct sdshdr {
  int len;//buf数组中已使用字节长度，等于sds字符串保存的长度
  int free;//buf数组中未使用的字节的数量
  char buf[];//，字节数组，保留字符串
}
```
