## <center>与django的交互</center>
### 配置URL
在app.urls.py文件编辑URL如下格式，然后再在去project.urls那里引入即可

```
# urls.py
from django.urls import path
from . import views
urlpatterns = [
    path('articles/2003/', views.special_case_2003),
    path('articles/<int:year>/', views.year_archive),
    path('articles/<int:year>/<int:month>/', views.month_archive)]
```

### 视图 函数
视图：
```
# views.py
from django.http import HttpResponse
def my_view(request):
    return HttpResponse(status=201) # Return a "created" (201) response code.
```

### django.http.HttpRequest
***[HttpRequest类属性官方文档](https://docs.djangoproject.com/en/dev/ref/request-response/#django.http.HttpRequest "HttpRequest类属性官方文档")***

当请求到来时，django会创建一个 HttpRequest 对象来包含请求的元数据，然后将这个 HttpRequest 对象作为视图函数的第一个参数
```
# from django.http.request import HttpRequest / from django.http import HttpRequest（http目录的__init__文件引入了HttpRequest类）
# django.http.request.py
class HttpRequest:
    # A basic HTTP request对象. #

scheme             -> str                                   # 模式，HTTP 还是 HTTPS
body               -> bytes                                 # 二进制表示，原生 HTTP 请求数据
path               -> str                                   # 请求的完整URL，不包括模式和域名，例如："/music/bands/the_beatles/"
path_info          -> str                                   # 请求的 URL ，只包括 WSGIServer 发送到app的URL
method             -> str                                   # 请求方法
encoding           -> str                                   # 请求数据的编码方式，如果是 None 则使用 DEFAULT_CHARSET 默认字符配置
content_type       -> str                                   # ????
GET                -> QueryDict: Dict like                  # 包含了 GET 请求提交的所有参数，ps：QueryDict 是 Dict 的子类，解决了 HTML 复选框这种一个 KEY 多个 Value 参数的 Dict 冲突
POST               -> QueryDict: Dict like                  # 包含 POST 提交的表单数据，不包括上传的文件
COOKIES            -> dict                                  # 包含了所有的 cookies
FILES              -> MultiValueDict: Dict like             # 包含了所有上传的文件，key 是上传时表单上的 name，value 是一个 UploadedFile 对象，只有当 POST 的 Content-Type 是 "multipart/form-data" 格式的时候才会有数据，ps:MultiValueDict 是 Dict 的子类，
```

### django.http.HttpResponse
***[HttpResponse类属性官方文档](https://docs.djangoproject.com/en/dev/ref/request-response/#django.http.HttpResponse "HttpResponse类属性官方文档")***

***[StreamingHttpResponse类属性官方文档](https://docs.djangoproject.com/en/dev/ref/request-response/#django.http.StreamingHttpResponse "StreamingHttpResponse")***

服务端的工作就是根据请求，构造响应数据并返回

设置响应正文，可以传入string，bytestring 或者 memoryview对象 以及所有的可迭代对象
```
from django.http import HttpResponse
response = HttpResponse("Here's the text of the Web page.")
response = HttpResponse(b'Bytestrings are also accepted.')
response = HttpResponse(memoryview(b'Memoryview as well.'))

# 如果初始化之后希望增加正文，可以像文件一样写入
response = HttpResponse()
response.write("<p>Here's the text of the Web page.</p>")

# 如果传入一个可迭代对象，HttpResponse 会马上将其迭代并将结果以str返回，最后如果有 close() 的对象还会马上关闭，例如 file() 对象，如果希望以 stream 流的形式返回给客户端，则需要用到 StreamingHttpResponse 对象
```

设置响应的消息报头，用法跟字典一样，不过对于 Cache-Control 和 Vary 推荐使用 django.utils.cache.patch_cache_control() 和 django.utils.cache.patch_vary_headers()，因为这两个响应头能有多个独立的值，上面两个方法能避免中间件的设置不被覆盖
```
response = HttpResponse()
response['Age'] = 120       # 不允许有换行符
del response['Age']         # 即使没有该字段删除也不会抛出异常，这点跟 dict 不同

# 如果希望返回一个文件，则需要设置 Content-Type 为对应值，并且设置 Content-Disposition 响应头，例如下面返回一个xls文件
response = HttpResponse(my_data, content_type='application/vnd.ms-excel')
response['Content-Disposition'] = 'attachment; filename="foo.xls"'
```

HttpResponse 属性
```
content             -> bytes  # 返回内容
charset             -> str    # 字符集
status_code         -> int    # 返回状态码
reason_phrase       -> str    # 与状态码配对的文本，不设置则自动使用默认标准
streaming           -> bool   # 为了方便区分是 HttpResponse 对象还是 StreamingHttpResponse 对象，前者是 False , 后者是 True
closed              -> bool   # 如果为True则表明 Response 对象相关的属性都已经关闭
```

### django.http.StreamingHttpResponse
StreamingHttpResponse 和 HttpResponse 都是 HttpResponseBase 的子类

前面说了当 HttpResponse 用当可迭代对象初始化的时候会直接执行完该可迭代对象，但是当一个可迭代对象非常大的时候（例如一个大CSV数据文件），内存会崩溃，此时就可以使用 Stream 流式传输

StreamingHttpResponse 与 HttpResponse 几乎一样，但有以下差别
- 只能传入可迭代对象，并且每次 yield 返回的 string 会作为传输内容
- 只能通过传入的可迭代对象来获取传输内容（HttpResponse初始化之后可以访问content属性）
- 没有 content 属性，只有 streaming_contetn 属性
- 不能像 file-like 对象那样调用 tell(), write()方法在初始化后添加返回内容