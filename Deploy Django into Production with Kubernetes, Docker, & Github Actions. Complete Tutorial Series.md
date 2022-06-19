# 使用Kubernetes，Docker，GitHub Actions部署Django生产环境的完整指南
[原视频链接](https://www.youtube.com/watch?v=NAOsLaB6Lfc&t=7047s)

## Kubernetes和Django
空

## 要求和建议

* Python3.9+
* Django3.2+
* Docker
* Kubectl

## 在macOS上安装Kubectl
空

## 在Windows上安装Kubectl
空
## 创建Python虚拟环境

```shell
python3.9 -m venv venv
```

## 安装依赖，创建Django项目
Django2.2LTS截止于2022.Q1

Django3.2LTS截止于2024.Q1

Django4.2LTS截止于2025.Q3+ 

使用pycharm或`django-admin`创建项目

创建`requirements.txt`：
```txt
django
gunicorn
requests
django-dotenv
psycopg2-binary
django-storages
boto3
```

## 使用dotenv创建环境变量文件

`dotenv`用来分离敏感的配置信息，在项目根目录下创建`.env`文件，并将其添加到`.gitgnore`：
```ini
DEBUG=1
REGION=texas
DJANGO_SUPERUSER_USERNAME=admin
DJANGO_SUPERUSER_PASSWORD=mydjangopw
DJANGO_SUPERUSER_EMAIL=test@gmail.com
DJANGO_SECRET_KEY=django-insecure-x84e_fd!g%ow%g1eupgp9bwkwcjhh&*@6oh993q#-mc@z!f@)^

POSTGRES_READY=0
POSTGRES_DB=dockerdc
POSTGRES_PASSWORD=mysecretpassword
POSTGRES_USER=myuser
POSTGRES_HOST=postgres_db
POSTGRES_PORT=5433

REDIS_HOST=redis_db
REDIS_PORT=6380
```

## 配置dotenv读取环境变量文件
修改`wsgi.py`，在`import os`后添加如下代码用来读取`.env`：
```python
import pathlib

import dotenv

from django.core.wsgi import get_wsgi_application

CURRENT_DIR = pathlib.Path(__file__).parent
BASE_DIR = CURRENT_DIR.parent
ENV_FILE_PATH = BASE_DIR / '.env'

dotenv.read_dotenv(str(ENV_FILE_PATH))
```

修改`manage.py`读取`.env`：
```python
import dotenv
...
...
if __name__ == '__main__':
    dotenv.read_dotenv()
    main()
```

现在打开控制台输入`python manage.py shell`进入交互环境确认：
```python
>>> import os
>>> print(os.environ.get("REGION"))
texas
```
成功读取到环境变量

## 将环境变量应用到Django的settings文件

修改如下部分的代码，将密钥放置在`.env`中：
```python
SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY")

DEBUG = str(os.environ.get("DEBUG"))
```

在默认数据库后添加如下代码，以分离开发和生产数据库：
```python
DB_USERNAME = os.environ.get("POSTGRES_USER")
DB_PASSWORD = os.environ.get("POSTGRES_PASSWORD")
DB_DATABASE = os.environ.get("POSTGRES_DB")
DB_HOST = os.environ.get("POSTGRES_HOST")
DB_PORT = os.environ.get("POSTGRES_PORT")
DB_IS_AVIAL = all([
    DB_USERNAME,
    DB_PASSWORD,
    DB_DATABASE,
    DB_HOST,
    DB_PORT
])
POSTGRES_READY = str(os.environ.get("POSTGRES_READY")) == "1"

if DB_IS_AVIAL and POSTGRES_READY:
    DATABASES = {
        "default": {
            "ENGINE": "django.db.backends.postgresql",
            "NAME": DB_DATABASE,
            "USER": DB_USERNAME,
            "PASSWORD": DB_PASSWORD,
            "HOST": DB_HOST,
            "PORT": DB_PORT,
        }
    }
```
完成配置后，如果`.env`的`POSTGRES_READY`设置为1时会切换到生产数据库。

## Docker，Dockerfile和dockerignore

在根目录创建`dockerignore`，复制`.gitgnore`内容到`dockerignore`。

在根目录创建`Dockerfile`：
```docker
FROM python:3.9.13-slim

COPY . /app
WORKDIR /app

RUN python3 -m venv /opt/venv/

RUN /opt/venv/bin/pip install pip --upgrade && \
    /opt/venv/bin/pip install -r requirements.txt && \
    chmod +x entrypoint.sh

CMD ["/app/entrypoint.sh"]
```

在根目录创建入口脚本`entrypoint.sh`：
```bash
#!/bin/bash
APP_PORT=${PORT:-8000}
cd /app/
/opt/venv/bin/gunicorn --worker-tmp-dir /dev/shm django_k8s.wsgi:application --bind "0.0.0.0:${APP_PORT}"
```


## 创建迁移脚本

在根目录创建迁移脚本`migrate.sh`：
```bash
#!/bin/bash

DJANGO_SUPERUSER_EMAIL=${DJANGO_SUPERUSER_EMAIL:"test@gmail.com"}
cd /app/

/opt/venv/bin/python manage.py migrate --noinput
/opt/venv/bin/python manage.py createsuperuser --email $DJANGO_SUPERUSER_EMAIL --noinput || true

```

## Docker Compose 1

创建`docker-compose.yaml`：
```yaml
version: "3.9"
services:
  web:
    depends_on:
      - postgres_db
    build:
      context: ../django_k8s # 你的目录
      dockerfile: Dockerfile
    image: django_k8s:1.0.0
    environment:
      - PORT=8020
    env_file:
      - .env
    ports:
      - "8001:8020"
    command: sh -c "chmod +x /app/migrate.sh && sh /app/migrate.sh && /app/entrypoint.sh"
  postgres_db:
    image: postgres
    command:
      - -p 5433
    env_file:
      - .env
    expose:
      - 5433
    ports:
      - "5433:5433"
    volumes:
      - postgres_data:/var/lib/postgresql/data/
  redis_db:
    image: redis
    restart: always
    expose:
      - 6380
    ports:
      - "6380:6380"
    volumes:
      - redis_data:/data
    entrypoint: redis-server --appendonly yes --port 6380



volumes:
  postgres_data:
  redis_data:
```

执行命令`docekr compose up --build`进行构建测试。

## Docker Compose 2

在Django shell中检查数据库设置：
```python
>>> from django.conf import settings
>>> print(settings.DB_IS_AVAIL)
```

修改`.env`中的POSTGRES_HOST=0.0.0.0可以调整数据库绑定地址

## 在DigitalOcean上部署Kubernetes
空

## 通过kubectl连接Kubernetes
空

## 在Kubernetes部署你的第一个容器