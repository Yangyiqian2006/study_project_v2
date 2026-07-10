# 学习养成计划 V3：所有功能实现路径详解

本文按“功能”拆解代码路径。每个功能都按下面顺序讲：

```text
HTML 页面入口
→ app.js 绑定事件/读取数据
→ app.js 请求后端或渲染页面
→ server.cpp 接收请求
→ server.cpp 处理内存数据/文件数据
→ 返回 JSON 或静态文件
→ app.js 更新页面
```

四个核心文件：

```text
frontend/index.html
frontend/styles.css
frontend/app.js
backend/server.cpp
```

---

## 0. 项目启动与四个文件连接

### 0.1 bat 启动后端

`双击运行项目.bat`：

```bat
@echo off
chcp 65001 >nul
cd /d "%~dp0backend"
start "" "http://localhost:8090"
server.exe
pause
```

逐句解释：

```bat
cd /d "%~dp0backend"
```

进入项目的 `backend` 文件夹。因为 `server.exe` 在这里。

```bat
start "" "http://localhost:8090"
```

打开浏览器访问 `http://localhost:8090`。

```bat
server.exe
```

启动 C++ 后端程序。

### 0.2 C++ 后端监听 8090 端口

`server.cpp`：

```cpp
const int PORT = 8090;
```

表示后端服务端口是 `8090`。

```cpp
SOCKET serverSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
```

创建 TCP socket。

```cpp
addr.sin_family = AF_INET;
addr.sin_addr.s_addr = INADDR_ANY;
addr.sin_port = htons(PORT);
```

设置监听地址：

- `AF_INET`：IPv4。
- `INADDR_ANY`：接收本机所有网卡来的请求。
- `htons(PORT)`：把 8090 转成网络字节序。

```cpp
bind(serverSocket, reinterpret_cast<sockaddr*>(&addr), sizeof(addr))
```

把 socket 绑定到 8090 端口。

```cpp
listen(serverSocket, SOMAXCONN);
```

开始监听浏览器请求。

```cpp
while (true) {
    SOCKET client = accept(serverSocket, nullptr, nullptr);
    if (client != INVALID_SOCKET) {
        handleClient(client);
        closesocket(client);
    }
}
```

循环等待浏览器连接。每来一个请求，就交给 `handleClient(client)` 处理。

### 0.3 后端返回 index.html

浏览器访问：

```text
GET / HTTP/1.1
```

`server.cpp` 进入：

```cpp
void handleClient(SOCKET client) {
    char buffer[32768];
    int received = recv(client, buffer, sizeof(buffer) - 1, 0);
```

`recv()` 从浏览器读取 HTTP 请求。

```cpp
buffer[received] = '\0';
string request(buffer);
```

把浏览器发来的字节变成 C++ 字符串。

```cpp
string firstLine = request.substr(0, request.find("\r\n"));
```

取 HTTP 请求第一行，比如：

```text
GET / HTTP/1.1
```

```cpp
string method, rawPath;
stringstream line(firstLine);
line >> method >> rawPath;
```

解析出：

```cpp
method = "GET";
rawPath = "/";
```

```cpp
string path = cleanPath(rawPath);
```

去掉查询参数。这里仍然是：

```cpp
path = "/";
```

如果不是 API，就进入静态文件逻辑：

```cpp
string filePath = path == "/" ? "../frontend/index.html" : "../frontend" + path;
```

当 `path == "/"` 时：

```cpp
filePath = "../frontend/index.html";
```

```cpp
string fileBody = readFile(filePath);
```

读取 `index.html` 文件内容。

```cpp
sendResponse(client, fileBody, contentType(filePath));
```

把 HTML 内容返回给浏览器。

### 0.4 浏览器继续请求 CSS 和 JS

`index.html` 中：

```html
<link rel="stylesheet" href="styles.css" />
```

浏览器看到这行，会请求：

```text
GET /styles.css HTTP/1.1
```

后端拼出：

```cpp
filePath = "../frontend" + "/styles.css";
```

也就是：

```text
../frontend/styles.css
```

`index.html` 中：

```html
<script src="app.js"></script>
```

浏览器会请求：

```text
GET /app.js HTTP/1.1
```

后端拼出：

```text
../frontend/app.js
```

---

## 1. 静态文件返回功能

### 1.1 读取文件

`server.cpp`：

```cpp
string readFile(const string& path) {
    ifstream in(path, ios::binary);
    if (!in) return "";
    stringstream ss;
    ss << in.rdbuf();
    return ss.str();
}
```

解释：

```cpp
ifstream in(path, ios::binary);
```

按二进制方式打开文件。

```cpp
if (!in) return "";
```

打不开就返回空字符串。

```cpp
ss << in.rdbuf();
```

把整个文件读进字符串流。

```cpp
return ss.str();
```

返回文件内容。

### 1.2 判断 Content-Type

```cpp
string contentType(const string& path) {
    if (path.find(".css") != string::npos) return "text/css; charset=utf-8";
    if (path.find(".js") != string::npos) return "application/javascript; charset=utf-8";
    if (path.find(".html") != string::npos) return "text/html; charset=utf-8";
    return "text/plain; charset=utf-8";
}
```

解释：

- 路径包含 `.css`，告诉浏览器这是 CSS。
- 路径包含 `.js`，告诉浏览器这是 JavaScript。
- 路径包含 `.html`，告诉浏览器这是 HTML。
- 否则按普通文本处理。

### 1.3 拼 HTTP 响应

```cpp
void sendResponse(SOCKET client, const string& body, const string& type, int status = 200) {
    string statusText = status == 200 ? "OK" : (status == 404 ? "Not Found" : "Bad Request");
    stringstream response;
    response << "HTTP/1.1 " << status << " " << statusText << "\r\n"
             << "Content-Type: " << type << "\r\n"
             << "Access-Control-Allow-Origin: *\r\n"
             << "Access-Control-Allow-Methods: GET, POST, OPTIONS\r\n"
             << "Access-Control-Allow-Headers: Content-Type\r\n"
             << "Content-Length: " << body.size() << "\r\n"
             << "Connection: close\r\n\r\n" << body;
    string text = response.str();
    send(client, text.c_str(), static_cast<int>(text.size()), 0);
}
```

关键点：

```cpp
HTTP/1.1 200 OK
```

告诉浏览器请求成功。

```cpp
Content-Type: text/html; charset=utf-8
```

告诉浏览器内容格式。

```cpp
Content-Length
```

告诉浏览器正文长度。

```cpp
\r\n\r\n
```

HTTP 头和正文之间的空行。

```cpp
send(...)
```

把响应发回浏览器。

---

## 2. 用户注册功能

### 2.1 HTML 入口

```html
<form id="registerForm" class="auth-form hidden">
  <label>用户名<input id="registerUsername" required placeholder="至少 3 个字符" /></label>
  <label>显示名称<input id="registerDisplayName" placeholder="可以填写昵称" /></label>
  <label>密码<input id="registerPassword" type="password" required placeholder="至少 4 个字符" /></label>
  <button class="primary-btn" type="submit">注册并登录</button>
</form>
```

用户输入：

- 用户名：`registerUsername`
- 显示名称：`registerDisplayName`
- 密码：`registerPassword`

### 2.2 JS 找到元素

```js
registerForm: $("#registerForm"),
registerUsername: $("#registerUsername"),
registerDisplayName: $("#registerDisplayName"),
registerPassword: $("#registerPassword"),
```

这里 `$()` 等价于：

```js
document.querySelector(...)
```

### 2.3 JS 绑定注册事件

```js
el.registerForm.addEventListener("submit", event => {
  event.preventDefault();
  register(
    el.registerUsername.value.trim(),
    el.registerDisplayName.value.trim(),
    el.registerPassword.value
  );
});
```

解释：

```js
event.preventDefault()
```

阻止表单刷新页面。

```js
value.trim()
```

读取输入框内容，并去掉前后空格。

### 2.4 JS 请求后端

```js
async function register(username, displayName, password) {
  const result = await postForm(API.register, { username, displayName, password });
  if (!result.ok) {
    el.authMessage.textContent = result.message || "注册失败";
    return;
  }
  saveCurrentUser(result.user);
  enterApp();
}
```

`API.register` 是：

```js
register: "/api/register"
```

所以请求地址是：

```text
POST /api/register
```

### 2.5 postForm 如何提交

```js
async function postForm(url, data = {}) {
  const params = new URLSearchParams();
  Object.entries(data).forEach(([key, value]) => params.append(key, value));
  const response = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: params.toString()
  });
  if (!response.ok) throw new Error("后端提交失败");
  return response.json();
}
```

如果输入：

```js
username = "alice";
displayName = "小爱";
password = "1234";
```

请求 body 会变成：

```text
username=alice&displayName=%E5%B0%8F%E7%88%B1&password=1234
```

### 2.6 C++ 接收注册请求

```cpp
else if (method == "POST" && path == "/api/register")
    sendResponse(client, authRegister(body), "application/json; charset=utf-8");
```

如果请求路径是 `/api/register`，调用：

```cpp
authRegister(body)
```

### 2.7 C++ 解析注册数据

```cpp
string authRegister(const string& body) {
    map<string, string> data = parseForm(body);
    string username = data.count("username") ? cleanField(data["username"]) : "";
    string password = data.count("password") ? cleanField(data["password"]) : "";
    string displayName = data.count("displayName") && !data["displayName"].empty()
        ? cleanField(data["displayName"])
        : username;
```

解释：

```cpp
parseForm(body)
```

把 `username=alice&password=1234` 拆成 `map`。

```cpp
cleanField(...)
```

清理用户输入，避免 `|`、换行破坏 txt 存储格式。

### 2.8 C++ 校验注册数据

```cpp
if (username.size() < 3) return "{\"ok\":false,\"message\":\"用户名至少 3 个字符\"}";
if (password.size() < 4) return "{\"ok\":false,\"message\":\"密码至少 4 个字符\"}";
if (findUser(username)) return "{\"ok\":false,\"message\":\"用户名已存在\"}";
```

分别检查：

- 用户名长度。
- 密码长度。
- 用户名是否重复。

### 2.9 C++ 保存用户

```cpp
User user{username, password, displayName, todayDate()};
users.push_back(user);
saveUsers();
return "{\"ok\":true,\"user\":" + userJson(user) + "}";
```

`User` 结构体：

```cpp
struct User {
    string username;
    string password;
    string displayName;
    string createdDate;
};
```

`saveUsers()`：

```cpp
void saveUsers() {
    ofstream out(USER_FILE);
    for (const User& user : users) {
        out << cleanField(user.username) << '|'
            << cleanField(user.password) << '|'
            << cleanField(user.displayName) << '|'
            << user.createdDate << '\n';
    }
}
```

保存到：

```cpp
const string USER_FILE = "users_v3.txt";
```

文件格式：

```text
username|password|displayName|createdDate
```

---

## 3. 用户登录功能

### 3.1 HTML 入口

```html
<form id="loginForm" class="auth-form">
  <label>用户名<input id="loginUsername" required placeholder="请输入用户名" /></label>
  <label>密码<input id="loginPassword" type="password" required placeholder="请输入密码" /></label>
  <button class="primary-btn" type="submit">登录</button>
</form>
```

### 3.2 JS 绑定登录事件

```js
el.loginForm.addEventListener("submit", event => {
  event.preventDefault();
  login(el.loginUsername.value.trim(), el.loginPassword.value);
});
```

### 3.3 JS 调用登录接口

```js
async function login(username, password) {
  const result = await postForm(API.login, { username, password });
  if (!result.ok) {
    el.authMessage.textContent = result.message || "登录失败";
    return;
  }
  saveCurrentUser(result.user);
  enterApp();
}
```

`API.login`：

```js
login: "/api/login"
```

请求：

```text
POST /api/login
```

### 3.4 C++ 接收登录

```cpp
else if (method == "POST" && path == "/api/login")
    sendResponse(client, authLogin(body), "application/json; charset=utf-8");
```

### 3.5 C++ 验证账号密码

```cpp
string authLogin(const string& body) {
    map<string, string> data = parseForm(body);
    string username = data.count("username") ? data["username"] : "";
    string password = data.count("password") ? data["password"] : "";
    User* user = findUser(username);
    if (!user || user->password != password)
        return "{\"ok\":false,\"message\":\"用户名或密码错误\"}";
    return "{\"ok\":true,\"user\":" + userJson(*user) + "}";
}
```

```cpp
findUser(username)
```

在 `users` 数组里找用户：

```cpp
User* findUser(const string& username) {
    for (User& user : users)
        if (user.username == username) return &user;
    return nullptr;
}
```

### 3.6 前端进入应用

```js
saveCurrentUser(result.user);
enterApp();
```

保存当前用户：

```js
function saveCurrentUser(user) {
  currentUser = user;
  localStorage.setItem("study_plan_v3_user", JSON.stringify(user));
}
```

进入主页面：

```js
async function enterApp() {
  el.authView.classList.add("hidden");
  el.appView.classList.remove("hidden");
  el.userLabel.textContent = `当前用户：${currentUser.displayName || currentUser.username}`;
  el.serverState.textContent = `账号：${currentUser.username}`;
  await loadAll();
}
```

---

## 4. 自动登录状态保存功能

### 4.1 页面启动时读取 localStorage

`app.js` 最后：

```js
currentUser = readCurrentUser();
if (currentUser) enterApp();
```

### 4.2 读取函数

```js
function readCurrentUser() {
  const raw = localStorage.getItem("study_plan_v3_user");
  if (!raw) return null;
  try { return JSON.parse(raw); } catch { return null; }
}
```

解释：

```js
localStorage.getItem(...)
```

从浏览器本地存储读取之前保存的用户字符串。

```js
JSON.parse(raw)
```

把字符串还原成对象。

---

## 5. 退出登录功能

### 5.1 HTML 入口

```html
<button id="logoutBtn" class="ghost-btn full-btn">退出登录</button>
```

### 5.2 JS 绑定点击

```js
el.logoutBtn.addEventListener("click", logout);
```

### 5.3 JS 退出逻辑

```js
function logout() {
  localStorage.removeItem("study_plan_v3_user");
  currentUser = null;
  tasks = [];
  el.appView.classList.add("hidden");
  el.authView.classList.remove("hidden");
  switchAuthMode("login");
}
```

逐句解释：

```js
localStorage.removeItem(...)
```

删除浏览器保存的登录状态。

```js
currentUser = null;
tasks = [];
```

清空当前用户和任务。

```js
el.appView.classList.add("hidden");
```

隐藏主应用。

```js
el.authView.classList.remove("hidden");
```

显示登录页。

---

## 6. 登录/注册 Tab 切换功能

### 6.1 HTML 入口

```html
<button id="loginTab" class="auth-tab active">登录</button>
<button id="registerTab" class="auth-tab">注册</button>
```

### 6.2 JS 绑定

```js
el.loginTab.addEventListener("click", () => switchAuthMode("login"));
el.registerTab.addEventListener("click", () => switchAuthMode("register"));
```

### 6.3 切换逻辑

```js
function switchAuthMode(mode) {
  const isLogin = mode === "login";
  el.loginTab.classList.toggle("active", isLogin);
  el.registerTab.classList.toggle("active", !isLogin);
  el.loginForm.classList.toggle("hidden", !isLogin);
  el.registerForm.classList.toggle("hidden", isLogin);
  el.authMessage.textContent = "";
}
```

解释：

```js
classList.toggle("active", isLogin)
```

`isLogin` 为 true 时添加 `active`，为 false 时删除。

```js
authMessage.textContent = ""
```

切换 tab 时清空错误提示。

---

## 7. 加载全部数据功能

### 7.1 调用时机

`loadAll()` 会在这些时候调用：

```js
enterApp()
addTaskFromForm()
completeTask()
deleteTask()
resetDemoData()
refreshBtn 点击
```

### 7.2 具体代码

```js
async function loadAll() {
  if (!currentUser) return;
  try {
    const [taskData, statData, reminderData, recommendationData] = await Promise.all([
      getJson(API.tasks + userQuery()),
      getJson(API.stats + userQuery()),
      getJson(API.reminders + userQuery()),
      getJson(API.recommendations)
    ]);
    tasks = taskData;
    stats = statData;
    recommendations = recommendationData;
    renderAll(reminderData);
  } catch (error) {
    el.taskList.innerHTML = `<article class="task-card">无法连接 C++ 后端，请先运行 backend/server.exe。</article>`;
  }
}
```

解释：

```js
Promise.all([...])
```

同时请求任务、统计、提醒、推荐。

```js
tasks = taskData;
stats = statData;
recommendations = recommendationData;
```

把后端返回数据存到前端变量。

```js
renderAll(reminderData);
```

重新渲染页面。

---

## 8. 用户自定义任务添加功能

### 8.1 HTML 表单

```html
<form id="taskForm" class="panel form-panel">
  <label>任务名称<input id="titleInput" required /></label>
  <label>学习主题<input id="topicInput" required /></label>
  <input id="dateInput" type="date" required />
  <input id="timeInput" type="time" required />
  <select id="priorityInput">
    <option value="3">高</option>
    <option value="2" selected>中</option>
    <option value="1">低</option>
  </select>
  <input id="minuteInput" type="number" min="5" max="600" value="30" />
  <textarea id="noteInput"></textarea>
  <input id="reviewInput" type="checkbox" />
  <button type="submit">添加任务</button>
</form>
```

### 8.2 设置默认输入

```js
function setDefaultInputs() {
  el.dateInput.value = today();
  el.timeInput.value = "20:00";
  el.priorityInput.value = "2";
  el.minuteInput.value = "30";
}
```

页面启动时调用：

```js
setDefaultInputs();
```

### 8.3 提交事件

```js
el.taskForm.addEventListener("submit", addTaskFromForm);
```

### 8.4 读取输入并提交

```js
async function addTaskFromForm(event) {
  event.preventDefault();
  const result = await postForm(API.tasks, {
    username: currentUser.username,
    title: el.titleInput.value.trim(),
    topic: el.topicInput.value.trim(),
    dueDate: el.dateInput.value,
    dueTime: el.timeInput.value,
    priority: el.priorityInput.value,
    estimatedMinutes: el.minuteInput.value,
    note: el.noteInput.value.trim(),
    needReview: el.reviewInput.checked ? "true" : "false"
  });
```

每个字段和后端 `Task` 对应：

| 前端字段 | 后端字段 |
|---|---|
| `title` | `task.title` |
| `topic` | `task.topic` |
| `dueDate` | `task.dueDate` |
| `dueTime` | `task.dueTime` |
| `priority` | `task.priority` |
| `estimatedMinutes` | `task.estimatedMinutes` |
| `note` | `task.note` |
| `needReview` | `task.needReview` |

### 8.5 后端接收

```cpp
else if (method == "POST" && path == "/api/tasks")
    sendResponse(client, addTaskAction(body), "application/json; charset=utf-8");
```

### 8.6 后端校验

```cpp
string validateTaskInput(const map<string, string>& data) {
    if (!data.count("title") || data.at("title").empty()) return "任务名称不能为空";
    if (!data.count("topic") || data.at("topic").empty()) return "学习主题不能为空";
    if (!data.count("dueDate") || !validDate(data.at("dueDate"))) return "完成日期格式错误";
    if (!data.count("dueTime") || !validTime(data.at("dueTime"))) return "完成时间格式错误";
    if (!data.count("priority")) return "优先级不能为空";
    int priority = stoi(data.at("priority"));
    if (priority < 1 || priority > 3) return "优先级只能是 1 到 3";
    return "";
}
```

### 8.7 后端创建任务

```cpp
Task task;
task.id = nextTaskId++;
task.username = username;
task.title = cleanField(data["title"]);
task.topic = cleanField(data["topic"]);
task.dueDate = data["dueDate"];
task.dueTime = data["dueTime"];
task.priority = stoi(data["priority"]);
task.needReview = data.count("needReview") && data["needReview"] == "true";
task.completed = false;
task.createdDate = todayDate();
task.completedDate = "-";
task.estimatedMinutes = data.count("estimatedMinutes") && !data["estimatedMinutes"].empty() ? stoi(data["estimatedMinutes"]) : 30;
task.note = data.count("note") ? cleanField(data["note"]) : "";
tasks.push_back(task);
saveTasks();
```

解释：

```cpp
completed = false
```

新任务默认未完成。

```cpp
completedDate = "-"
```

还没有完成日期。

```cpp
createdDate = todayDate()
```

记录任务添加日期，用于每日统计。

---

## 9. 系统推荐任务功能

### 9.1 后端定义推荐任务

```cpp
vector<Recommendation> recommendations() {
    return {
        {"背诵英语单词 30 个", "英语", 2, 35, "适合每天碎片时间完成"},
        {"完成一道数据结构练习题", "数据结构", 3, 45, "建议先画图再写代码"},
        {"复习 C++ 文件读写", "C++", 2, 40, "重点理解 ifstream 和 ofstream"},
        {"整理今天的课堂笔记", "学习总结", 1, 25, "把知识点整理成提纲"},
        {"完成一页错题归纳", "复盘", 2, 30, "记录错误原因和正确思路"},
        {"阅读教材 10 页", "自主学习", 1, 30, "读完后写三句话总结"}
    };
}
```

### 9.2 前端请求推荐任务

```js
getJson(API.recommendations)
```

`API.recommendations`：

```js
"/api/recommendations"
```

### 9.3 前端渲染推荐任务

```js
function renderRecommendations() {
  el.recommendList.innerHTML = recommendations.map((item, index) => `
    <article class="recommend-card">
      <div><strong>${item.title}</strong>
      <div class="meta">
        <span class="badge">${item.topic}</span>
        <span class="badge ${priorityClass(item.priority)}">${priorityName(item.priority)}优先级</span>
        <span class="badge">${item.estimatedMinutes} 分钟</span>
      </div>
      <p class="task-note">${item.note}</p></div>
      <button class="small-btn" data-recommend="${index}">采用</button>
    </article>`).join("");
}
```

### 9.4 采用推荐任务

```js
el.recommendList.addEventListener("click", event => {
  const button = event.target.closest("[data-recommend]");
  if (button) useRecommendation(Number(button.dataset.recommend));
});
```

```js
function useRecommendation(index) {
  const item = recommendations[index];
  if (!item) return;
  el.titleInput.value = item.title;
  el.topicInput.value = item.topic;
  el.priorityInput.value = String(item.priority);
  el.minuteInput.value = item.estimatedMinutes;
  el.noteInput.value = item.note;
  el.reviewInput.checked = item.priority >= 2;
  switchView("tasks");
}
```

这里只是填表，不保存。保存仍然走添加任务功能。

---

## 10. 任务筛选功能

### 10.1 HTML 按钮

```html
<button class="filter active" data-filter="all">全部</button>
<button class="filter" data-filter="todo">未完成</button>
<button class="filter" data-filter="done">已完成</button>
<button class="filter" data-filter="high">高优先级</button>
```

### 10.2 当前筛选状态

```js
let currentFilter = "all";
```

默认显示全部。

### 10.3 点击筛选按钮

```js
$$(".filter").forEach(button => button.addEventListener("click", () => {
  currentFilter = button.dataset.filter;
  $$(".filter").forEach(item => item.classList.remove("active"));
  button.classList.add("active");
  renderTasks();
}));
```

解释：

```js
button.dataset.filter
```

读取按钮上的 `data-filter`。

```js
renderTasks()
```

重新渲染任务列表。

### 10.4 筛选逻辑

```js
function filteredTasks() {
  if (currentFilter === "todo") return tasks.filter(task => !task.completed);
  if (currentFilter === "done") return tasks.filter(task => task.completed);
  if (currentFilter === "high") return tasks.filter(task => task.priority === 3);
  return tasks;
}
```

解释：

- `todo`：只要未完成。
- `done`：只要已完成。
- `high`：只要高优先级。
- 其他情况：全部任务。

---

## 11. 任务列表渲染功能

### 11.1 HTML 容器

```html
<div id="taskList" class="task-list"></div>
```

### 11.2 渲染函数

```js
function renderTasks() {
  const visible = filteredTasks();
  if (visible.length === 0) {
    el.taskList.innerHTML = `<article class="task-card">当前筛选条件下暂无任务。</article>`;
    return;
  }
```

先筛选任务。没有任务就显示空状态。

```js
el.taskList.innerHTML = visible.map(task => `
  <article class="task-card ${task.completed ? "done" : ""}">
    <div>
      <div class="task-title">${task.title}</div>
      <div class="meta">
        <span class="badge">${task.topic}</span>
        <span class="badge ${priorityClass(task.priority)}">${priorityName(task.priority)}优先级</span>
        <span class="badge">截止 ${task.dueDate} ${task.dueTime}</span>
        <span class="badge">${task.estimatedMinutes} 分钟</span>
        <span class="badge">${task.needReview ? "需要复习" : "无需复习"}</span>
        <span class="badge">${task.completed ? `已完成 ${task.completedDate}` : "未完成"}</span>
      </div>
      ${task.note ? `<p class="task-note">${task.note}</p>` : ""}
    </div>
    <div class="actions">
      <button class="small-btn done" data-complete="${task.id}" ${task.completed ? "disabled" : ""}>打卡</button>
      <button class="small-btn delete" data-delete="${task.id}">删除</button>
    </div>
  </article>`).join("");
}
```

重点：

```js
task.completed ? "done" : ""
```

完成后给卡片加 `done` 样式。

```js
data-complete="${task.id}"
```

把任务 id 放到打卡按钮上。

```js
data-delete="${task.id}"
```

把任务 id 放到删除按钮上。

---

## 12. 打卡功能

### 12.1 点击打卡按钮

```js
el.taskList.addEventListener("click", event => {
  const completeButton = event.target.closest("[data-complete]");
  const deleteButton = event.target.closest("[data-delete]");
  if (completeButton) completeTask(Number(completeButton.dataset.complete));
  if (deleteButton) deleteTask(Number(deleteButton.dataset.delete));
});
```

`closest("[data-complete]")` 找用户点击位置附近的打卡按钮。

### 12.2 前端请求后端

```js
async function completeTask(id) {
  const result = await postForm(`/api/tasks/${id}/complete${userQuery()}`);
  if (result.ok) showToast("打卡成功");
  await loadAll();
}
```

如果 `id=3`，用户是 `alice`：

```text
/api/tasks/3/complete?username=alice
```

### 12.3 后端接收

```cpp
else if (method == "POST" && path.find("/api/tasks/") == 0 && path.find("/complete") != string::npos)
    sendResponse(client, completeTaskAction(idFromPath(path, "/complete"), queryValue(rawPath, "username")), "application/json; charset=utf-8");
```

### 12.4 提取 id

```cpp
int idFromPath(const string& path, const string& suffix) {
    size_t start = string("/api/tasks/").size();
    size_t end = path.find(suffix);
    if (end == string::npos || end <= start) return -1;
    return stoi(path.substr(start, end - start));
}
```

对于：

```text
/api/tasks/3/complete
```

提取出：

```cpp
3
```

### 12.5 修改任务状态

```cpp
string completeTaskAction(int id, const string& username) {
    Task* task = findTask(id, username);
    if (!task) return "{\"ok\":false,\"message\":\"任务不存在\"}";
    task->completed = true;
    task->completedDate = todayDate();
    saveTasks();
    return "{\"ok\":true}";
}
```

核心：

```cpp
completed = true
completedDate = todayDate()
saveTasks()
```

---

## 13. 删除任务功能

### 13.1 前端删除按钮

```html
<button class="small-btn delete" data-delete="${task.id}">删除</button>
```

### 13.2 前端删除函数

```js
async function deleteTask(id) {
  if (!confirm("确定删除这个任务吗？")) return;
  const result = await postForm(`/api/tasks/${id}/delete${userQuery()}`);
  if (result.ok) showToast("任务已删除");
  await loadAll();
}
```

```js
confirm(...)
```

浏览器确认框。取消就不删除。

### 13.3 后端接收

```cpp
else if (method == "POST" && path.find("/api/tasks/") == 0 && path.find("/delete") != string::npos)
    sendResponse(client, deleteTaskAction(idFromPath(path, "/delete"), queryValue(rawPath, "username")), "application/json; charset=utf-8");
```

### 13.4 后端删除

```cpp
string deleteTaskAction(int id, const string& username) {
    size_t oldSize = tasks.size();
    tasks.erase(remove_if(tasks.begin(), tasks.end(), [id, username](const Task& task) {
        return task.id == id && task.username == username;
    }), tasks.end());
    if (tasks.size() == oldSize) return "{\"ok\":false,\"message\":\"任务不存在\"}";
    saveTasks();
    return "{\"ok\":true}";
}
```

解释：

```cpp
remove_if(...)
```

找到满足条件的任务：

```cpp
task.id == id && task.username == username
```

既要 id 相同，也要属于当前用户。

```cpp
tasks.erase(...)
```

真正从数组中删除。

---

## 14. 任务提醒功能

### 14.1 前端请求

```js
getJson(API.reminders + userQuery())
```

请求：

```text
/api/reminders?username=alice
```

### 14.2 后端接收

```cpp
else if (method == "GET" && path == "/api/reminders")
    sendResponse(client, remindersJson(username), "application/json; charset=utf-8");
```

### 14.3 未完成任务提醒规则

```cpp
for (const Task& task : tasks) {
    if (task.username != username || task.completed) continue;
    int diff = dateToDays(task.dueDate) - todayDays;
    string reason;
    if (diff < 0) reason = "已逾期";
    else if (diff == 0) reason = "今天截止";
    else if (task.priority == 3 && diff <= 7) reason = "高优先级，建议每天推进";
    else if (task.priority == 2 && diff <= 3) reason = "中优先级，截止时间较近";
    else if (task.priority == 1 && diff <= 1) reason = "低优先级，临近截止";
    if (!reason.empty()) items.push_back(task.title + "：" + reason);
}
```

精确规则：

- 逾期：`diff < 0`
- 今天截止：`diff == 0`
- 高优先级 7 天内提醒。
- 中优先级 3 天内提醒。
- 低优先级 1 天内提醒。

### 14.4 复习提醒规则

```cpp
for (const Task& task : tasks) {
    if (task.username != username || !task.completed || !task.needReview || task.completedDate == "-") continue;
    int passed = todayDays - dateToDays(task.completedDate);
    if (passed == 1 || passed == 3 || passed == 7)
        items.push_back(task.title + " 已完成 " + to_string(passed) + " 天，建议复习");
}
```

条件：

- 当前用户任务。
- 已完成。
- `needReview == true`。
- 完成后第 1、3、7 天。

### 14.5 前端显示提醒

```js
function renderReminders(reminders) {
  el.reminderList.innerHTML = reminders.length
    ? reminders.map(text => `<div class="reminder">${text}</div>`).join("")
    : `<div class="reminder">今天没有需要提醒的任务。</div>`;
}
```

---

## 15. 统计数据功能

### 15.1 前端请求统计

```js
getJson(API.stats + userQuery())
```

请求：

```text
/api/stats?username=alice
```

### 15.2 后端统计总览

```cpp
int total = 0, completed = 0, minutes = 0;
map<string, pair<int, int>> daily;
map<string, pair<int, int>> topic;
map<int, int> priorityCount;
map<string, int> punchDays;
```

这些变量分别保存：

- 总任务数。
- 完成任务数。
- 总预计分钟数。
- 每日添加/完成。
- 每主题添加/完成。
- 每优先级数量。
- 每天打卡数量。

### 15.3 遍历任务统计

```cpp
for (const Task& task : tasks) {
    if (task.username != username) continue;
    total++;
    daily[task.createdDate].first++;
    topic[task.topic].first++;
    priorityCount[task.priority]++;
    minutes += task.estimatedMinutes;
    if (task.completed) {
        completed++;
        daily[task.createdDate].second++;
        topic[task.topic].second++;
        if (task.completedDate != "-") punchDays[task.completedDate]++;
    }
}
```

解释：

```cpp
if (task.username != username) continue;
```

只统计当前用户。

```cpp
daily[task.createdDate].first++;
```

当天添加数 +1。

```cpp
daily[task.createdDate].second++;
```

当天完成数 +1。

```cpp
punchDays[task.completedDate]++;
```

某天打卡次数 +1。

### 15.4 计算完成率

```cpp
int rate = total == 0 ? 0 : completed * 100 / total;
```

总数为 0 时完成率为 0，避免除以 0。

---

## 16. 顶部统计卡片和进度条

### 16.1 HTML 容器

```html
<strong id="totalCount">0</strong>
<strong id="doneCount">0</strong>
<strong id="rateCount">0%</strong>
<strong id="minuteCount">0</strong>
<div class="big-progress"><div id="mainProgress"></div></div>
<div id="priorityBars" class="mini-bars"></div>
```

### 16.2 渲染

```js
function renderMetrics() {
  el.totalCount.textContent = stats.total;
  el.doneCount.textContent = stats.completed;
  el.rateCount.textContent = `${stats.rate}%`;
  el.minuteCount.textContent = stats.estimatedMinutes;
  el.mainProgress.style.width = `${stats.rate}%`;
  el.progressHint.textContent = `已完成 ${stats.completed}/${stats.total}`;
```

解释：

```js
textContent
```

修改文字。

```js
style.width
```

修改进度条宽度。

### 16.3 优先级条

```js
const max = Math.max(1, ...stats.priority.map(item => item.count));
el.priorityBars.innerHTML = stats.priority.map(item => {
  const width = Math.round(item.count * 100 / max);
  return `<div class="mini-row"><strong>${item.name}优先级</strong><div class="mini-track"><div class="mini-fill" style="width:${width}%"></div></div><span>${item.count}</span></div>`;
}).join("");
}
```

`max` 是最大优先级数量。  
`width` 是每条柱子的百分比。

---

## 17. 打卡记录表

### 17.1 HTML

```html
<tbody id="recordTable"></tbody>
```

### 17.2 渲染

```js
function renderRecords() {
  const doneTasks = tasks
    .filter(task => task.completed)
    .sort((a, b) => b.completedDate.localeCompare(a.completedDate));
```

解释：

```js
filter(task => task.completed)
```

只取已完成任务。

```js
sort(...)
```

按完成日期倒序排列。

```js
el.recordTable.innerHTML = doneTasks.length
  ? doneTasks.map(task => `<tr><td>${task.completedDate}</td><td>${task.title}</td><td>${task.topic}</td><td>${task.estimatedMinutes} 分钟</td></tr>`).join("")
  : `<tr><td colspan="4">暂无打卡记录</td></tr>`;
}
```

把已完成任务变成表格行。

---

## 18. 打卡日历

### 18.1 HTML

```html
<input id="monthInput" type="month" />
<div id="calendar" class="calendar"></div>
```

### 18.2 月份变化

```js
el.monthInput.addEventListener("change", renderCalendar);
```

用户切换月份时重新画日历。

### 18.3 渲染日历

```js
function renderCalendar() {
  const value = el.monthInput.value || monthNow();
  const [year, month] = value.split("-").map(Number);
  const firstDay = new Date(year, month - 1, 1).getDay();
  const totalDays = new Date(year, month, 0).getDate();
```

解释：

```js
value = "2026-07"
```

```js
split("-").map(Number)
```

得到：

```js
year = 2026;
month = 7;
```

```js
firstDay
```

本月 1 号是星期几。

```js
totalDays
```

本月有多少天。

### 18.4 打卡日期映射

```js
const hit = new Map();
stats.punchDays.forEach(item => {
  if (item.date.startsWith(value))
    hit.set(Number(item.date.slice(8, 10)), item.count);
});
```

把后端返回的：

```js
{ date: "2026-07-10", count: 2 }
```

变成：

```js
hit.set(10, 2)
```

### 18.5 生成日历格子

```js
const cells = ["日", "一", "二", "三", "四", "五", "六"]
  .map(name => `<div class="day-name">${name}</div>`);
for (let i = 0; i < firstDay; i++)
  cells.push(`<div class="day-cell empty"></div>`);
for (let day = 1; day <= totalDays; day++) {
  const count = hit.get(day) || 0;
  cells.push(`<div class="day-cell ${count ? "hit" : ""}">${day}${count ? `<small>${count}</small>` : ""}</div>`);
}
el.calendar.innerHTML = cells.join("");
}
```

如果某天有打卡，添加 `hit` 类。

---

## 19. 每日任务分析

### 19.1 后端生成 daily

```cpp
daily[task.createdDate].first++;
```

添加数 +1。

```cpp
daily[task.createdDate].second++;
```

完成数 +1。

```cpp
int dayRate = added == 0 ? 0 : done * 100 / added;
```

计算当天完成率。

### 19.2 前端渲染 daily

```js
el.dailyStats.innerHTML = stats.daily.length
  ? stats.daily.map(item => `<article class="stat-card"><span>${item.date}</span><strong>${item.rate}%</strong><div class="day-track"><div class="day-fill" style="width:${item.rate}%"></div></div><p class="subtle">添加 ${item.added} 个，完成 ${item.done} 个</p></article>`).join("")
  : `<article class="stat-card">暂无每日统计数据。</article>`;
```

每个日期显示：

- 日期。
- 完成率。
- 进度条。
- 添加数。
- 完成数。

---

## 20. 主题完成情况分析

### 20.1 后端生成 topic

```cpp
topic[task.topic].first++;
```

某主题任务总数 +1。

```cpp
topic[task.topic].second++;
```

某主题完成数 +1。

```cpp
int topicRate = count == 0 ? 0 : done * 100 / count;
```

计算主题完成率。

### 20.2 前端渲染 topic

```js
el.topicStats.innerHTML = stats.topic.length
  ? stats.topic.map(item => `<article class="topic-card"><strong>${item.topic}</strong><div class="topic-track"><div class="topic-fill" style="width:${item.rate}%"></div></div><span>${item.rate}%</span></article>`).join("")
  : `<article class="topic-card">暂无主题统计数据。</article>`;
```

每个主题显示：

- 主题名。
- 完成率条。
- 完成率数字。

---

## 21. 生成演示数据功能

### 21.1 HTML 按钮

```html
<button id="resetDemoBtn" class="ghost-btn">生成演示数据</button>
```

### 21.2 JS 绑定

```js
el.resetDemoBtn.addEventListener("click", resetDemoData);
```

### 21.3 JS 请求

```js
async function resetDemoData() {
  if (!confirm("会用演示数据覆盖当前账号的任务，确定继续吗？")) return;
  await postForm(API.resetDemo + userQuery());
  showToast("当前账号的演示数据已生成");
  await loadAll();
}
```

请求：

```text
/api/demo/reset?username=alice
```

### 21.4 后端处理

```cpp
string resetDemoData(const string& username) {
    if (username.empty()) return "{\"ok\":false,\"message\":\"请先登录\"}";
    tasks.erase(remove_if(tasks.begin(), tasks.end(), [username](const Task& task) {
        return task.username == username;
    }), tasks.end());
```

删除当前用户已有任务。

```cpp
vector<Recommendation> seed = recommendations();
for (size_t i = 0; i < 4; ++i) {
    Task task;
    task.id = nextTaskId++;
    task.username = username;
    task.title = seed[i].title;
    task.topic = seed[i].topic;
    task.dueDate = todayDate();
    task.dueTime = i % 2 == 0 ? "20:00" : "21:30";
    task.priority = seed[i].priority;
    task.needReview = i % 2 == 0;
    task.completed = i == 0;
    task.createdDate = todayDate();
    task.completedDate = task.completed ? todayDate() : "-";
    task.estimatedMinutes = seed[i].estimatedMinutes;
    task.note = seed[i].note;
    tasks.push_back(task);
}
saveTasks();
return "{\"ok\":true}";
}
```

用推荐任务前 4 条生成演示数据。

---

## 22. 视图切换功能

### 22.1 HTML 导航

```html
<button class="nav-item active" data-view="dashboard">总览</button>
<button class="nav-item" data-view="tasks">任务</button>
<button class="nav-item" data-view="records">打卡</button>
<button class="nav-item" data-view="analysis">分析</button>
```

### 22.2 HTML 页面区块

```html
<section id="dashboardView" class="view active">
<section id="tasksView" class="view">
<section id="recordsView" class="view">
<section id="analysisView" class="view">
```

### 22.3 CSS 控制显示

```css
.view { display: none; }
.view.active { display: block; }
```

### 22.4 JS 切换

```js
function switchView(viewName) {
  $$(".nav-item").forEach(item =>
    item.classList.toggle("active", item.dataset.view === viewName)
  );
  $$(".view").forEach(view => view.classList.remove("active"));
  $(`#${viewName}View`).classList.add("active");
  const titleMap = {
    dashboard: "学习总览",
    tasks: "任务管理",
    records: "打卡记录",
    analysis: "数据分析"
  };
  el.pageTitle.textContent = titleMap[viewName];
}
```

如果点击任务：

```js
viewName = "tasks"
```

则：

```js
$(`#${viewName}View`)
```

变成：

```js
$("#tasksView")
```

任务页面显示。

---

## 23. Toast 提示功能

### 23.1 HTML

```html
<div id="toast" class="toast"></div>
```

### 23.2 CSS

```css
.toast {
  position: fixed;
  right: 24px;
  bottom: 24px;
  transform: translateY(90px);
  opacity: 0;
  transition: .22s ease;
}
.toast.show {
  transform: translateY(0);
  opacity: 1;
}
```

默认隐藏。加 `show` 后出现。

### 23.3 JS

```js
function showToast(message) {
  el.toast.textContent = message;
  el.toast.classList.add("show");
  setTimeout(() => el.toast.classList.remove("show"), 1800);
}
```

解释：

```js
textContent = message
```

设置提示文字。

```js
classList.add("show")
```

显示 toast。

```js
setTimeout(...)
```

1.8 秒后隐藏。

---

## 24. 数据文件读写功能

### 24.1 启动时读取

```cpp
loadUsers();
loadTasks();
```

### 24.2 读取用户

```cpp
void loadUsers() {
    users.clear();
    ifstream in(USER_FILE);
    string line;
    while (getline(in, line)) {
        vector<string> p = split(line, '|');
        if (p.size() != 4) continue;
        users.push_back({p[0], p[1], p[2], p[3]});
    }
}
```

每行格式：

```text
username|password|displayName|createdDate
```

### 24.3 读取任务

```cpp
void loadTasks() {
    tasks.clear();
    nextTaskId = 1;
    ifstream in(TASK_FILE);
    string line;
    while (getline(in, line)) {
        vector<string> p = split(line, '|');
        if (p.size() != 13) continue;
        Task task;
        task.id = stoi(p[0]);
        task.username = p[1];
        task.title = p[2];
        task.topic = p[3];
        task.dueDate = p[4];
        task.dueTime = p[5];
        task.priority = stoi(p[6]);
        task.needReview = p[7] == "1";
        task.completed = p[8] == "1";
        task.createdDate = p[9];
        task.completedDate = p[10];
        task.estimatedMinutes = stoi(p[11]);
        task.note = p[12];
        tasks.push_back(task);
        nextTaskId = max(nextTaskId, task.id + 1);
    }
}
```

`nextTaskId` 保证新任务 id 不重复。

### 24.4 保存任务

```cpp
void saveTasks() {
    ofstream out(TASK_FILE);
    for (const Task& task : tasks) {
        out << task.id << '|'
            << cleanField(task.username) << '|'
            << cleanField(task.title) << '|'
            << cleanField(task.topic) << '|'
            << task.dueDate << '|'
            << task.dueTime << '|'
            << task.priority << '|'
            << (task.needReview ? 1 : 0) << '|'
            << (task.completed ? 1 : 0) << '|'
            << task.createdDate << '|'
            << task.completedDate << '|'
            << task.estimatedMinutes << '|'
            << cleanField(task.note) << '\n';
    }
}
```

任务文件每行 13 个字段。

---

## 25. 整体功能调用总图

```text
浏览器打开 localhost:8090
  ↓
server.cpp 返回 index.html / styles.css / app.js
  ↓
app.js 初始化元素、绑定事件
  ↓
用户登录/注册
  ↓
postForm() 请求 C++ API
  ↓
server.cpp handleClient() 分发请求
  ↓
authLogin / authRegister / addTaskAction / completeTaskAction / statsJson 等函数处理
  ↓
saveUsers / saveTasks 写入 txt
  ↓
sendResponse() 返回 JSON
  ↓
response.json() 解析
  ↓
loadAll() 更新 tasks/stats/recommendations
  ↓
renderAll() 重新画页面
```

