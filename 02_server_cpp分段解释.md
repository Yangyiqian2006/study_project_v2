# server.cpp 分段解释

这个文件是 C++ 后端的核心代码。

它不是普通控制台程序，而是一个本地 HTTP 服务。

运行后浏览器访问：

`http://localhost:8090`

## 1. 头文件区域

```cpp
#include <winsock2.h>
#include <ws2tcpip.h>
#include <windows.h>
#include <algorithm>
#include <ctime>
#include <fstream>
#include <iostream>
#include <map>
#include <sstream>
#include <string>
#include <vector>
```

这些是程序需要用到的库。

- `winsock2.h`、`ws2tcpip.h`：网络服务相关。
- `windows.h`：Windows 控制台编码相关。
- `algorithm`：算法函数，例如删除、替换、最大值。
- `ctime`：日期时间。
- `fstream`：文件读写。
- `iostream`：控制台输出。
- `map`：统计用映射表。
- `sstream`：字符串拼接和拆分。
- `string`：字符串。
- `vector`：动态数组。

## 2. Task 结构体

```cpp
struct Task {
    int id;
    string title;
    string topic;
    string dueDate;
    string dueTime;
    int priority;
    bool needReview;
    bool completed;
    string createdDate;
    string completedDate;
    int estimatedMinutes;
    string note;
};
```

这是任务的数据模型。

每个任务都有这些信息：

| 字段 | 含义 |
|---|---|
| id | 任务编号 |
| title | 任务名称 |
| topic | 学习主题 |
| dueDate | 完成日期 |
| dueTime | 完成时间 |
| priority | 优先级，1 低，2 中，3 高 |
| needReview | 是否需要复习提醒 |
| completed | 是否已经完成 |
| createdDate | 创建日期 |
| completedDate | 完成日期 |
| estimatedMinutes | 预计学习时长 |
| note | 备注 |

## 3. Recommendation 结构体

```cpp
struct Recommendation {
    string title;
    string topic;
    int priority;
    int estimatedMinutes;
    string note;
};
```

这是系统推荐任务的数据模型。

推荐任务不是用户真正创建的任务，只是模板。

用户点击“采用”后，推荐任务会填入表单。

## 4. 全局变量

```cpp
const int PORT = 8090;
const string DATA_FILE = "study_tasks_v2.txt";
vector<Task> tasks;
int nextId = 1;
```

含义：

- `PORT`：后端端口号。
- `DATA_FILE`：保存任务的文件名。
- `tasks`：保存所有任务。
- `nextId`：下一个任务编号。

## 5. 日期函数 todayDate

这个函数获取今天日期。

后端需要用今天日期来：

1. 判断任务是否今天截止。
2. 判断任务是否逾期。
3. 记录打卡日期。
4. 计算复习提醒。

## 6. split 函数

`split` 用来拆分字符串。

任务保存到文本文件时，每个字段之间用 `|` 分隔。

读取文件时，就要把一行数据重新拆成多个字段。

## 7. jsonEscape 函数

这个函数用于处理 JSON 特殊字符。

如果任务备注里有双引号，直接拼 JSON 会出错。

所以要把特殊字符转义。

## 8. parseForm 函数

前端添加任务时，会用表单格式提交数据。

例如：

```text
title=复习C++&topic=C++&priority=2
```

`parseForm` 把它解析成 map：

```cpp
data["title"] = "复习C++";
data["topic"] = "C++";
data["priority"] = "2";
```

## 9. validDate 和 validTime

这两个函数用于输入校验。

后端不能完全相信前端传来的数据。

所以后端要自己检查：

- 日期是不是合法。
- 时间是不是合法。
- 优先级是不是 1 到 3。

## 10. loadTasks 函数

程序启动时调用。

作用：从 `study_tasks_v2.txt` 读取历史任务。

如果没有这个函数，每次重启后端任务都会丢失。

## 11. saveTasks 函数

任务发生变化时调用。

例如：

1. 添加任务。
2. 删除任务。
3. 完成打卡。
4. 生成演示数据。

它会把 `tasks` 数组重新写入文件。

## 12. taskJson 函数

把 C++ 中的 Task 结构体转换成 JSON 字符串。

前端不能直接理解 C++ 结构体。

所以后端必须把它转换成前端能理解的 JSON。

## 13. remindersJson 函数

这个函数负责生成提醒。

提醒规则：

1. 逾期任务提醒。
2. 今天截止提醒。
3. 高优先级任务 7 天内提醒。
4. 中优先级任务 3 天内提醒。
5. 低优先级任务 1 天内提醒。
6. 完成后第 1、3、7 天复习提醒。

## 14. statsJson 函数

这个函数负责统计分析。

统计内容包括：

1. 总任务数。
2. 已完成任务数。
3. 未完成任务数。
4. 完成率。
5. 预计学习时长。
6. 每日添加数。
7. 每日完成数。
8. 按主题完成率。
9. 按优先级统计。
10. 打卡日期统计。

## 15. addTaskAction 函数

这是添加任务接口的处理函数。

流程：

1. 解析前端提交的数据。
2. 校验任务是否合法。
3. 创建 Task 结构体。
4. 加入 `tasks` 数组。
5. 保存到文件。
6. 返回成功 JSON。

## 16. completeTaskAction 函数

这是完成打卡接口。

流程：

1. 根据 id 找任务。
2. 如果找到，把 `completed` 改成 true。
3. 把 `completedDate` 改成今天日期。
4. 保存任务文件。
5. 返回成功。

## 17. deleteTaskAction 函数

这是删除任务接口。

它使用 `remove_if` 找到要删除的任务，再用 `erase` 从 vector 中移除。

## 18. sendResponse 函数

这个函数负责把内容发回浏览器。

返回内容可能是：

1. HTML 页面。
2. CSS 样式。
3. JS 脚本。
4. JSON 数据。

## 19. handleClient 函数

这是后端最重要的请求分发函数。

它会判断浏览器请求的路径：

- 如果是 `/api/tasks`，就返回任务。
- 如果是 `/api/stats`，就返回统计。
- 如果是 `/api/reminders`，就返回提醒。
- 如果是 `/api/tasks` 的 POST，就添加任务。
- 如果是普通网页文件，就返回 HTML/CSS/JS。

## 20. main 函数

程序从 main 开始运行。

主要流程：

1. 设置控制台编码。
2. 读取历史任务。
3. 启动 Windows 网络库。
4. 创建 socket。
5. 绑定 8090 端口。
6. 开始监听。
7. 循环等待浏览器请求。
8. 每收到一个请求，就交给 `handleClient` 处理。