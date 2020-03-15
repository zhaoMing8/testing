#  pytest测试框架4-插件与hook函数
## 简介
pytest的自带功能很强大，通过添加插件可以扩展功能，pytest的代码结构适合定制和扩展插件，  
可以借助hook函数来实现。  
把fixture函数或者hook函数添加到conftest文件里，就已经创建了一个本地的conftest插件！
## pytest plugin加载的几种方式：
1.内置plugins：从代码内部的_pytest目录加载；  
2.外部插件（第三方插件）：通过setuptools entry points机制发现的第三方插件模块；  
推荐的第三方的pytest的插件：https://docs.pytest.org/en/latest/plugins.html  
3.conftest.py形式的本地插件：测试目录下的自动模块发现机制  
通过pytest --trace-config命令可以查看当前pytest中所有的plugin。  
在pytest中，所谓plugin其实就是能被pytest发现的一些带有pytest hook方法的文件或对象。
## What is a hook
要理解pytest hook，首先要知道什么是hook方法（钩子函数）。  
这里举一个简单的例子，比如说你写了一个框架类的程序，然后你希望这个框架可以“被代码注入”，即别人可以加入代码对你这个框架进行定制化，该如何做比较好？一种很常见的方式就是约定一个规则，框架初始化时会收集满足这个规则的所有代码（文件），然后把这些代码加入到框架中来，在执行时一并执行即可。所有这一规则下可以被框架收集到的方法就是hook方法。
## 编写自己的插件
插件可以改变pytest行为，可用的hook函数很多，详细的定义：  
http://doc.pytest.org/en/latest/_modules/_pytest/hookspec.html
1.pytest_addoption为例，基本每个pytest plugin都会有这个hook方法，它的作用是为pytest命令行添加自定义的参数。  
parser:用户命令行参数与ini文件值的解析器
```python
def pytest_addoption(parser):
    parser.addoption("--env",    ##注册一个命令行选项
                    default="test",
                    dest="env",
                    help="set test run env")
```
pytest_addoption: Hook function, 这里创建了一个argparser的group，通过addoption方法添加option，使得显示help信息时相关option显示在一个group下面，更加友好。  
命令行输入：
pytest --help 就可以看到  

2.修改pytest_collection_modifyitems  
能解决什么实际问题？  
测试case中 case名字为中文时，显示的时乱码！  
完成所有测试项的收集后，pytest调用的钩子
```python

def pytest_collection_modifyitems(items):
    """
    测试用例收集完成时，将收集到的item的name和nodeid的中文显示在控制台上
所有的测试用例收集完毕后调用, 可以再次过滤或者对它们重新排序
 items （收集的测试项目列表）
    """
    for item in items:
        item.name = item.name.encode("utf-8").decode("unicode_escape")
        item._nodeid = item.nodeid.encode("utf-8").decode("unicode_escape")
```
3.可以实现自己的自定义动态参数化方案或扩展
```python
def pytest_generate_tests(metafunc):
    #""" generate (multiple) parametrized calls to a test function."""
    if "param" in metafunc.fixturenames:
        metafunc.parametrize("param",metafunc.module.par_to_test,ids=metafunc.module.case, scope="function")
 
```
然后测试用例的编写如下：
```python
import pytest
import requests
 
from utils.get_data import get_data_path
from utils.get_data import get_test_data
import logging
case,par_to_test = get_test_data(get_data_path(__file__))
 
 
class TestFixture3(object):
    """
    """
    def test_fixture_3(self,param,env):
        url= env["host"]["local"]+env["APIS"]["add_message"]
        response = requests.request("POST", url, data=param[3], headers=param[1])
        res = response.json()
        print(res)

```
测试数据：
```python
{
 
    "test": [
      {
        "case": "这是第一个测试用例",
        "headers": {
          "Content-Type": "application/x-www-form-urlencoded"
        },
        "querystring": {
        },
        "payload": {
                "mid" :"115",
                "name" :"android9",
                "content" : "8" ,
                "status": "1",
                "author" :"xixi"
        },
        "expected":
          {
}
      },
 
      {
        "case": "这是第2个测试用例",
        "headers": {
          "Content-Type": "application/x-www-form-urlencoded"
        },
        "querystring": {
        },
        "payload": {
                "mid" :"115",
                "name" :"android9",
                "content" : "8" ,
                "status": "1",
                "author" :"xixi"
        },
        "expected":
          {
}
      },
 
      {
        "case": "这是第3个测试用例",
        "headers": {
          "Content-Type": "application/x-www-form-urlencoded"
        },
        "querystring": {
        },
        "payload": {
                "mid" :"115",
                "name" :"android9",
                "content" : "8" ,
                "status": "1",
                "author" :"xixi"
        },
        "expected":
          {
}
      }
 
 
    ]
}
```
获取测试数据的代码：
```python
def get_data_path(case_path):
    file_name = os.path.dirname(case_path).split(os.sep + 'tests' + os.sep, 1)
    test_data = os.sep.join([file_name[0], 'data', file_name[1], os.path.basename(case_path).replace('.py', '.json')])
    return test_data
 
def get_test_data(test_data_path):
    case = []
    headers = []
    querystring = []
    payload = []
    expected = []
    with open(test_data_path,encoding='utf-8') as f:
        dat = json.loads(f.read())
        test = dat['test']
        for td in test:
            case.append(td['case'])
            headers.append(td.get('headers', {}))
            querystring.append(td.get('querystring', {}))
            payload.append(td.get('payload', {}))
            expected.append(td.get('expected', {}))
    list_parameters = list(zip(case, headers, querystring, payload, expected))
    return case,list_parameters
```
##Conclusion
pytest通过这种plugin的方式，大大增强了这个测试框架的实用性，可以看到pytest本身的许多组件也是通过plugin的方式加载的，可以说pytest就是由许许多多个plugin组成的。另外，通过定义好一些hook spec，可以有效地控制plugin的“权限”，再通过类似pytest.hookimpl这样的装饰器又可以增强了各种plugin的“权限”。这种design对于pytest这样复杂的框架而言无疑是非常重要的，这可能也是pytest相比于其他测试框架中越来越?的原因吧。

# pytest测试框架-强大的Fixture功能
##fixture是 干什么用的？？
fixture是在测试函数运行前后，由pytest执行的外壳函数；代码可以定制，满足多变的测试需求;  
包括定义传入测试中的数据集，配置测试前系统的初始状态，为批量测试提供数据源等等...  
fixture是pytest用于将测试前后进行预备，清理工作的代码分离出核心测试逻辑的一种机制！  
例如：
```python
@pytest.fixture() # 装饰器用于声明函数是一个fixture
def some_data():
    return 48
 
def test_some_data(some_data):
    assert some_data == 48

"""
如果测试函数的参数列表中包含fixture名字，那么pytest会检测到，  
检测顺序是：优先搜索该测试所在的模块，然后搜索conftest.py
并在测试函数运行之前执行该fixture，
fixture可以完成测试任务，也可以返回数据给测试函数
pytest --setup-show test_example1.py
"""
```
##fixture函数放在哪里合适?
1.可以放在单独的测试文件里  
2.如果希望多个测试文件共享fixture，可以放在某个公共目录下新建一个conftest文件，将fixture放在里面。
## 使用fixture传递测试数据
fixture非常适合存放测试数据，并且他可以返回任何数据
例如：
```python
@pytest.fixture()
def a_list():
    return [1,2,3,44,5]
 
 
def test_a_list(a_list):
    assert a_list[2] == 3
```
##指定fixture作用范围
fixture里面有个scope参数可以控制fixture的作用范围:session > module > class > function
```python

1)function
#每一个函数或方法都会调用 \
@pytest.fixture()
def first():
    print("\n获取用户名")
    a = "zhaoming"
    return a
 
 
@pytest.fixture(scope="function")
def sencond():
    print("\n获取密码")
    b = "123456"
    return b
 
 
def test_1(first):
    '''用例传fixture'''
    print("测试账号：%s" % first)
    assert first == "zhaoming"
 
 
def test_2(sencond):
    '''用例传fixture'''
    print("测试密码：%s" % sencond)
    assert sencond == "123456"
 
 
2).class #每一个类调用一次，一个类可以有多个方法
 
 
@pytest.fixture(scope="class")
def first():
    print("\n获取用户名,scope为class级别只运行一次")
    a = "zhaoming"
    return a
 
 
class TestCase():
    def test_1(self, first):
        '''用例传fixture'''
        print("测试账号：%s" % first)
        assert first == "zhaoming"
 
    def test_2(self, first):
        '''用例传fixture'''
        print("测试账号：%s" % first)
        assert first == "zhaoming"
 
 
3).module，#每一个.py文件调用一次，该文件内又有多个function和class
import pytest
 
 
@pytest.fixture(scope="module")
def first():
    print("\n获取用户名,scope为module级别当前.py模块只运行一次")
    a = "zhaoming"
    return a
 
 
def test_1(first):
    '''用例传fixture'''
    print("测试账号：%s" % first)
    assert first == "zhaoming"
 
 
class TestCase():
    def test_2(self, first):
        '''用例传fixture'''
        print("测试账号：%s" % first)
        assert first == "zhaoming"
 
 
4).session
#是多个文件调用一次，可以跨.py文件调用，每个.py文件就是module
#当我们有多个.py文件的用例时候，如果多个用例只需调用一次fixture，那就可以设置为scope = "session"，并且写到conftest.py文件里
 
#conftest.py
 
import pytest
 
 
@pytest.fixture(scope="session")
def first():
    print("\n获取用户名,scope为session级别多个.py模块只运行一次")
    a = "zhaoming"
    return a
 
 
test_fixture11.py
 
import pytest
 
 
def test_1(first):
    '''用例传fixture'''
    print("测试账号：%s" % first)
    assert first == "zhaoming"
 
 
import pytest
 
 
def test_2(first):
    '''用例传fixture'''
    print("测试账号：%s" % first)
    assert first == "zhaoming"

```
##fixture的参数化
pytest支持在多个完整测试参数化方法:

1).pytest.fixture(): 在fixture级别的function处参数化

2).@pytest.mark.parametrize：允许在function或class级别的参数化,为特定的测试函数或类提供了多个argument/fixture设置。

3_.pytest_generate_tests：可以实现自己的自定义动态参数化方案或扩展。
```python
import pytest
import requests
 
par_to_test=[{
      "case": "serach a word :haha",
      "headers": {},
      "querystring": {
        "wd":"hah"
      },
      "payload": {},
      "expected": {
        "status_code":200
      }
    },
{
      "case": "serach a word2 :kuku",
      "headers": {},
      "querystring": {
        "wd":"kuku"
      },
      "payload": {},
      "expected": {
        "status_code":200
      } },
 
{
      "case": "serach a word3 :xiaoyulaoshi",
      "headers": {},
      "querystring": {
        "wd":"xiaoyulaoshi"
      },
      "payload": {},
      "expected": {
        "status_code":200
      } }
]
 
@pytest.fixture(params = par_to_test)
def class_scope(request):
    return request.param
 
def test_baidu_search(class_scope):
    url = "https://www.baidu.com"
    r = requests.request("GET", url, data=class_scope["payload"], headers=class_scope["headers"], params=class_scope["querystring"])
    assert r.status_code == class_scope["expected"]["status_code"]
```

# pytest配置文件
##一.pytest里都有哪些非测试文件？
1.pytest.ini: pytest的主配置文件,可以改变pytest的默认行为,其中有很多可配置的选项

2.conftest.py:是本地的插件库,其中的hook函数和fixture将作用于该文件所在的目录以及所有子目录。

3.__init__py:每个测试子目录都包含该文件时,那么在多个测试目录中可以出现同名测试文件。

##二.如何查看ini文件选项

使用pytest--help命令查看pytest.ini的所有设置选项 pytest --help

##三.更改默认命令行选项
我们已经用过很多pytest命令行选项了,比如-v--verbose可以输出详细信息,
~~~
[pytest]

addopts = -v --alluredir ./allure-results

ps：
~~~
###如何使用allure生成报告？

1.brew install allure（安装allure命令）

2.安装 pip install allure-pytest （安装相应的allure的包）

3.运行case时增加命令行选项 pytest -v --alluredir ./allure-results test.py

4.生成报告 allure generate allure-results -o allure

 

##有哪些常用的命令行选项呢？

-v：输出详细信息

--collect-only 展示在给定的配置下哪些测试用例会被运行

-k 允许使用表达式指定希望运行的测试用例

-m marker用于标记测试并分组

 
##四.注册标记来防范拼写错误

自定义标记可以简化测试工作，但是标记容易拼写错误，默认情况下不会引起错误，pytest以为这是另外一个标记。为了避免拼写错误，可以在pytest.ini文件里注册标记

markers =

data_file: a test_data get_and_format marker

通过命令查看：

pytest --markers

没有注册的标记不会出现在 --markers列表里，如果使用 --strict选项，遇到拼写错误的标记或未注册的标记就会报错

如何自己增加一个测试函数的标记呢？

@pytest.mark.smoke

pytest -m 'smoke' test.py

 

##五.指定pytest的最低版本号

minversion = 6.0

minversion选项可以指定运行测试用例的pytest的最低版本

 

##六.指定pytest忽略某些目录

pytest执行用例搜索时，会递归遍历所有子目录，包括某些你明知没必要遍历的目录

norecursedirs = .* data config utils

可以使用norecursedirs缩小pytest的搜索范围

指定访问目录

testpaths = tests


##七.避免文件名冲突
增加__init__.py 文件
