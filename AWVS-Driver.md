


AWVS-Driver 是一个AWVS的API系统的RPC封装驱动。

# 1.工具使用


AWVS本身提供了REST API的接口,通过进一步的抽象，简化和隐藏了复杂的调用过程。
为了便于简单实现对AWVS的操作，最后就变成了简单的一条命令调用。


```
python manage.py dsl -d lua.ren
```

Django Command的功能实现，是整个调用时许的入口，假设扫描的需求和设置很简答，只有一个扫描域名的设定。


# 2.功能函数

扫描功能实现，只靠一整个时序链调和来完成的，如果直接从Django Command调用Django RPC，参于的调用数据总体会比再加入一层REST API调用更简单。
而整个调用层级的构建，让一个复杂的API调用，分角简单化了。


对于AWVS最核心的驱动函数：一个是授权auth，另一个就是添加测试任务。


## 2.1 授权

meta数据结构中存放的是基本的授权用户信息, email和password。

```python
    def auth(self, meta):
        import urllib2
        import ssl
        import json
        ssl._create_default_https_context = ssl._create_unverified_context
        url_login="https://localhost:3443/api/v1/me/login"
        
        send_headers_login={
                'Host': 'localhost:3443',
                'Accept': 'application/json, text/plain, */*',
                'Accept-Language': 'zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3',
                'Accept-Encoding': 'gzip, deflate, br',
                'Content-Type': 'application/json;charset=utf-8'
                }
        data_login='{"email":"' +meta['email'] + '",' + '"password":"'+ meta['password']+'","remember_me":false}'
        req_login = urllib2.Request(url_login,headers=send_headers_login)
        response_login = urllib2.urlopen(req_login,data_login)
        xauth = response_login.headers['X-Auth']
        COOOOOOOOkie = response_login.headers['Set-Cookie']
        print COOOOOOOOkie,xauth
        return True
```



## 2.2 添加扫描任务


用Auth取回的Cookie信息，再进行API的调用，来完玘任务注册。


```python
    def addTarget(self, formaturl):
        url="https://localhost:3443/api/v1/targets"
        send_headers2={ 
        'Host':'servers:3443',
        'Accept': 'application/json, text/plain, */*',
        'Accept-Language': 'zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3',
        'Content-Type':'application/json;charset=utf-8',
        'X-Auth':xauth,
        'Cookie':COOOOOOOOkie,
        }
        
        try:
            for i in formaturl:
                target_url='http://'+i.strip()
                data='{"description":"222","address":"'+target_url+'","criticality":"10"}'
                req = urllib2.Request(url,headers=send_headers2)
                response = urllib2.urlopen(req,data)
                jo=eval(response.read())
                target_id=jo['target_id']
            
         
            url_scan="https://localhost:3443/api/v1/scans"
            headers_scan={
           'Host': 'localhost:3443',
           'Accept': 'application/json, text/plain, */*',
           'Accept-Language': 'zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3',
           'Accept-Encoding': 'gzip, deflate, br',
           'Content-Type': 'application/json;charset=utf-8',
           'X-Auth':xauth,
           'Cookie':COOOOOOOOkie,
            }
            data_scan='{"target_id":'+'\"'+target_id+'\"'+',"profile_id":"11111111-1111-1111-1111-111111111111","schedule":{"disable":false,"start_date":null,"time_sensitive":false},"ui_session_id":"66666666666666666666666666666666"}'
            req_scan=urllib2.Request(url_scan,headers=headers_scan)
            response_scan=urllib2.urlopen(req_scan,data_scan)
            print response_scan.read()
            
        except Exception,e:
            print e
        return True
```


这两个函数是最底层的函数，关于AWVS的API封装DEMO网上有，大家可参自行参考。


# 3.测试用例 


如果直接联调， 调试成本其实也不低，为了可以更好的理解，这套AWVS的函数，是如何在我设计的这个结构中被调用的。我们用PYTSET把重点函数做了单体测试。


## 3.1 测试认证过程

```python
@pytest.mark.scan
def test_5(setup_module):
    import awvs 
    ins = awvs.AWVS()
    ins.auth({"email":"name", "password":"pwd"})
    assert True == ret
```


## 3.2 测试添加扫描任务过程 

```python
@pytest.mark.scan
def test_6(setup_module):
    import awvs 
    ins = awvs.AWVS()
    ret = ins.addTarget(['lua.ren\n','candylab.net\n'])
    assert True == ret
```


## 3.3 添加认证并扫描的过程

```python
@pytest.mark.scan
def test_7(setup_module):
    import awvs 
    ins = awvs.AWVS()
    ins.auth({"email":"name", "password":"pwd"})
    ret = ins.addTarget(['lua.ren\n','candylab.net\n'])
    assert True == ret
```

其实认证和扫描的过程，前期是拆开测试的，如果不先认证，基出上就异常了，无法添加扫描任务。 单测试用例是为了提供单体质量，提高结合测试的成功效率。

整体测试的还是auth函数用户信息字典入参的测试，与addTarget函数域名列表的测试。



## 3.4 自动化测试

这个工程使用的测试工具是pytest。我们想通过自动监听test.py的python单体测试程序源码的变更，自动调用pytest去扫行单体测试脚本。

如果在linux平台一下可以使用tup，是一个很好用的工具。因我们在mac环境下扫行单体测试程序，我们使用fswatch完成这个功能。


### 3.4.1 安装fswatch

```
brew intall fswatch
```



### 3.4.2 监听脚本

```shell
#!/bin/bash 
DIR=$1 
if [ ! -n "$DIR" ] ;then 
    echo "you have not choice Application directory !" 
    exit 
fi 

fswatch $DIR | while read file 
do 
    #echo "${file} was modify" >> unittest.log 2>&1 
    echo "${file} was modify" 
    pytest -v -s -m"scan" ${file} 
done
```

### 3.4.3 驱动脚本
   
```shell
#!/bin/bash 
sh autotest.sh test.py
```


# 4. RPC接口功能

当单体功能达到我们设想的要求时，需要封装一个RPC服务对外提供服务。



```python
@jsonrpc_method('myapp.autoscanner')
def auto_scanner(request, domain='lua.ren'):
    import awvs 
    ins = awvs.AWVS()
    ins.auth({"email":"name", "password":"pwd"})
    ins.addTask(['lua.ren\n','candylab.net\n'])
    return True 
```


RPC功能相当于把单体调用集成到一个接口，正常完备的结果要做入参的检查工作，过滤掉非法入参。

因为我们最开始是考虑用新加的REST API作与外部调用者的通信， 在REST API做入参检查， 并且 REST API不需求外部调用者谳用时，要依赖RPC客户端。


# 5. Django Command功能实现

实现了单体对AWVS的封装，并实现RPC服务，先不考虑REST和前端的控制，实际上我们想当于把AWVS的REST功能命令行化。

```python
from django.core.management.base import BaseCommand, CommandError
import traceback
class Command(BaseCommand):
    def add_arguments(self, parser):
        parser.add_argument(
            '-d',
            '--domain',
            action='store',
            dest='domain',
            default='lua.ren',
            help='domain.',
        )

    def handle(self, *args, **options):
        try:
            if options['domain']:
                print 'scan domain, %s' % options['domain']

            from jsonrpc.proxy import ServiceProxy
            s = ServiceProxy('http://localhost:5000/json/') 
            s.myapp.autoscanner(options['domain'])

            self.stdout.write(self.style.SUCCESS(u'命令%s执行成功, 参数为%s' % (__file__, options['domain'])))
        except Exception, ex:
            traceback.print_exc()
            self.stdout.write(self.style.ERROR(u'命令执行出错'))
```

# 5. REST API实现


将功能性的内容用RPC实现， 将check业务划分和检查放到了REST API层，这样后端服务调用依赖RPC Server和RPC Client，而REST API调用层不用考虑这个问题。


```python
@csrf_exempt
def addItem(request):
    if request.method == 'GET':
        return JSONResponse("GET")
    
    if request.method == 'POST':
        data = JSONParser().parse(request)
        flg_key = data.has_key('key')
        if not flg_key:            
            return JSONResponse('key is empty!')

        access_key = data['key']
        if cmp(access_key, "test"):
            return JSONResponse("access key error.")

        flg_domain = data.has_key('domain')
        if not flg_domain:            
            result = {"error":"-1","errmsg":"domain is empty"}
            return HttpResponse(json.dumps(result,ensure_ascii=False),content_type="application/json,charset=utf-8")


        from jsonrpc.proxy import ServiceProxy
        s = ServiceProxy('http://localhost:5000/json/')
        import awvs 
        ins = awvs.AWVS()
        ins.auth({"email":"name", "password":"pwd"})
        ins.addTask(['lua.ren\n','candylab.net\n'])

        result = {"error":"0","errmsg":"none"}
        return HttpResponse(json.dumps(result,ensure_ascii=False),content_type="application/json,charset=utf-8")
```


Django REST让REST的实现更便利，这样可以把重点放到业务逻辑检查对接，相对单层的测试更有重点。

REST API路由可以快速建立。

```python
urlpatterns = [
    url(r'scanner/$', views.addItem),          
]
```

用CURL客户端测试REST API。

```
curl -l -H "Content-type: application/json" -X POST -d '{"key":"test","domain":"test.com"}'  127.0.0.1:8080/scanner/
```




# 6. 命令行


最终我们实现了AWVS的REST API的RPC和REST封装，然后命令行化，当然的其中RPC和REST API可以其它的地方复用。

## 6.1 Django Command

```
python manage.py dsl -d lua.ren
```

## 6.2 CURL & REST API

```
curl -l -H "Content-type: application/json" -X POST -d '{"key":"test","domain":"test.com"}'  127.0.0.1:8080/scanner/
```
