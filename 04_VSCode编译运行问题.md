# VS Code 编译运行常见问题

## 1. 为什么会出现 undefined reference to send@16

因为 C++ 后端用了 Windows 网络函数，例如：

- send
- recv
- socket
- bind
- listen
- accept
- WSAStartup

这些函数属于 Windows 的 `ws2_32` 网络库。

如果编译时没有链接这个库，就会报：

```text
undefined reference to `send@16`
undefined reference to `recv@16`
undefined reference to `WSAStartup@8`
```

## 2. 正确编译命令

必须使用：

```powershell
g++ server.cpp -std=c++17 -finput-charset=UTF-8 -fexec-charset=UTF-8 -o server.exe -lws2_32
```

重点是最后：

```text
-lws2_32
```

## 3. 为什么 exe 会消失

有些情况下，VS Code 或编译器在重新编译时会先清理旧 exe。

如果新编译失败，旧 exe 没了，新 exe 又没生成，就会看起来“exe 消失”。

解决方法：重新用正确命令编译。

## 4. 最稳运行方式

直接双击或运行：

```text
D:\Yiqian_Yang _Homework_c++\project\fullstack_v2\run_backend.bat
```

如果想重新编译：

```text
D:\Yiqian_Yang _Homework_c++\project\fullstack_v2\compile_backend.bat
```

## 5. VS Code 里怎么编译

打开 `fullstack_v2` 文件夹。

按：

```text
Ctrl + Shift + B
```

它会读取 `.vscode/tasks.json`，使用正确命令编译。

## 6. 双击 exe 输出乱码怎么办

我已经在后端 main 函数中加入：

```cpp
SetConsoleOutputCP(CP_UTF8);
SetConsoleCP(CP_UTF8);
```

并且 bat 文件里加入：

```bat
chcp 65001
```

这样中文输出会尽量正常。

如果你电脑的控制台字体不支持中文，可以换成“新宋体”或“Consolas + 中文回退字体”。