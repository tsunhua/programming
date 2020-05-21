# Shell

## screen

GNU Screen是一款由GNU计划开发的用于命令行终端切换的自由软件。用户可以通过该软件同时连接多个本地或远程的命令行会话，并在其间自由切换。

（1）安装

参考：

1. [https://superuser.com/questions/681490/install-screen-from-source-code](https://superuser.com/questions/681490/install-screen-from-source-code)
2. [https://www.gnu.org/software/screen/manual/html\_node/Compiling-Screen.html\#Compiling-Screen](https://www.gnu.org/software/screen/manual/html_node/Compiling-Screen.html#Compiling-Screen)

```bash
# install package ncurses-devel
sudo yum install ncurses-devel
# install new version screen
wget https://ftp.gnu.org/gnu/screen/screen-4.6.0.tar.gz
tar -xzf screen-4.6.0.tar.gz
cd screen-4.6.0
sh ./configure
make
cp screen /usr/bin
```

（2）使用示例，部署 Go 服务

```bash
#!/bin/bash -e
echo "Selected Code Branch：$GIT_BRANCH"
echo "Building Go SERVER"
go build -v

echo "Deploy Server"
cd ..
COMMAND_GO_SERVER=./ec-solution/app
go_server_pid=`pidof ${COMMAND_GO_SERVER}`
if [  $go_server_pid ]
 then
      date +"%Y/%m/%d %H:%M:%S-Go Server Restart"
      kill -9 $go_server_pid
      sudo screen -d -m -L -Logfile ./log/ec-solution.log ${COMMAND_GO_SERVER}
      date +"%Y/%m/%d %H:%M:%S-Restart completed"
else
      sudo screen -d -m -L -Logfile ./log/ec-solution.log ${COMMAND_GO_SERVER}
       date +"%Y/%m/%d %H:%M:%S-Start completed"
fi
```

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



