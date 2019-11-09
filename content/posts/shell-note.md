---
title:  shell note
date: 2017-12-03
categories: [
    "linux",
]
tags: ["shell", "linux"]
---

### shell note

---

#### 查看进程启动时间:

```
ps -A -opid,stime,etime,args

```

#### [sort 命令](https://www.cnblogs.com/zhangsubai/p/6861697.html)

```
grep key file | grep -w 88 | awk -F '[()]' '{map[$4]+=$12}END{for(i in map){print i" "map[i]}}' | sort -un -k2
```

#### 操作数据库

```
result=mysql ${config} -e "${sql}"
```

#### 字符串操作

##### 取数组操作

```
arr=(${out})
eg: out=12:23:34:45:56
arr=(${out//:/}) //12 23 34 45 56 数组
len=${\#arr[@]} // 数组长度
${arr[0]}
```

##### 取字符串长度

```
string=abc12342341 // 等号二边不要有空格
echo ${\#string} // 结果 11
expr length $string // 结果 11
expr "$string" : ".\*" // 结果 11 分号二边要有空格, 这里的: 根 match 的用法差不多
```

##### 字符串所在位置

```
expr index $string '123' // 结果 4 字符串对应的下标是从 0 开始的
```

##### 字符串截取

```
echo ${string:4} //2342341 从第 4 位开始截取后面所有字符串
echo ${string:3:3} //123 从第 3 位开始截取后面 3 位
echo ${string:3:6} //123423 从第 3 位开始截取后面 6 位
echo ${string: -4} //2341 ：右边有空格 截取后 4 位
echo ${string:(-4)} //2341 同上
expr substr $string 3 3 //123 从第 3 位开始截取后面 3 位
```

##### 从字符串开头到子串的最大长度

```
expr match $string 'abc.\*3' // 结果 9
```

##### 匹配显示内容

```
expr match $string '\\([a-c]\*[0-9]\*\\)' //abc12342341
expr $string : '\\([a-c]\*[0-9]\\)' //abc1
expr $string : '.\*\\([0-9][0-9][0-9]\\)' //341 显示括号中匹配的内容
```

##### 截取不匹配的内容

```
echo ${string\#a\*3} //42341 从 $string 左边开始，去掉最短匹配子串
echo ${string\#c\*3} //abc12342341 这样什么也没有匹配到
echo ${string\#\*c1\*3} //42341 从 $string 左边开始，去掉最短匹配子串
echo ${string\#\#a\*3} //41 从 $string 左边开始，去掉最长匹配子串
echo ${string%3\*1} //abc12342 从 $string 右边开始，去掉最短匹配子串
echo ${string%%3\*1} //abc12 从 $string 右边开始，去掉最长匹配子串
```

##### 匹配并且替换

```
echo ${string/23/bb} //abc1bb42341 替换一次
echo ${string//23/bb} //abc1bb4bb41 双斜杠替换所有匹配
echo ${string/\#abc/bb} //bb12342341 \# 以什么开头来匹配，根 php 中的 ^ 有点像
echo ${string/%41/bb} //abc123423bb % 以什么结尾来匹配，根 php 中的 $ 有点像
```