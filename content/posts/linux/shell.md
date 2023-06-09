---
title: "Shell 学习笔记"
date: 2023-06-07T14:20:33+08:00
draft: false
categories: ["Linux"]
tags: [Linux,Shell]
---
记录了 shell 语法的学习笔记。

<!--more-->

![hello world](/img/shell.png)

## 应用场景
- 自动化任务
- 定时任务

## 主要内容

### 变量

- 区分大小写
- 变量名大写（惯例）
- 通过`$`使用变量：`$NAME`
- 变量名包含英文、数字、下划线，以英文或者下划线开头

```shell
#!/bin/bash
MY_NAME1="Madhav Bahl"
MY_NAME2="Madhav Bahl"
echo "Hello, I am $MY_NAME1"
echo "Hello, I am ${MY_NAME2}"
```

我们也可以将命令的输出传递给变量：
```shell
#!/bin/bash
SERVER_NAME=$(hostname)
```

### 用户输入

```shell
#!/bin/bash
read -p "Please Enter Your Name: " NAME
echo "Your Name Is $NAME"
```

### 流程控制

```shell
#!/bin/bash
MYPASSWORD="123456"
read -p "Please Enter Your Password: " PASSWORD
echo "Your Password Is $PASSWORD"
if [ $PASSWORD -eq $MYPASSWORD ]
then
        echo "Success!"
else
        echo "Password Is Invalid!"
fi

```

### 循环

找出当前目录下所有以`.sh`结尾的文件（ls只支持通配符*）
```shell
#!/bin/bash
FILES=$(ls *.sh)
for FILE in $FILES
do
	echo "File Name Is: $FILE"
done
```

### Case

直接看文档

### While

直接看文档


### 命令行参数

```shell
$ ./script.sh param1 param2 param3 param4
```

```text
$0 -- "script.sh"
$1 -- "param1"
$2 -- "param2"
$3 -- "param3"
$4 -- "param4"
$@ -- array of all positional parameters
```

### 退出代码

- 0 means success
- Any code other than 0 means an error condition.

`$?`表示先前执行的命令的返回代码

```shell
#!/bin/bash
HOST="google.com"
ping -c 1 $HOST     # -c is used for count, it will send the request, number of times mentioned
RETURN_CODE=$?
if [ "$RETURN_CODE" -eq "0" ]
then
  echo "$HOST reachable"
else
  echo "$HOST unreachable"
fi
```

### 逻辑运算符

- `&&`
- `||`

```shell
mkdir tempDir && cd tempDir && mkdir subTempDir
```

### 函数

```shell
#!/bin/bash
function myFunc () {
    echo "Shell Scripting Is Fun!"
}
myFunc
```

### 通配符

转义字符：`\`

- `*`
- `?`：匹配一个字符
- `[]`
- `[!]`