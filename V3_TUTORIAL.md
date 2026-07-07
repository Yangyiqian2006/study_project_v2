# V3 教程：注册登录版学习养成计划怎么理解

## 1. V3 和 V2 有什么区别

V2 是前端 + C++ 后端，但没有用户账号。

所有任务都属于同一个本地系统。

V3 新增了：

1. 注册。
2. 登录。
3. 退出登录。
4. 用户数据隔离。
5. 用户文件 `users_v3.txt`。
6. 任务中新增 `username` 字段。

也就是说，V3 中每个任务都知道自己属于哪个用户。

## 2. 新增的 C++ 数据结构

### 2.1 User 结构体

```cpp
struct User {
    string username;
    string password;
    string displayName;
    string createdDate;
};
```

### 语法解释

`struct User` 定义了一个用户类型。

每个用户有：

- `username`：用户名。
- `password`：密码。
- `displayName`：显示名称。
- `createdDate`：注册日期。

### 项目意义

没有 User，就无法区分谁登录了系统。

有了 User，系统就可以判断用户名密码是否正确。

### 2.2 Task 新增 username

V3 的任务结构体中新增：

```cpp
string username;
```

### 作用

这个字段表示任务属于哪个用户。

例如：

```text
任务 A 的 username = alice
任务 B 的 username = bob
```

alice 登录后，只显示 username 为 alice 的任务。

bob 登录后，只显示 username 为 bob 的任务。

## 3. 新增的全局数据

```cpp
vector<User> users;
vector<Task> tasks;
```

### users

保存所有注册用户。

### tasks

保存所有用户的任务。

注意：所有用户的任务都放在同一个 `tasks` 里，但是每个任务都有 `username` 字段，所以可以按用户筛选。

## 4. 用户文件 users_v3.txt

V3 新增用户保存文件：

```text
users_v3.txt
```

每一行保存一个用户。

格式：

```text
username|password|displayName|createdDate
```

例如：

```text
alice|1234|Alice|2026-07-07
```

## 5. 任务文件 study_tasks_v3.txt

V3 任务文件比 V2 多了 username 字段。

格式：

```text
id|username|title|topic|dueDate|dueTime|priority|needReview|completed|createdDate|completedDate|estimatedMinutes|note
```

这样任务就和用户关联起来了。

## 6. 注册流程

### 前端

用户填写注册表单：

- 用户名
- 显示名称
- 密码

前端调用：

```js
postForm(API.register, { username, displayName, password })
```

### 后端

后端执行：

```cpp
authRegister(body)
```

流程：

1. `parseForm(body)` 解析表单。
2. 检查用户名至少 3 个字符。
3. 检查密码至少 4 个字符。
4. 检查用户名是否已经存在。
5. 创建 User。
6. 加入 `users` 数组。
7. 调用 `saveUsers()` 保存。
8. 返回注册成功 JSON。

## 7. 登录流程

### 前端

用户填写用户名和密码。

前端调用：

```js
postForm(API.login, { username, password })
```

### 后端

后端执行：

```cpp
authLogin(body)
```

流程：

1. 解析用户名和密码。
2. 调用 `findUser(username)` 查找用户。
3. 如果用户不存在，返回失败。
4. 如果密码错误，返回失败。
5. 如果正确，返回用户信息。

### 前端保存当前用户

登录成功后，前端执行：

```js
localStorage.setItem("study_plan_v3_user", JSON.stringify(user));
```

这表示把当前登录用户保存到浏览器本地。

刷新页面后，前端可以恢复登录状态。

## 8. 退出登录流程

前端执行：

```js
localStorage.removeItem("study_plan_v3_user");
```

然后隐藏主页面，显示登录页面。

这就是退出登录。

## 9. 用户数据隔离如何实现

### 获取任务时

前端请求：

```text
GET /api/tasks?username=alice
```

后端只返回 alice 的任务。

### 添加任务时

前端提交表单时带上：

```js
username: currentUser.username
```

后端保存任务时：

```cpp
task.username = username;
```

### 打卡和删除时

后端不只检查任务 id，还检查 username。

```cpp
if (task.id == id && task.username == username)
```

这样用户 A 不能操作用户 B 的任务。

## 10. 为什么没有真正数据库

你目前只学了基础 C++ 和基础数据结构。

所以 V3 仍然使用文本文件保存数据。

这样你可以理解文件读写和数据结构，而不用突然学习数据库。

后续如果升级，可以把：

- `users_v3.txt`
- `study_tasks_v3.txt`

换成 SQLite 数据库。

## 11. 为什么密码明文保存

这是课程学习项目，为了降低难度，密码直接保存在文本文件。

真实项目中不能这样做。

真实项目需要：

1. 密码哈希。
2. 加盐。
3. Session 或 Token。
4. HTTPS。

报告里可以写：

> 当前版本为课程演示系统，密码采用文本方式保存，后续可使用哈希算法进行安全升级。

## 12. V3 答辩说法

可以这样说：

本项目 V3 采用前端 + C++ 后端结构。前端负责登录注册页面、任务页面和数据可视化；后端负责用户注册、登录校验、任务保存、任务筛选、提醒和统计。系统通过 username 字段实现用户与任务的关联，使不同用户登录后只能看到自己的学习任务。
