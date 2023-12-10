---
tags:
  - 6s081
category: 6s081
title: xv6代码及调试
---
##  安装

关于xv6代码的运行首先需要下载一下工具包
可以前往[课程主页](https://pdos.csail.mit.edu/6.828/2022/schedule.html) 查看
在ubuntu系统 执行以下命令
```bash
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 
```

然后将代码clone到本地
```bash
git clone git://g.csail.mit.edu/xv6-labs-2022
```

然后就可以快乐的运行代码了


关于xv6代码的导读可以查看[jyy的视频](https://www.bilibili.com/video/BV1DY4y1a7YD/?spm_id_from=333.999.0.0&vd_source=83ccbbcd32d65afa4747be47eb98ca15)
## vscode调试xv6

在vscode上调试xv6

首先在根目录下创建.vscode文件夹，并创建如下内容的两个文件：launch.json、tasks.json

```json
//launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "debug xv6",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/kernel/kernel",
            "args": [],
            "stopAtEntry": true,
            "cwd": "${workspaceFolder}",
            "miDebuggerServerAddress": "127.0.0.1:25000",
            //这里的端口号需查看.gdbinit 中的端口号
            
            "miDebuggerPath": "/usr/bin/gdb-multiarch",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "preLaunchTask": "xv6build",
            "setupCommands": [
                {
                    "description": "pretty printing",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true,
                },
            ],
            
        }
    ]
}

```


```json
// tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "xv6build",
            "type": "shell",
            "isBackground": true,
            "command": "make qemu-gdb",
            "problemMatcher": [
                {
                    "pattern": [
                        {
                            "regexp": ".",
                            "file": 1,
                            "location": 2,
                            "message": 3
                        }
                    ],
                    "background": {
                        "beginsPattern": ".*Now run 'gdb' in another window.",
                        
                        "endsPattern": "."
                    }
                }
            ]
        }
    ]
}

```

然后执行以下命令
```bash
echo "add-auto-load-safe-path /home/path/xv6-riscv/.gdbinit" >> "/home/yourhome/.gdbinit"
```

最后还需要在.gdbinit中注释掉端口号一行


```bash
set confirm off
set architecture riscv:rv64
# target remote 127.0.0.1:1234
symbol-file kernel/kernel
set disassemble-next-line auto
set riscv use-compressed-breakpoints yes

```

`注意：如果需要在命令行调试需要取消注释这一行`


如果需要调试内核文件可以直接打断点，如果调试用户文件，需要在调试控制台中输入

```bash
-exec file user/_file
```

file即对应的文件。


