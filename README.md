# djcelery demo
基于django 2.0.4 的 djcelery 配置示例

[TOC]

## 所需环境

python 3.5.2

rabbitmq

### 安装所需的包

`pip install -r requirements.txt`

## QuickStart

### 创建Django项目

创建一个名为proj的Django项目

`django-admin startproject proj`

### 创建Django App

创建一个用于演示的django app，这里名为demo

`django-admin startapp demo`

在创建的app中，增加tasks.py文件，用于编写celery任务

### 基础配置项目

修改proj/settings.py配置文件，增加celery相关配置。

#### 增加djcelery app

修改settings.py中INSTALLED_APPS，增加djcelery及app

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'djcelery',
    'demo'
]
```

#### celery相关的参数配置

如果仅仅需求使用celery异步执行任务的话，以下最基础的配置就可以满足需求

```python
# 导入tasks文件，因为我们使用autodiscover_tasks
# 会自动导入每个app下的tasks.py，所以这个配置不是很必要
# 如果需要导入其他非tasks.py的模块，则需要再此配置需要导入的模块
# CELERY_IMPORTS = ('demo.tasks', )
# 配置 celery broker
CELERY_BROKER_URL = 'amqp://user:password@127.0.0.1:5672//'
# 配置 celery backend 用Redis会比较好
# 因为手上没有redis服务器，所以演示时用RabbitMQ替代
CELERY_RESULT_BACKEND = 'amqp://user:password@127.0.0.1:5672//'
```

### 创建Celery实例

在proj目录下，编辑celery.py文件，用于创建celery实例

```python
from celery import Celery
from django.conf import settings

# 创建celery应用
celery_app = Celery('proj', broker=settings.CELERY_BROKER_URL)
# 从配置文件中加载除celery外的其他配置
celery_app.config_from_object('django.conf:settings')
# 自动检索每个app下的tasks.py
celery_app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)
```

### 编写异步任务

在之前创建的demo/tasks.py中，编写一个用于演示的异步任务。

注意每个异步任务之前都需要使用@celery_app.task装饰器。

celery_app实际是之前在proj/celery.py中创建的celery的实例，如果你的实例名称不一样，做对应的修改即可。

```Python
import logging
from proj.celery import celery_app

@celery_app.task
def async_task():
    logging.info('run async_task')
```

### 调用异步任务

在demo/views.py中定义一个页面，只用来调用异步任务。

```python
from django.http import HttpResponse
from demo.tasks import async_demo_task

# Create your views here.
def demo_task(request):
	# delay表示将任务交给celery执行
    async_demo_task.delay()
    return HttpResponse('任务已经运行')
```

在proj/urls.py中注册对应的url。

```python
from django.contrib import admin
from django.urls import path
from demo.views import demo_task

urlpatterns = [
    path('admin/', admin.site.urls),
    path('async_demo_task', demo_task),
]
```

### 启动Celery Worker

使用命令启动worker：

`manage.py celery -A proj worker -l info`

对参数做个简单的说明：

-A proj是指项目目录下的celery实例。演示项目名为proj，所以-A的值是proj。如果项目名是其他名字，将proj换成项目对应的名字。

-l info 是指日志记录的级别，这里记录的是info级别的日志。

如果配置没有问题，能成功连接broker，则会有类似以下的日志：

```powershell
 -------------- celery@Matrix.local v3.1.26.post2 (Cipater)
---- **** ----- 
--- * ***  * -- Darwin-17.5.0-x86_64-i386-64bit
-- * - **** --- 
- ** ---------- [config]
- ** ---------- .> app:         proj:0x108ab1eb8
- ** ---------- .> transport:   amqp://user:**@127.0.0.1:5672//
- ** ---------- .> results:     amqp://
- *** --- * --- .> concurrency: 4 (prefork)
-- ******* ---- 
--- ***** ----- [queues]
 -------------- .> celery           exchange=celery(direct) key=celery
                

[tasks]
  . demo.tasks.async_demo_task

[2018-04-24 08:24:47,656: INFO/MainProcess] Connected to amqp://user:**@127.0.0.1:5672//
```

需要注意的是日志中的tasks部分，可以看到已经自动识别到了demo.tasks.async_demo_task这个用于演示的任务。

如果没有识别到，检查下celery实例是否调用autodiscover_tasks方法，或配置文件的CELERY_IMPORTS是否配置正确。

### 调用异步任务

在demo/views.py中定义一个页面，只用来调用异步任务。

```python
from django.http import HttpResponse
from demo.tasks import async_demo_task

# Create your views here.
def demo_task(request):
	# delay表示将任务交给celery执行
    async_demo_task.delay()
    return HttpResponse('任务已经运行')
```

在proj/urls.py中注册对应的url。

```python
from django.contrib import admin
from django.urls import path
from demo.views import demo_task

urlpatterns = [
    path('admin/', admin.site.urls),
    path('async_demo_task', demo_task),
]
```

最后，启动django，访问url http://127.0.0.1:8000/async_demo_task 调用异步任务。

在worker的日志中，可以看到类似的执行结果，即说明任务已经由celery异步执行。

```powershell
[2018-04-24 09:25:52,677: INFO/MainProcess] Received task: demo.tasks.async_demo_task[1105c262-9371-4791-abd2-6f78d654b391]
[2018-04-24 09:25:52,681: INFO/Worker-4] run async_task
[2018-04-24 09:25:52,899: INFO/MainProcess] Task demo.tasks.async_demo_task[1105c262-9371-4791-abd2-6f78d654b391] succeeded in 0.21868160199665s: None
```

