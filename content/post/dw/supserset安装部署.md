---
title: "Superset安装部署"
date: 2020-08-11T01:37:56+08:00
lastmod: 2020-08-11T01:37:56+08:00
draft: false
tags: ["SQL", "优化"]
categories: ["SQL"]
author: "ChavinKing"
---

1、安装python环境

superset运行要求python3.6环境

1）安装miniconda 
下载地址：https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

2）安装miniconda

sh Miniconda3-latest-Linux-x86_64.sh

是否每次登陆自动激活base环境：conda config --set auto_activate_base false

3）创建python3.6环境

配置conda国内镜像 
bin/conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free 
bin/conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main 
bin/conda config --set show_channel_urls yes

创建python3.6环境 
bin/conda create --name superset python=3.6

附件：
conda环境管理命令 
创建环境：conda create -n env_name 
查看所有环境：conda info -envs 
删除环境: conda remove -n env_name --all

4）激活superset环境：
bin/conda activate superset

停止superset环境：
bin/conda deactivate

5）查看python版本信息

(superset) [root@chavin miniconda3]# python
Python 3.6.10 |Anaconda, Inc.| (default, Mar 25 2020, 23:51:54) 
[GCC 7.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
\>>>


2、安装superset

1）安装superset环境依赖

sudo yum upgrade python-setuptools
sudo yum install gcc gcc-c++ libffi-devel python-devel python-pip python-wheel openssl-devel cyrus-sasl-devel openldap-devel

2）通过获取最新的pip 和setuptools库，将所有机会都放在您的身边。：

bin/pip install --upgrade setuptools pip -i https://pypi.douban.com/simple/


3）安装superset ：依赖较多，可能较慢

bin/pip install apache-superset -i https://pypi.douban.com/simple/


4）编写superset配置文件，主要更改元数据配置信息

a)配置python环境变量：
$ vim ~/.bash_profile

PYTHONPATH="/opt/miniconda3/conf:{$PYTHONPATH}"
export PYTHONPATH


b)编写配置文件 
$ vim /opt/miniconda/conf/superset_config.py

\#---------------------------------------------------------
\# Superset specific config
\#---------------------------------------------------------
ROW_LIMIT = 5000

SUPERSET_WEBSERVER_PORT = 8088
\#---------------------------------------------------------

\#---------------------------------------------------------
\# Flask App Builder configuration
\#---------------------------------------------------------
\# Your App secret key
SECRET_KEY = '\2\1thisismyscretkey\1\2\e\y\y\h'

\# The SQLAlchemy connection string to your database backend
\# This connection defines the path to the database that stores your
\# superset metadata (slices, connections, tables, dashboards, ...).
\# Note that the connection information to connect to the datasources
\# you want to explore are managed directly in the web UI

\# 注释掉默认数据库 ，同时配置mysql元数据库地址
\# SQLALCHEMY_DATABASE_URI = 'sqlite:////path/to/superset.db' 
SQLALCHEMY_DATABASE_URI = 'mysql://superset:mysql@chavin.king/superset'

\# Flask-WTF flag for CSRF
WTF_CSRF_ENABLED = True
\# Add endpoints that need to be exempt from CSRF protection
WTF_CSRF_EXEMPT_LIST = []
\# A CSRF token that expires in 1 year
WTF_CSRF_TIME_LIMIT = 60 * 60 * 24 * 365

\# Set this API key to enable Mapbox visualizations
MAPBOX_API_KEY = ''


5）初始化数据库 
a）首先对接mysql数据源：
bin/conda install mysqlclient

b）执行superset初始化 
bin/superset db upgrade

附加 - 此处可能报错如下：
Traceback (most recent call last):
File "bin/superset", line 18, in <module>
from superset.cli import superset
File "/opt/miniconda3/lib/python3.7/site-packages/superset/__init__.py", line 21, in <module>
from superset.app import create_app
File "/opt/miniconda3/lib/python3.7/site-packages/superset/app.py", line 43, in <module>
from superset.security import SupersetSecurityManager
File "/opt/miniconda3/lib/python3.7/site-packages/superset/security/__init__.py", line 17, in <module>
from superset.security.manager import SupersetSecurityManager # noqa: F401
File "/opt/miniconda3/lib/python3.7/site-packages/superset/security/manager.py", line 25, in <module>
from flask_appbuilder.security.sqla.manager import SecurityManager
File "/opt/miniconda3/lib/python3.7/site-packages/flask_appbuilder/security/sqla/manager.py", line 19, in <module>
from ..manager import BaseSecurityManager
File "/opt/miniconda3/lib/python3.7/site-packages/flask_appbuilder/security/manager.py", line 17, in <module>
from .registerviews import (
File "/opt/miniconda3/lib/python3.7/site-packages/flask_appbuilder/security/registerviews.py", line 10, in <module>
from .forms import LoginForm_oid, RegisterUserDBForm, RegisterUserOIDForm
File "/opt/miniconda3/lib/python3.7/site-packages/flask_appbuilder/security/forms.py", line 54, in <module>
class RegisterUserDBForm(DynamicForm):
File "/opt/miniconda3/lib/python3.7/site-packages/flask_appbuilder/security/forms.py", line 72, in RegisterUserDBForm
validators=[DataRequired(), Email()],
File "/opt/miniconda3/lib/python3.7/site-packages/wtforms/validators.py", line 332, in __init__
raise Exception("Install 'email_validator' for email validation support.")
Exception: Install 'email_validator' for email validation support.

解决办法 - bin/pip install email_validator

错误2 - 初始化数据库失败 
Loaded your LOCAL configuration at [/opt/miniconda3/conf/superset_config.py]
Failed to create app
Traceback (most recent call last):
File "/opt/miniconda3/lib/python3.7/site-packages/superset/app.py", line 59, in create_app
app_initializer.init_app()
File "/opt/miniconda3/lib/python3.7/site-packages/superset/app.py", line 464, in init_app
self.setup_db()
File "/opt/miniconda3/lib/python3.7/site-packages/superset/app.py", line 592, in setup_db
pessimistic_connection_handling(db.engine)
File "/opt/miniconda3/lib/python3.7/site-packages/flask_sqlalchemy/init.py", line 937, in engine
return self.get_engine()
File "/opt/miniconda3/lib/python3.7/site-packages/flask_sqlalchemy/init.py", line 956, in get_engine
return connector.get_engine()
File "/opt/miniconda3/lib/python3.7/site-packages/flask_sqlalchemy/init.py", line 561, in get_engine
self.engine = rv = self.sa.create_engine(sa_url, options)
File "/opt/miniconda3/lib/python3.7/site-packages/flask_sqlalchemy/init.py", line 966, in create_engine
return sqlalchemy.create_engine(sa_url, engine_opts)
File "/opt/miniconda3/lib/python3.7/site-packages/sqlalchemy/engine/init.py", line 479, in create_engine
return strategy.create(*args, kwargs)
File "/opt/miniconda3/lib/python3.7/site-packages/sqlalchemy/engine/strategies.py", line 87, in create
dbapi = dialect_cls.dbapi(dbapi_args)
File "/opt/miniconda3/lib/python3.7/site-packages/sqlalchemy/dialects/mysql/mysqldb.py", line 118, in dbapi
return import("MySQLdb")
ModuleNotFoundError: No module named 'MySQLdb'
Traceback (most recent call last):
File "bin/superset", line 21, in <module>
superset()
File "/opt/miniconda3/lib/python3.7/site-packages/click/core.py", line 829, in call
return self.main(*args, kwargs)
File "/opt/miniconda3/lib/python3.7/site-packages/flask/cli.py", line 586, in main
return super(FlaskGroup, self).main(args, kwargs)
File "/opt/miniconda3/lib/python3.7/site-packages/click/core.py", line 782, in main
rv = self.invoke(ctx)
File "/opt/miniconda3/lib/python3.7/site-packages/click/core.py", line 1256, in invoke
Command.invoke(self, ctx)
File "/opt/miniconda3/lib/python3.7/site-packages/click/core.py", line 1066, in invoke
return ctx.invoke(self.callback, ctx.params)
File "/opt/miniconda3/lib/python3.7/site-packages/click/core.py", line 610, in invoke
return callback(args, kwargs)
File "/opt/miniconda3/lib/python3.7/site-packages/click/decorators.py", line 21, in new_func
return f(get_current_context(), *args, kwargs)
File "/opt/miniconda3/lib/python3.7/site-packages/flask/cli.py", line 425, in decorator
with ctx.ensure_object(ScriptInfo).load_app().app_context():
File "/opt/miniconda3/lib/python3.7/site-packages/flask/cli.py", line 381, in load_app
app = call_factory(self, self.create_app)
File "/opt/miniconda3/lib/python3.7/site-packages/flask/cli.py", line 119, in call_factory
return app_factory()
File "/opt/miniconda3/lib/python3.7/site-packages/superset/app.py", line 66, in create_app
raise ex
File "/opt/miniconda3/lib/python3.7/site-packages/superset/app.py", line 59, in create_app
app_initializer.init_app()
File "/opt/miniconda3/lib/python3.7/site-packages/superset/app.py", line 464, in init_app
self.setup_db()
File "/opt/miniconda3/lib/python3.7/site-packages/superset/app.py", line 592, in setup_db
pessimistic_connection_handling(db.engine)
File "/opt/miniconda3/lib/python3.7/site-packages/flask_sqlalchemy/init.py", line 937, in engine
return self.get_engine()
File "/opt/miniconda3/lib/python3.7/site-packages/flask_sqlalchemy/init.py", line 956, in get_engine
return connector.get_engine()
File "/opt/miniconda3/lib/python3.7/site-packages/flask_sqlalchemy/init.py", line 561, in get_engine
self.engine = rv = self.sa.create_engine(sa_url, options)
File "/opt/miniconda3/lib/python3.7/site-packages/flask_sqlalchemy/init.py", line 966, in create_engine
return sqlalchemy.create_engine(sa_url, **engine_opts)
File "/opt/miniconda3/lib/python3.7/site-packages/sqlalchemy/engine/init.py", line 479, in create_engine
return strategy.create(*args, kwargs)
File "/opt/miniconda3/lib/python3.7/site-packages/sqlalchemy/engine/strategies.py", line 87, in create
dbapi = dialect_cls.dbapi(dbapi_args)
File "/opt/miniconda3/lib/python3.7/site-packages/sqlalchemy/dialects/mysql/mysqldb.py", line 118, in dbapi
return import__("MySQLdb")
ModuleNotFoundError: No module named 'MySQLdb'

解决办法 - bin/pip install mysqlclient


6）创建管理员用户 
export FLASK_APP=superset
bin/superset fab create-admin

7）初始化superset 
bin/superset init

8）启动superset

a）安装gunicorn 
bin/pip install gunicorn -i https://pypi.douban.com/simple/

b）启动superset ，确保当前环境为superset 
bin/gunicorn --workers 2 --timeout 120 --bind chavin.king:8088 "superset.app:create_app()"

--workers : 指定进程个数 
--timeout ：worker进程超时时间，超时会自动重启 
--bind : 绑定本机地址 ，即superset访问地址


或者

bin/superset run -h 0.0.0.0 -p 8088 --with-threads --reload --debugger

c）停止superset 
ps -ef|awk '/gunicorn/ %% !/awk/{print $2}' | xargs kill -9

d）退出superset 
conda deactivate

e）登录superset 
http://chavin.king:8088
chavin/superset