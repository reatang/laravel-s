# 已知的兼容性问题

## 使用包 [jenssegers/agent](https://github.com/jenssegers/agent)
> [监听系统事件](https://github.com/hhxsv5/laravel-s/blob/master/README-CN.md#%E7%B3%BB%E7%BB%9F%E4%BA%8B%E4%BB%B6)

```PHP
// 重置Agent
\Event::listen('laravels.received_request', function (\Illuminate\Http\Request $req, $app) {
    $app->agent->setHttpHeaders($req->server->all());
    $app->agent->setUserAgent();
});
```

## 使用包 [barryvdh/laravel-debugbar](https://github.com/barryvdh/laravel-debugbar)
> 官方不支持`cli`模式，需手动注释掉此判断，但启用后不排除会有其他问题。

```PHP
// 搜索 runningInConsole()，并注释掉该判断
$this->enabled = $configEnabled /*&& !$this->app->runningInConsole()*/ && !$this->app->environment('testing');
```

## 使用包 [laracasts/flash](https://github.com/laracasts/flash)
> 常驻内存后，每次调用flash()会追加消息提醒，导致叠加展示消息提醒。有以下两个方案。

1.通过中间件在每次请求`处理前`或`处理后`重置$messages `app('flash')->clear();`。

2.每次请求处理后重新注册`FlashServiceProvider`，配置[register_providers](https://github.com/hhxsv5/laravel-s/blob/master/Settings-CN.md)。

## 不能使用这些函数

- `flush`/`ob_flush`/`ob_end_flush`/`ob_implicit_flush`：`swoole_http_response`不支持`flush`。

- `dd()`/`exit()`/`die()`: 将导致Worker/Task/Process进程立即退出，建议通过抛异常跳出函数调用栈，[Swoole文档](https://wiki.swoole.com/wiki/page/501.html)。

- `header()`/`setcookie()`/`http_response_code()`：HTTP响应只能通过Laravel/Lumen的`Response`对象。

## 不能使用的全局变量

- `$_SESSION`

## 大小限制

- `Swoole`限制了`GET`请求头的最大尺寸为`8KB`，建议`Cookie`的不要太大，不然`$_COOKIE`可能解析失败。

- `POST`数据或文件上传的最大尺寸受`Swoole`配置[`package_max_length`](https://wiki.swoole.com/wiki/page/301.html)影响，默认上限`2M`。

## Inotify监听文件数达到上限
> `Warning: inotify_add_watch(): The user limit on the total number of inotify watches was reached`

- `Linux`中`Inotify`监听文件数默认上限一般是`8192`，实际项目的文件数+目录树很可能超过此上限，进而导致后续的监听失败。

- 增加此上限到`524288`：`echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf && sudo sysctl -p`，注意`Docker`时需设置启用`privileged`。

## 注意include/require与(include/require)_once
> 看看鸟哥这篇文章[再一次, 不要使用(include/require)_once](http://www.laruence.com/2012/09/12/2765.html)

- 引入`类`、`接口`、`trait`、`函数`时使用(include/require)_once，其他情况使用include/require。

- 在多进程模式下，子进程会继承父进程资源，一旦父进程引入了某个需要被执行的文件，子进程再次`require_once()`时会直接返回`true`，导致该文件执行失败。此时，你应该使用include/require。


## 对于swoole版本号小于 1.9.17 的环境

小于 1.9.17 静态资源文件将由LaravelS组件自己处理，由于php环境的一些原因可能识别不到静态资源的MimeType类型，比如会造成浏览器不识别css和js文件的情况。

- 尽量使用新版的swoole
- 使用 \Symfony\Component\HttpFoundation\File\MimeType\MimeTypeGuesser::getInstance()->register(MimeTypeGuesserInterface $guesser); 手动添加自定义解析器。

