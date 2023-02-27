# 使用Kubernetes，Docker，GitHub Actions部署Django生产环境的完整指南
[原视频链接](https://www.youtube.com/watch?v=NAOsLaB6Lfc&t=7047s)

## Kubernetes和Django介绍
跳过

## 要求和建议

* Python3.9+
* Django3.2+
* Docker
* Kubectl

## 在macOS上安装Kubectl
跳过

## 在Windows上安装Kubectl
跳过

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

修改`.env`中的`POSTGRES_HOST=0.0.0.0`可以调整数据库绑定地址

## 在DigitalOcean上部署Kubernetes
跳过

## 通过kubectl连接Kubernetes
跳过

## 在Kubernetes部署你的第一个容器

在根目录创建文件夹：/k8s/nginx/
在该文件夹中创建`deployment.yaml`文件
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: nginx-deployment
    template:
      metadata:
        labels:
          apps: nginx-deployment
      spec:
        containers:
          - name: nginx
            image: nginx:latest
            ports:
              - containerPort: 80

```

执行命令：`kubectl apply -f k8s/nginx/deployment.yaml`

## 使用负载均衡对外暴露你的部署

在/k8s/nginx/创建`service.yaml`文件：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  ports:
    - name: http
    protocol: TCP
    port: 80 # 外部可访问端口
    targetPort: 80 # 和deployment端口一致
  selector:
    app: nginx-deployment
```

执行命令：`kubectl apply -f k8s/nginx/service.yaml`

## 部署一个最小化的FastAPI APP

创建文件夹/k8s/apps/

创建文件/k8s/apps/iac-python.yaml：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iac-python-deployment
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: iac-python-deployment
    template:
      metadata:
        labels:
          apps: iac-python-deployment
      spec:
        containers:
          - name: iac-python
            image: codingforentrepreneurs/iac-python:latest # 可替换成你自己的镜像
            env:
              - name: PORT
                value: "8181"
            ports:
              - containerPort: 8181
---
apiVersion: v1
kind: Service
metadata:
  name: iac-python-service
spec:
  type: LoadBalancer
  ports:
    - name: http
    protocol: TCP
    port: 80 # 外部可访问端口
    targetPort: 8181 # 和deployment端口一致
  selector:
    app: iac-python-deployment
```
执行命令：`kubectl apply -f k8s/apps/iac-python.yaml`

## DigitalOcean容器仓库

跳过

## 构建&推送容器到DO容器仓库

跳过

## 管理PG数据库

创建文件`/web/.env.prod`用来存放生产配置数据，复制`.env`的内容到`.env.prod`，将其添加进ignore

修改生产env配置，匹配你的数据库设置

## 配置env的K8s-Secrets

执行命令：`kubectl create secret generic django-k8s-web-prod-env --from-env-file=web/.env.prod`

执行命令：`kubectl get secret`查看创建的secrets

执行命令：`kubectl get secret django-k8s-web-prod-env -o YAML`查看具体数据

## Django部署&服务

在`.env`文件中添加一行用来忽视数据库SSL需求：

`DB_IGNORE_SSL=true`

修改django项目settings文件，添加：
```python
DB_IGNORE_SSL=os.environ.get("DB_IGNORE_SSL") == "ture"

if DB_IS_AVAIL:
  DATABASES = {...}
  if not DB_IGNORE_SSL:
    DATABASES['default']['OPTIONS'] = {
      'sslmode': 'require'
    }
```

在`k8s/apps`下创建`django-k8s-web.yaml`文件：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-k8s-web-deployment
  labels:
    app: django-k8s-web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: django-k8s-web-deployment
  template:
    metadata:
      labels:
        app: django-k8s-web-deployment
    spec:
      containers:
      - name: django-k8s-web
        image: your-registry/django-k8s-web-deployment:latest
        ports:
        - containerPort: 80
```

现在我们需要关联secret和我们即将部署的容器

执行命令：`kubectl get serviceaccount default -o YAML`可查看默认serviceaccount

修改`django-k8s-web.yaml`，添加关联:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-k8s-web-deployment
  labels:
    app: django-k8s-web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: django-k8s-web-deployment
  template:
    metadata:
      labels:
        app: django-k8s-web-deployment
    spec:
      containers:
      - name: django-k8s-web
        image: your-registry/django-k8s-web-deployment:latest
        envFrom:
          - secretRef:
              name: django-k8s-web-prod-env
        env:
          - name: PORT
            value: "8001"
        ports:
        - containerPort: 8001
      imagePullSecrets:
        - name: cfe-k8s
---
apiVersion: v1
kind: Service
metadata:
  name: django-k8s-web-service
spec:
  type: LoadBalancer
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8001
  selector:
    app: django-8ks-web-deployment
```

执行命令：`kubectl apply -f k8s/apps/django-k8s-web.yaml`创建服务。

## 完整部署&bug修复

修改生产环境文件`.env.prod`，添加一行代码`ENV_ALLOWED_HOST=*`，并删除代码`DEBUG=1`

修改django的settings文件：
```python
ENV_ALLOWED_HOST = os.environ.get('ENV_ALLOWED_HOST')
ALLOWED_HOSTS = []
if ENV_ALLOWED_HOST:
  ALLOWED_HOSTS = [ENV_ALLOWED_HOST]
```

我们修改了配置文件，删除旧的secrets，重新执行命令创建新的secrets：`kubectl create secret generic django-k8s-web-prod-env --from-env-file=web/.env.prod`

因为我们修改了settings的代码，重新构建docker镜像。 

开启一个新的终端执行命令：`kubectl get pods -w`

重新执行，观察新终端变化：`kubectl apply -f k8s/apps/django-k8s-web.yaml`

## 编写部署指南

在`web/django_k8s/`目录下创建`deployment-guide.md`文件：
1. 测试Django
```
python manage.py test
```
2. 构建容器
```
docker build -f Dockerfile -t your-target-registry:label .
```
3. 推送镜像到仓库
```
docker push your-target-registry --all-tags
```
4. 更新k8s secrets 
```
kubectl delete secret django-k8s-web-prod-env
kubectl create secret generic django-k8s-web-prod-env --from-env-file=web/.env.prod
```

5. 更新部署

```
kubectl apply -f k8s/apps/django-k8s-web.yaml
```

6. 等待rollout完成

```
kubectl rollout status deployment/django-k8s-web-deployment
```

7. migrate数据库
```shell
export SINGLE_POD_NAME=$(kubectl get pod -l app=django-k8s-web-deployment -o jsonpath="{.items[0].metadata.name}")
```

```shell
kubectl exec -it $SINGLE_POD_NAME -- bash /app/migrate.sh
```

## 使用Github Actions自动测试Django
跳过