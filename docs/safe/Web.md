# Web安全入门

## 靶场搭建

### DVWA

Github：[https://github.com/digininja/DVWA](https://github.com/digininja/DVWA)

Damn Vulnerable Web Application (DVWA)(译注：可以直译为："该死的"不安全Web应用网站)，是一个编码糟糕的、易受攻击的 PHP/MySQL Web应用程序。 它的主要目的是帮助安全专业人员在合法的环境中，测试他们的技能和工具，帮助 Web 开发人员更好地了解如何增强 Web 应用程序的安全性，并帮助学生和教师在受控的课堂环境中，了解 Web 应用程序的安全。

DVWA的具体目标是通过简单明了的界面，来**演练一些最常见的 Web 漏洞**，这些漏洞具有**不同的难度级别**。 请注意，此软件**存在说明和未说明的漏洞**。 这是故意的。 我们鼓励您尝试并发现尽可能多的安全问题

<br />

**（1）使用Docker部署**

```bash
# 启动容器
[root@localhost ~]# docker container run --name dvwa -itd -p 8080:80 --restart always vulnerables/web-dvwa

# 查看容器
[root@localhost ~]# docker container ps
CONTAINER ID   IMAGE                  COMMAND      CREATED          STATUS          PORTS                                   NAMES
909d3a465861   vulnerables/web-dvwa   "/main.sh"   22 seconds ago   Up 21 seconds   0.0.0.0:8080->80/tcp, :::8080->80/tcp   dvwa

# 查看资源占用
[root@localhost ~]# docker container stats --no-stream
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT     MEM %     NET I/O      BLOCK I/O       PIDS
909d3a465861   dvwa      0.22%     108.8MiB / 1.934GiB   5.49%     1.1kB / 0B   156MB / 234MB   37
```

**（2）Web页面登录**

![image-20220707144151261](https://tuchuang-1257805459.cos.accelerate.myqcloud.com//image-20220707144151261.png)

**（3）初始化数据库**

![image-20220707144256505](https://tuchuang-1257805459.cos.accelerate.myqcloud.com//image-20220707144256505.png)

**（4）查看一下基本信息**

```bash
# 系统版本
[root@localhost ~]# docker container exec -it dvwa cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"
VERSION_ID="9"
VERSION="9 (stretch)"
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"

# PHP版本
[root@localhost ~]# docker container exec -it dvwa php --version
PHP 7.0.30-0+deb9u1 (cli) (built: Jun 14 2018 13:50:25) ( NTS )
Copyright (c) 1997-2017 The PHP Group
Zend Engine v3.0.0, Copyright (c) 1998-2017 Zend Technologies
    with Zend OPcache v7.0.30-0+deb9u1, Copyright (c) 1999-2017, by Zend Technologies

# MySQL版本
[root@localhost ~]# docker container exec -it dvwa mysqld --version
mysqld  Ver 10.1.26-MariaDB-0+deb9u1 for debian-linux-gnu on x86_64 (Debian 9.1)
```

### Vulhub

官网：[https://vulhub.org/](https://vulhub.org/)

Github：[https://github.com/vulhub/vulhub](https://github.com/vulhub/vulhub)



## XSS

### 反射型XSS

#### DVWA

::: details （1）DVWA Low级别 反射型XSS

Github：[https://github.com/digininja/DVWA/blob/master/vulnerabilities/xss_r/source/low.php](https://github.com/digininja/DVWA/blob/master/vulnerabilities/xss_r/source/low.php)

![image-20220708063018243](https://tuchuang-1257805459.cos.accelerate.myqcloud.com//image-20220708063018243.png)

:::

::: details （2）使用PHP编写简单的反射型XSS漏洞代码

`index.php`

```php
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>反射型XSS：PHP基础示例</title>
</head>

<body>
    <h1>反射型XSS：PHP基础示例</h1>
    <div>
        <div>
            <!-- HTML表单：GET提交 -->
            <form name="form-get" action="#" method="get">
                <p>
                    What's your name?
                    <input type="text" name="name" autofocus>
                    <input type="submit" value="GET 提交">
                </p>
            </form>

            <!-- 解析GET方法提交的表单数据 -->
            <?php
            if (array_key_exists("name", $_GET) && $_GET['name'] != NULL) {
                echo '<pre style="color: red;">Hello ' . $_GET['name'] . '</pre>';
            }
            ?>
        </div>

        <div>
            <!-- HTML表单：POST提交 -->
            <form name="form-get" action="#" method="post">
                <p>
                    What's your name?
                    <input type="text" name="name">
                    <input type="submit" value="POST提交">
                </p>
            </form>

            <!-- 解析POST方法提交的表单数据 -->
            <?php
            if (array_key_exists("name", $_POST) && $_POST['name'] != NULL) {
                echo '<pre style="color: red;">Hello ' . $_POST['name'] . '</pre>';
            }
            ?>
        </div>
    </div>
</body>

</html>
```

![q4HnuXcU](https://tuchuang-1257805459.cos.accelerate.myqcloud.com//q4HnuXcU.gif)

:::

::: details （3）DVWA Medium级别 反射型XSS

**弹窗测试**

我们直接输入`<script>alert(1)</script>`不会弹窗，如下图所示

![image-20220709065345578](https://tuchuang-1257805459.cos.accelerate.myqcloud.com//image-20220709065345578.png)

**查看源码**

Github：https://github.com/digininja/DVWA/blob/master/vulnerabilities/xss_r/source/medium.php

我们可以看一下源码是如何处理的

```php
<?php

header ("X-XSS-Protection: 0");

// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
	// Get input
	$name = str_replace( '<script>', '', $_GET[ 'name' ] );       // 这里将<script>标签替换为空字符串了

	// Feedback for end user
	$html .= "<pre>Hello ${name}</pre>";                         // 在这里接受的实际上是：alert(1)</script>
}

?>
```

**绕过思路**

```javascript
// 方法1：大小写方式
<Script>alert(1)</script>

// 方法2
<s<script>cript>alert(1)</script>

// 方法3：使用其他标签
<img src="#" onerror=alert(1)>
```

:::

::: details （4）DVWA High级别 反射型XSS

**查看源码**

Github：[https://github.com/digininja/DVWA/blob/master/vulnerabilities/xss_r/source/high.php](https://github.com/digininja/DVWA/blob/master/vulnerabilities/xss_r/source/high.php)

```php
<?php

header ("X-XSS-Protection: 0");

// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
	// Get input
	$name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET[ 'name' ] );	// 使用正则替换，且忽略大小写

	// Feedback for end user
	$html .= "<pre>Hello ${name}</pre>";
}

?>
```

**绕过思路**

```javascript
// 方法1：使用其他标签
<img src="#" onerror=alert(1)>
```

:::

::: details （5）DVWA Impossible级别 反射型XSS

**查看源码**

Github：[https://github.com/digininja/DVWA/blob/master/vulnerabilities/xss_r/source/impossible.php](https://github.com/digininja/DVWA/blob/master/vulnerabilities/xss_r/source/impossible.php)

```php
<?php

// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
	// Check Anti-CSRF token
	checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

	// Get input
	$name = htmlspecialchars( $_GET[ 'name' ] );	// 使用htmlspecialchars函数对输入进行转义

	// Feedback for end user
	$html .= "<pre>Hello ${name}</pre>";
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

**查看htmlspecialchars函数**

文档：[https://www.php.net/manual/en/function.htmlspecialchars.php](https://www.php.net/manual/en/function.htmlspecialchars.php)

`htmlspecialchars`会对`'`、`"`、`<`、`>`、`&`5个特殊符号进行转义，所以这里我们无法绕过，但并不代表使用`htmlspecialchars`就可以保证绝对安全

```php
<?php
echo 'PHP Version: ' . PHP_VERSION . "\n";  // 这里必须使用双引号
echo '参数: 无          ' . htmlspecialchars('" < > \' &') . "\n";
echo '参数: ENT_COMPAT：' . htmlspecialchars('" < > \' &', ENT_COMPAT) . "\n";

// 输出结果
// PHP Version: 8.1.8
// 参数: 无          &quot; &lt; &gt; &#039; &amp;
// 参数: ENT_COMPAT：&quot; &lt; &gt; ' &amp; 
?>
```

**htmlspecialchars可能引起的XSS攻击**

```php
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>反射型XSS：PHP htmlspecialchars XSS示例</title>
</head>

<body>
    <h1>反射型XSS：PHP htmlspecialchars XSS示例</h1>
    <div>
        <div>
            <!-- HTML表单：GET提交 -->
            <form name="form-get" action="#" method="get">
                <p>
                    What's your name?
                    <input type="text" name="name" autofocus>
                    <input type="submit" value="GET 提交">
                </p>
            </form>

            <!-- 解析GET方法提交的表单数据 -->
            <?php
            if (array_key_exists("name", $_GET) && $_GET['name'] != NULL) {

                // 核心是这里：对单引号'不进行编码，不同的PHP版本htmlspecialchars可能会有不同的结果，这里使用8.1.8，其他版本没有测试
                $name = htmlspecialchars($_GET['name'], ENT_COMPAT);                     
                
                echo "使用'可以触发XSS: <input type='text' value='$name'>" . "<br />";     // 这里使用单引号会有XSS漏洞
                echo "使用'无法触发XSS: <input type='text' value=\"$name\">" . "<br />";   // 使用双引号可避免单引号引起的XSS攻击
            }
            ?>
        </div>
    </div>
</body>

</html>
```

在页面中输入`' onclick='alert(1)`，然后点击下面的输入框即可触发`XSS`攻击

:::

#### Flask

**版本介绍**

| 名称       | 版本    | 官网                                                         |
| ---------- | ------- | ------------------------------------------------------------ |
| Flask      | `2.1.2` | [https://flask.palletsprojects.com/en/2.1.x/](https://flask.palletsprojects.com/en/2.1.x/) |
| Jinja2     | `3.1.2` | [https://jinja.palletsprojects.com/en/3.1.x/](https://jinja.palletsprojects.com/en/3.1.x/) |
| MarkupSafe | `2.1.1` | [https://markupsafe.palletsprojects.com/en/2.1.x/](https://markupsafe.palletsprojects.com/en/2.1.x/) |

<br />

**安装Flask**

```bash
pip install flask==2.1.2  # 这会自动安装上jinja2和MarkupSafe
```

<br />

**关于转义 💊**

<span style="background-color: gray; color: white; padding: 0 5px;">1、为什么Flask（实际上是jinja2）没有开启自动转义？</span>

文档：https://jinja.palletsprojects.com/en/3.1.x/faq/#why-is-html-escaping-not-the-default

解释： `jinja2`默认关闭了自动转义； `jinja2`也并不是完全不转义，而是使用了一个第三方库`MarkupSafe`来进行转义

<span style="background-color: gray; color: white; padding: 0 5px;">2、MarkupSafe 是如何做转义的？</span>

文档：[https://markupsafe.palletsprojects.com/en/2.1.x/escaping/#markupsafe.escape](https://markupsafe.palletsprojects.com/en/2.1.x/escaping/#markupsafe.escape)

解释：对 `&`, `<`, `>`, `'`，`"` 5个字符进行转义

3、<span style="background-color: gray; color: white; padding: 0 5px;">3、Jinja2 HTML转义概述</span>

文档：[https://jinja.palletsprojects.com/en/3.1.x/templates/#html-escaping](https://jinja.palletsprojects.com/en/3.1.x/templates/#html-escaping)

解释：

① 手动转义可以使用`jinja2`内置的过滤器`escape`（别名为`e`），比如`{{ user.username|e }}`

② 以下情况会取消转义：模板中传入的是`markupsafe.Markup`对象、模板中使用`jinja2`内置的过滤器`safe`

`escape`过滤器：[https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.escape](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.escape)

safe过滤器：[https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.safe](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.safe)

`forceescape`过滤器：[https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.forceescape](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.forceescape)

<br />

**Flask正确使用示例**

::: details 服务端代码 main.py

```python
#!/usr/bin/env python
# --*--coding:utf-8--*--

from flask import Flask, request, render_template, render_template_string
from jinja2 import Template

app = Flask(__name__)


@app.route('/')
def index():
    # 获取查询字符串
    demo = request.args.get("demo", "1")
    name = request.args.get("name")

    # 读取模板，demo2和demo3会用到
    with open("templates/index.html", mode="r", encoding="utf-8") as f:
        tpl = f.read()

    # flask render_template 示例
    if demo == "1":
        return render_template("index.html", name=name)

    # flask render_template_string 示例
    if demo == "2":
        return render_template_string(tpl, name=name)

    # jinja2 Template 示例
    if demo == "3":
        template = Template(tpl, autoescape=True)  # 这里需要手动转义一下，否则不会有任何转义
        return template.render(name=name)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080, debug=True)
```

:::

::: details HTML模板代码 templates/index.html（注意模板需要放到templates目录下）

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
    <style>
        h1,
        .main {
            width: 410px;
            margin: 0 auto;
        }

        .content {
            height: 50px;
            line-height: 50px;
            border: 1px solid #ccc;
            padding: 20px;
            color: red;
        }
    </style>
</head>
<body>
<h1>反射型XSS：Flask正确示例</h1>
<div class="main">
    <!-- for循环生成3个表单，表单之间使用 demo 字段来区分 -->
    <!-- for循环文档：https://jinja.palletsprojects.com/en/3.1.x/templates/#for -->
    {% for i in [1,2,3]%}
    <div>
        <!-- HTML表单：GET提交 -->
        <form name="form-get" action="" method="get">
            <!-- 隐藏字段，用来区分不同的表单 -->
            <input type="hidden" name="demo" value="{{ i }}">
            <!-- 可显示部分 -->
            <p>
                What's your name?
                <input type="text" name="name" value="<script>alert({{ i }})</script>">
                <input type="submit" value="GET 提交">
            </p>
        </form>
    </div>
    {% endfor %}

    <!-- 显示提交的内容 -->
    <div class="content">
        {% if name%}
        Hello {{ name }}
        {% endif %}
    </div>
</div>
</body>
</html>
```

:::

![image-20220711131953064](https://tuchuang-1257805459.cos.accelerate.myqcloud.com//image-20220711131953064.png)
