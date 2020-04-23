# Shell

## 传递参数

shell 传递参数时 ${1} 才是第一个参数，${0} 是其命令本身。

```bash
#!/bin/bash
# author:菜鸟教程
# url:www.runoob.com

echo "Shell 传递参数实例！";
echo "执行的文件名：$0";
echo "第一个参数为：$1";
echo "第二个参数为：$2";
echo "第三个参数为：$3";
```



