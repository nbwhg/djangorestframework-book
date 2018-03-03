## 介绍

在本章我们会创建一个简单的API来允许管理员账号可以查看和编辑用户和用户组。

## 项目设置

创建一个新的项目，命名为```tutorial```, 然后开启一个新的app，命名为```quickstart```。
```shell
# 创建项目目录
mkdir tutorial
cd tutorial

# 创建虚拟的环境
mkvirtualenv --no-site-packages -p /usr/local/bin/python3 env

# 在虚拟环境中安装Django和 Django REST framework
pip install django==1.11.10
pip install djangorestframework==3.7.7

# 创建一个新的项目并启动一个APP
django-admin.py startproject tutorial .  # 注意最后的 . 字符
cd tutorial/
django-admin.py startapp quickstart
cd ..
```

此时项目的目录结构如下：
```shell
tree ./
./
├── manage.py
└── tutorial
    ├── __init__.py
    ├── quickstart
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── apps.py
    │   ├── migrations
    │   │   └── __init__.py
    │   ├── models.py
    │   ├── tests.py
    │   └── views.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py

3 directories, 12 files
```
在项目的目录中创建应用，这种方式看起来和常规的方式不同。这里的目的主要为为了利用项目的命名空间，来避免和外部的其他模块造成冲突。

现在，进行首次的数据库同步：
```shell
python manage.py migrate
```

接下来我们创建一个初始的用户，用户名为```admin```, 密码为```password123```。
```shell
python manage.py createsuperuser --email admin@example.com --username admin
```

现在一切就绪，进入app目录，开始编码吧...
