# WSGI，uwsgi 与 uWSGI
1. Web Server Gate Interface(WSGI)是 Python 制定的一个规范，用来规范化 Web Server（web 服务器） 与 Web Application（web 框架）之间的通信（大家约定好接口之后，任意的 web 服务器 能与 任意的 web 框架组合）。
2. uwsgi 是 uWSGI 的独占协议，用于定义传输信息的类型(type of information)，与 WSGI 是两种东西，据说速度比 fcgi 协议快10倍。
3. uWSGI 是一个web服务器，实现了WSGI协议、uwsgi协议、http协议等。

# WSGI 协议
## WSGI Application
Web 框架需要实现一个可调用对象，需要接收两个参数：
1. 一个字典，该字典可以包含了客户端请求的信息以及其他信息，可以认为是请求上下文，一般叫做environment（编码中多简写为environ、env）
2. 一个用于发送HTTP响应状态（HTTP status ）、响应头（HTTP headers）的 **回调函数**
以 Django 的 WSGI Application 为例：
```
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        request = self.request_class(environ)
        response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = list(response.items())
        for c in response.cookies.values():
            response_headers.append(('Set-Cookie', c.output(header='')))
        start_response(status, response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```
1. 加载所有中间件，以及执行框架相关的操作，设置当前线程脚本前缀，发送请求开始信号；
2. 处理请求，调用get_response()方法处理当前请求，该方法的的主要逻辑是通过urlconf找到对应的view和callback，按顺序执行各种middleware和callback。
3. 调用由server传入的start_response()方法将响应header与status返回给server。
4. 返回响应正文


## Web Server
负责获取http请求，然后调用上面的 WSGIHandler 将请求传递给 WSGI application，由 application 处理请求后返回 response。

# uWSGI 服务器应用
1. 超快的性能
2. 低内存占用
3. 多app管理
4. 详尽的日志功能（可以用来分析app的性能和瓶颈）
5. 高度可定制（内存大小限制，服务一定次数后重启等）

uWSGI服务器自己实现了基于uwsgi协议的server部分，我们只需要在uwsgi的配置文件中指定application的地址，uWSGI就能直接和应用框架中的WSGI application通信。
