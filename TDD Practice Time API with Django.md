# TDD练习：开发基于Django的时间API
[原文链接](https://brntn.me/blog/django-tdd-practice-time-api/)

通过使用[Django](https://www.djangoproject.com/)搭建一个小型API联系TDD技巧。

把这当成一个[Kata](http://codekata.com/)。第一次**完全**按照下面的步骤进行，然后第二次全部都自己做。

当你自己做时可以删除原来的代码重建项目，或者使用其他语言或web框架。

## 所以，什么是TDD？


测试驱动开发（Test-Driven Development）是通过编写测试来指导软件开发的构建软件的技术。

———— [Martin Fowler](https://martinfowler.com/bliki/TestDrivenDevelopment.html)

这篇文章中我们遵循TDD的红色->绿色->重构联系。这意味着我们为应用程序编写的每一行代码都将使失败的测试用例通过。在每个周期中，我们将:

1. 编写一个测试
2. 编写代码让测试通过
3. 重构以确保代码整洁

当我们进行重构时，我们可以考虑对生产代码和测试用例进行更改。在测试通过之后，你并不总是需要重构，但是你应该总是花一些时间来回顾你所做的工作。

在实践中，我们将努力严格遵守[Bob叔叔](http://www.butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd)的TDD三条规则:

1. 除非是为了让失败的单元测试通过，否则不允许编写任何产品代码。
2. 不允许编写任何超过足以失败的单元测试；编译失败就是失败。
3. 不允许编写超过通过一个失败单元测试所需的任何生产代码。

## 设置环境

首先检查你的Python3版本：
```shell
> python3 --version
```

如果返回值低于Python 3.6，那么请转到[Python网站](https://python.org/)安装最新版本。

为项目创建一个目录，你可以随意起名，我这里起名叫`time-api`：
```shell
> mkdir time-api
> cd time-api
```
创建一个venv用来隔离其他项目：
```shell
> python3 -m venv venv
```

这会创建一个名为`venv`的新目录，其中包含我们的Python虚拟环境。要激活环境，你需要激活脚本：
```shell
> source venv/bin/activate
```

当激活时，你应该注意到shell上有一个`(venv)`前缀。你可以在任何时候通过执行`deactivate`或关闭终端窗口来退出虚拟环境。

现在我们安装Django到venv：
```shell
(venv)> python -m pip install django
```

最后启动Django项目。最后的`.`告诉`django-admin`在当前目录创建项目：
```shell
(venv)> django-admin startproject time_api .
```

## 用户故事

所有伟大的项目都起源于一个用户故事。它能为我们的客户提供某种价值。

**作为用户我想接收当前的UTC时间，用来确认我的表走时是否准确**

验收标准：
*  /api/time应该返回一个带有current_time键的json响应
* 如果成功，状态码应该是200OK
* 所有的时间应该是ISO 8601格式

## Kata-招式

让我们从这个情景的最小部分开始。

URL是`/api/time/`，我能写一个测试确保收到一个200OK的响应。

Django测试客户端为我们处理了很多繁重的工作——我们使用`self.client.get()`向所需的URL发起HTTP请求，然后用`response.status_code`确认响应的状态码。

首先在`time_api`目录中创建一个新的`tests.py`文件，其中包含下面的测试用例。Django的测试执行器将自动发现以`test`开头的文件并执行测试。

```python
# time_api/tests.py
from django.test import TestCase

class TimeApiTestCase(TestCase):

    def test_time_url_is_status_okay(self):
        response = self.client.get('/api/time/')
        self.assertEqual(200, response.status_code)
```
运行这个测试，确保测试失败。你应该收到一个404的报错而不是200。
```shell
> python manage.py test
Found 1 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
F
======================================================================
FAIL: test_time_url_is_status_okay (time_api.tests.TimeApiTestCase)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/brenton/Dev/time-api/time_api/tests.py", line 7, in test_time_url_is_status_okay
    self.assertEqual(200, response.status_code)
AssertionError: 200 != 404

----------------------------------------------------------------------
Ran 1 test in 0.003s

FAILED (failures=1)
Destroying test database for alias 'default'...
```
Django成功处理了请求并返回了一个默认404模板。

虽然这是由Python的`unittest`框架运行的，但这就是我们所说的集成测试。当我们调用`Client.get()`时，完整的Django请求/响应生命周期被运行。

现在我们向`urls.py`添加写代码。记住我们想写最少的代码以通过测试。

我们添加一个路径（#1），创建一个视图（#2），返回一个Http响应（#3）。

```python
# time_api/tests.py
from django.test import TestCase

class TimeApiTestCase(TestCase):

    def test_time_url_is_status_okay(self):
        response = self.client.get('/api/time/')
        self.assertEqual(200, response.status_code)
# time_api/urls.py
from django.contrib import admin
from django.urls import path

from django.http import HttpResponse  # 3

def time_api(request):  # 2
    return HttpResponse()

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/time/', time_api),  # 1
]
```

重新执行测试，你应该可以通过。
```python
> python manage.py test
Found 1 test(s).
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
.
----------------------------------------------------------------------
Ran 1 test in 0.002s

OK
Destroying test database for alias 'default'...
```

我们刚刚迈出了**测试驱动**的第一步，恭喜👏

现在我们要考虑是继续开发，还是重构部分代码。我感觉应该继续，你怎么说？

我们要开发的第二个功能是API应该返回一个JSON响应。你可能已经在思考如何开发代码实现它了，但是如何实现测试呢？

Django测试客户端返回的响应对象允许我们以字典的形式访问HTTP响应头。感谢这个功能能，我们可以写一个测试以确定Content-Type是application/json。
```python
# time_api/tests.py
from django.test import TestCase

class TimeApiTestCase(TestCase):

    def test_time_url_is_status_okay(self):
        response = self.client.get('/api/time/')
        self.assertEqual(200, response.status_code)

    def test_time_api_should_return_json(self):
        response = self.client.get('/api/time/')
        self.assertEqual('application/json', response['Content-Type'])
```

当我们运行测试时，我们会看到显示`text/html`不等于`application/json`，它们当然不相等。

最简单让该测试通过的方法是将我们的`HttpResponse`修改为`JsonResponse`（#1，#2）。Django内置的`JsonResponse`可以为我们处理序列化数据并让生活更简单美好。
```python
# time_api/urls.py
from django.contrib import admin
from django.urls import path

from django.http import HttpResponse, JsonResponse  # 2

def time_api(request):
    return JsonResponse({})  # 1

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/time/', time_api),
]
```
我们应该重构么？
是的！让我们移除无用的import `HttpResponse`，重新排序下import让`isort`井井有条：
```python
# time_api/tests.py
from django.test import TestCase

class TimeApiTestCase(TestCase):

    def test_time_url_is_status_okay(self):
        response = self.client.get('/api/time/')
        self.assertEqual(200, response.status_code)

    def test_time_api_should_return_json(self):
        response = self.client.get('/api/time/')
        self.assertEqual('application/json', response['Content-Type'])
```

```python
# time_api/urls.py
from django.contrib import admin
from django.http import JsonResponse
from django.urls import path

def time_api(request):
    return JsonResponse({})

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/time/', time_api),
]
```
很棒，我们通过了两个侧式，然后我们刚刚完成了第一次重构，提高了代码的可读性。

我们继续实现下一个特性：在结果中返回键为`current_time`的值。

测试依旧非常简单。我们对响应调用`.json()`将它转换为字典，用`in`操作符检查我们期待的键。
```python
# time_api/tests.py
from django.test import TestCase

class TimeApiTestCase(TestCase):

    def test_time_url_is_status_okay(self):
        response = self.client.get('/api/time/')
        self.assertEqual(200, response.status_code)

    def test_time_api_should_return_json(self):
        response = self.client.get('/api/time/')
        self.assertEqual('application/json', response['Content-Type'])

    def test_time_api_should_include_current_time_key(self):
        response = self.client.get('/api/time/')
        self.assertTrue('current_time' in response.json())
```
一切正常，我们的测试又因为正当的理由失败了，`current_time`不在我们返回的JSON中。

我们更新view来返回期待的键，现在返回一个空字符串就够了：
```python
# time_api/urls.py
from django.contrib import admin
from django.http import JsonResponse
from django.urls import path

def time_api(request):
    return JsonResponse({
        'current_time': ''  # 1
    })

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/time/', time_api),
]
```

重新执行测试确保它通过。

再次考虑重构这些代码。现在我觉得还好......接下来该轮到你了，想想你是否有兴趣改变什么。

现在是时候返回一个真实的日期了。我们需要它符合ISO 8601格式，可以通过在测试中解析它并确定得到一个`datetime`对象。
```python
# time_api/tests.py
from django.test import TestCase

class TimeApiTestCase(TestCase):

    def test_time_url_is_status_okay(self):
        response = self.client.get('/api/time/')
        self.assertEqual(200, response.status_code)

    def test_time_api_should_return_json(self):
        response = self.client.get('/api/time/')
        self.assertEqual('application/json', response['Content-Type'])

    def test_time_api_should_include_current_time_key(self):
        response = self.client.get('/api/time/')
        self.assertTrue('current_time' in response.json())

    def test_time_api_should_return_valid_iso8601_format(self):
        response = self.client.get('/api/time/')
        current_time = response.json()['current_time']
        dt = datetime.strptime(current_time, '%Y-%m-%dT%H:%M:%SZ')
        self.assertTrue(isinstance(dt, datetime))
```
当我们运行上述测试时，测试不会 *失败* 而是产生一个 *错误* 。

你可以花些时间把我们的测试调用`strptime`放入一个异常处理中，但由于我们正在接收的错误和日期格式有关，现在先放着它也是可以的。

考虑一下我们可以编写的最简单的代码。我们是否需要返回当前时间才能通过这个测试？不，不需要。

让我们更新下view返回一个有效的日期，在这个例子中，只是返回一个有效格式的硬编码日期：
```python
# time_api/urls.py
from django.contrib import admin
from django.http import JsonResponse
from django.urls import path

def time_api(request):
    return JsonResponse({
        'current_time': '2007-10-10T08:00:00Z'
    })

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/time/', time_api),
]
```

我们所做的就是写一个我们需要的最简单的代码。我们的测试很好地通过了，因为除了返回一个有效的`datetime`外，我们没测试任何东西。

这时我们可以重构以 **概括化（generalise）** 解决方案。让我们添加一个新的测试，迫使我们**概括化**解决方案。

通过使用Python3内置的mocking库，我们提前知道调用Django的`timezone.now()`返回的日期。

我们要做的是“给响应打个补丁”，像这样：
```python
with patch('django.utils.timezone.now') as mock_tz_now:
    expected_datetime = datetime(2018, 1, 1, 10, 10, tzinfo=timezone.utc)
    mock_tz_now.return_value = expected_datetime
```

上述代码让`django.utils.timezone.now`的返回值替换为2018年1月1日十点十分。

让我们添加该测试来概括化解决方案。
```python
# time_api/tests.py
from django.test import TestCase

class TimeApiTestCase(TestCase):

    def test_time_url_is_status_okay(self):
        response = self.client.get('/api/time/')
        self.assertEqual(200, response.status_code)

    def test_time_api_should_return_json(self):
        response = self.client.get('/api/time/')
        self.assertEqual('application/json', response['Content-Type'])

    def test_time_api_should_include_current_time_key(self):
        response = self.client.get('/api/time/')
        self.assertTrue('current_time' in response.json())

    def test_time_api_should_return_valid_iso8601_format(self):
        response = self.client.get('/api/time/')
        current_time = response.json()['current_time']
        dt = datetime.strptime(current_time, '%Y-%m-%dT%H:%M:%SZ')
        self.assertTrue(isinstance(dt, datetime))

    def test_time_api_should_return_current_utc_time(self):
        with patch('django.utils.timezone.now') as mock_tz_now:
            expected_datetime = datetime(2018, 1, 1, 10, 10, tzinfo=timezone.utc)
            mock_tz_now.return_value = expected_datetime

            response = self.client.get('/api/time/')
            current_time = response.json()['current_time']
            parsed_time = datetime.strptime(current_time, '%Y-%m-%dT%H:%M:%SZ')

            self.assertEqual(parsed_time, expected_datetime)
```

运行测试，肯定会失败。不管现在是什么时间，肯定不等于之前设定的时间。

现在我们改用`timezone.now()`（#1）来返回当前时间然后`strftime()`（#2）成正确的格式。

```python
# time_api/urls.py
from django.contrib import admin
from django.http import JsonResponse
from django.urls import path
from django.utils import timezone

def time_api(request):
    return JsonResponse({
        'current_time': timezone.now().strftime('%Y-%m-%dT%H:%M:%SZ')  # 2
    })

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/time/', time_api),
]
```
运行测试并通过。至此我们完成了该用户故事的需求，干得好！

现在考虑下重构。有几件事我觉得可以做得更好。

首先我们可以把时间格式移到settings（#1）里，然后更新view使用settings中的配置而不是硬编码（#2，#3）。

```python
# time_api/settings.py

DATETIME_FORMAT = '%Y-%m-%dT%H:%M:%SZ'  # 1
# time_api/urls.py
from django.conf import settings
from django.contrib import admin
from django.http import JsonResponse
from django.urls import path
from django.utils import timezone

def time_api(request):
    return JsonResponse({
        'current_time': timezone.now().strftime(settings.DATETIME_FORMAT)  # 2
    })

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/time/', time_api),
]
```

我们运行测试确保没搞坏任何东西。一切正常，让我们继续。

现在，我们应该单独创建一个view文件，而不是把我们的视图写在`urls.py`中。

首先创建一个叫times的app：
```shell
(venv)> python manage.py startapp times
```
然后我们迁移视图代码至`times/views.py`然后更新`time_api_urls.py`指向新的view。这可以帮助我们清理`urls.py`中的import，大部分import不再需要。
```python
# times/views.py
from django.conf import settings
from django.http import JsonResponse
from django.utils import timezone


def time_api(request):
    return JsonResponse({
        'current_time': timezone.now().strftime(settings.DATETIME_FORMAT)
    })
```
```python
# time_api/urls.py
from django.contrib import admin
from django.urls import path

from times.views import time_api

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/time/', time_api),
]
```
现在我们重新运行下测试确保一路绿灯。

最后一步，我们将测试移到`times/tests.py`中，这样它们就更接近所测试的代码。
```python
# times/tests.py
from datetime import datetime
from unittest.mock import patch

from django.test import TestCase

class TimeApiTestCase(TestCase):

    def test_time_url_is_status_okay(self):
        response = self.client.get('/api/time/')
        self.assertEqual(200, response.status_code)

    def test_time_api_should_return_json(self):
        response = self.client.get('/api/time/')
        self.assertEqual('application/json', response['Content-Type'])


    def test_time_api_should_include_current_time_key(self):
        response = self.client.get('/api/time/')
        self.assertTrue('current_time' in response.json())

    def test_time_api_should_return_valid_iso8601_format(self):
        response = self.client.get('/api/time/')
        current_time = response.json()['current_time']
        dt = datetime.strptime(current_time, '%Y-%m-%dT%H:%M:%SZ')
        self.assertTrue(isinstance(dt, datetime))

    def test_time_api_should_return_current_utc_time(self):
        with patch('django.utils.timezone.now') as mock_tz_now:
            expected_datetime = datetime(2018, 1, 1, 10, 10, tzinfo=timezone.utc)
            mock_tz_now.return_value = expected_datetime

            response = self.client.get('/api/time/')
            current_time = response.json()['current_time']
            parsed_time = datetime.strptime(current_time, '%Y-%m-%dT%H:%M:%SZ')

            self.assertEqual(parsed_time, expected_datetime)
```

酷。最后运行一次测试确保测试还是会正确覆盖。所有测试都通过了？喝杯咖啡庆祝一下☕️！

## 该你了

这就是我的做法。现在该你了：通过先写测试然后看着它们失败来开发。

考虑一下如何使用类型提示来帮助文档化这些接口。或者如果设置中的时区不是UTC会发生什么？有什么方法可以验证吗？

你可以使用 *任何语言或框架* 来完成该招式，希望你的反馈。