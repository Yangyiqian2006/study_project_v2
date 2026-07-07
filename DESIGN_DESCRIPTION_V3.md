# V3 详细设计说明

## 1. 架构设计

系统分为前端和后端。

前端使用 HTML、CSS、JavaScript。

后端使用 C++ 和 WinSock。

用户通过浏览器访问 `http://localhost:8090`。

后端同时负责两件事：

1. 返回前端静态页面。
2. 处理 `/api/...` 接口。

## 2. 用户模块设计

### 功能

1. 注册用户。
2. 登录用户。
3. 保存用户数据。
4. 校验用户名和密码。

### 数据结构

```cpp
struct User {
    string username;
    string password;
    string displayName;
    string createdDate;
};
```

### 主要函数

| 函数 | 作用 |
|---|---|
| loadUsers | 从 users_v3.txt 读取用户 |
| saveUsers | 保存用户到 users_v3.txt |
| findUser | 根据用户名查找用户 |
| authRegister | 处理注册请求 |
| authLogin | 处理登录请求 |

## 3. 任务模块设计

V3 的任务结构体新增 `username`。

这样每个任务都属于某个用户。

### 主要函数

| 函数 | 作用 |
|---|---|
| loadTasks | 读取任务文件 |
| saveTasks | 保存任务文件 |
| userTasks | 筛选当前用户任务 |
| addTaskAction | 添加任务 |
| completeTaskAction | 完成打卡 |
| deleteTaskAction | 删除任务 |

## 4. 统计模块设计

统计函数 `statsJson(username)` 只统计当前用户任务。

统计内容包括：

1. 总任务数。
2. 已完成数。
3. 完成率。
4. 预计学习时长。
5. 每日添加和完成情况。
6. 主题完成率。
7. 优先级分布。
8. 打卡日历数据。

## 5. 前端模块设计

前端新增登录注册页面。

登录成功后，前端把当前用户保存到 localStorage。

之后所有任务请求都带上当前用户名。

例如：

```js
GET /api/tasks?username=alice
```

## 6. 数据隔离设计

数据隔离通过 username 字段实现。

后端所有任务相关操作都必须带 username。

任务查询、统计、提醒、打卡和删除都会按 username 过滤。

## 7. 后续改进方向

1. 密码加密保存。
2. 使用数据库保存用户和任务。
3. 使用 Session 或 Token 管理登录状态。
4. 增加修改任务功能。
5. 增加管理员账号。
