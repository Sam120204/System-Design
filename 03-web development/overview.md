# Web 应用基础

## Web应用

1. 浏览器发送一个HTTP request
2. 服务器收到请求，生成一个HTML document
3. 服务器把HTML文档作为HTTP response的payload发送给浏览器
4. 浏览器收到HTTP响应，从HTTP Body取出HTML文档并显示

## 路由和视图

最简单的情况下，你需要一个index.html页面。index页面可以只包含一行
```HTML
<h1>Hello World!</h1>
```
访问这个页面，你还需要一个url，比如www.XYZ.com/index.html
/ 前面叫domain name 域名，后面叫path路径

### 路由和视图

url是可变的，比如 https://www.facebook.com/profile.php?id=XXXX
我们看到的页面也会随之变化，这叫做路由
路由会处理URL并调用对应的视图函数
```JS
add_url_rule(‘/’, home_page)
add_url_rule(‘/profile’, user_profile_page)
```

### 动态路由

Url不仅是可变的，甚至还可以传递参数
比如 https://www.facebook.com/profile.php?id=XXXX
```JS
add_url_rule(‘/profile’, user_profile_page)

function profile(id: int):
    return ‘<h1>Hello, user: {id} </h1>’
```

## Request - Response

Web服务器会监听一个端口，通常是80（HTTP协议）
客户端（浏览器）会发送request请求到80端口
服务器收到请求后，根据url mapping查找对应的视图函数。调用视图函数后，会将对应的页面作为response响应的内容

要注意HTTP协议不仅包含response body，header中的状态码也很重要：200（成功）， 404， 302（重定向）

## 视图函数

视图函数的作用：生成request对应的response
举个例子：用户注册账户

1. 用户在web form表单中输入电子邮件地址和密码，然后点击提交按钮。
2. 服务器接收到包含用户输入数据的请求，把请求分派给视图函数。
3. 视图函数需要
    1. 访问数据库，添加新用户
    2. 然后生成响应回送浏览器，指明操作成功还是失败。

最后两步分别是业务逻辑和表现逻辑

## 模板

混用业务逻辑和表现逻辑，会导致代码难以理解和维护。我们需要将表现逻辑和业务逻辑解耦decoupling。

最简单的解决方案是：前端页面是一个HTML文件，并且大部分内容是固定的。所以我们可以把固定的部分提取出来，作为模板文件template。
其中动态部分使用placeholder占位符，比如
```JS
<h1>Hello, user: {user.username} </h1>
```
使用真实值替换变量，再返回最终得到的响应字符串，这一过程称为渲染rendering。

## 数据库

关系型数据库把数据存储在表中

我们的业务逻辑需要操作数据库
方案1. 数据库接受SQL接口，我们在业务逻辑中加入SQL语句，调用数据库执行:
```SQL
INSERT INTO users VALUES (1, xxx@gmail.com, passwd123);
```

方案2: 在数据库和业务代码中间加入一个抽象层，叫做ORM对象关系映射。ORM隐藏了数据库操作，我们在业务代码中操作对象:
```python
db.session.add(user); db.session.commit();
```

## MVC

我们利用template解藕了表现代码和业务代码，在利用ORM解藕了业务代码和数据库
这就构成了一个简单的MVC架构 (model - view - controller)，这是一种非常常用的web项目架构

+ 模型（Model）：封装数据（以及对数据的处理）
+ 视图（View）：实现数据的展示
+ 控制器（Controller）：处理事件并作出响应

### Design Data Entity

回顾一下Function requirement：
+ 用户需要注册登录
+ 关注别的用户
+ 看到别的用户发的贴

用两张模型表存储数据实体，一张关联表记录多对多关系
+ User表
+ Post表
+ Follow表

展示某个用户关注的其他用户最近发表的10篇post
```SQL
SELECT p.id, p.title, p.body, p.user_id, p.created_at
FROM posts p
JOIN follows f ON p.user_id = f.followed_user_id
WHERE f.following_user_id = <User ID>
ORDER BY p.created_at DESC
LIMIT 10 OFFSET 0;
```

### 提供RESTful API

在REST架构中，资源就是一切。在我们的网站中，用户、发帖都是资源。
我们在之前使用了template作为视图层。这种方案的缺点是我们模糊了服务器和客户端的界限。模板必须先在服务器端渲染，才能作为网页返回给客户端。

RESTful 的方案仅在服务器端返回资源，由客户端选择视图层方案（web页面，移动app，甚至手表弹窗，语音播报...）

### 获取用户时间线
```
GET /api/timeline?user={userId}
```
请求参数：
+ userId：用户ID，路径参数，必需。
+ page：页码，查询参数，可选，默认为1。
+ pageSize：每页的记录数，查询参数，可选，默认为10。

通过提供RESTful API，前端可以方便地调用接口，获取某个用户关注的其他用户发表的帖子，并实现分页展示的功能。

### 发布帖子
```
POST /api/posts

header：
+ Content-Type: application/json
+ Authorization: Bearer <token>

body:
{ "title": "帖子标题", "body": "帖子内容" }
```
Authorization 字段用于传递用户的身份认证令牌（token），以确保只有经过身份验证的用户才能发布帖子。令牌通常在用户登录后获取，并在后续的请求中携带。
