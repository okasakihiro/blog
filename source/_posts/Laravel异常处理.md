---
title: Laravel 异常处理
date: 2019/7/3 12:25:22
categories: 后端
tag: [PHP, Laravel]
toc: true
---

最近在学习Laravel框架，在调试过程中无意中发现了一个问题：

首先看一下路由：
```PHP
Route:get('/get-exchange-rate', 'ExchangeRateController@getExchangeRate');
```

再看一下控制器：
```PHP
class ExchangeRateController extends Controller
{
    /**
     * @param Request $request
     * @return mixed
     */
    public function getExchangeRate(Request $request)
    {
        //return ...
    }
}
```

现在想加入请求方式认证，在控制器中加入了该语句：
```PHP
if($requert->method() !== 'get') {
    //return ...
}
```

但是使用post的请求方式时，直接抛出了异常。
> Symfony \ Component \ HttpKernel \ Exception \ MethodNotAllowedHttpException
>
> **The POST method is not supported for this route. Supported methods: GET, HEAD.**


当时想了想，应该可以有三种方式解决：
1. 将路由中的方法全部修改为any，之后在每个路由中进行限制（这么解决太low了）。
2. 关闭Debug模式（掩耳盗铃）。
3. 从抛出异常的地方入手。

想了想也就第三种方法比较靠谱。

顺藤摸瓜，去 `\app\Exceptions\Handel.php` 中看了下：

```PHP
class Handler extends ExceptionHandler
{
    //...
    /**
     * Report or log an exception.
     *
     * @param  \Exception  $exception
     * @return void
     */
    public function report(Exception $exception)
    {
        parent::report($exception);
    }

    /**
     * Render an exception into an HTTP response.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Exception  $exception
     * @return \Illuminate\Http\Response
     */
    public function render($request, Exception $exception)
    {
        return parent::render($request, $exception);
    }
}
```

在这里分别有 `report()` 和 `render()` 两个方法，我们重点看一下render方法：
> 使用 HTTP 响应报告异常

说明这里可以捕获异常，再查看一下它的父类：

> **此类内容过长，直接切入重点**

无意中在最上方看到了该类中使用了很多其他封装好的异常类：

```PHP
//...
use Illuminate\Auth\AuthenticationException;
use Illuminate\Contracts\Container\Container;
use Illuminate\Contracts\Support\Responsable;
use Illuminate\Session\TokenMismatchException;
use Illuminate\Validation\ValidationException;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Http\Exceptions\HttpResponseException;
use Symfony\Component\Debug\Exception\FlattenException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Symfony\Component\HttpKernel\Exception\HttpException;
use Symfony\Component\Console\Application as ConsoleApplication;
use Symfony\Component\HttpFoundation\Response as SymfonyResponse;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
use Symfony\Component\Debug\ExceptionHandler as SymfonyExceptionHandler;
use Illuminate\Contracts\Debug\ExceptionHandler as ExceptionHandlerContract;
use Symfony\Component\HttpFoundation\Exception\SuspiciousOperationException;
use Symfony\Component\HttpFoundation\RedirectResponse as SymfonyRedirectResponse;
```

第一反应想到是不是可以使用 `instantof` 来区别这些异常内容，改造一下如下：

```PHP
class Handler extends ExceptionHandler
{
    //...
    /**
     * Render an exception into an HTTP response.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Exception  $exception
     * @return \Illuminate\Http\Response
     */
    public function render($request, Exception $exception)
    {
        //拦截405异常
        if ($exception instanceof MethodNotAllowedHttpException) {
            return response([
                'code' => 405,
                'message' => 'Method not allowed'
            ]);
        }
        //拦截404异常
        if ($exception instanceof NotFoundHttpException) {
            return response([
                'code' => 404,
                'message' => 'Not Found'
            ]);
        }
        // ...
        return parent::render($request, $exception);
    }
}
```

用postman测试一下，成功了！

```JSON
{"code":405,"message":"Method not allowed"}
```

可接下来问题又来了，异常那么多，每一个都用 `if` + `instanceof` 方法来判断，一是看起来不太好看，二是扩展起来也不太方便，有没有更好可以判断的方法？

再回到 `Handler` 的父类中看一下，发现存在一个 `isHttpException()` 方法：

```PHP
protected function isHttpException(Exception $e)
    {
        return $e instanceof HttpExceptionInterface;
    }
```

原来Laravel已经为我们封装了这个方法，那么把上面的方法改造一下：

```PHP
class Handler extends ExceptionHandler
{
    //...
    /**
     * Render an exception into an HTTP response.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Exception  $exception
     * @return \Illuminate\Http\Response
     */
    public function render($request, Exception $exception)
    {
        //判断是否存是HTTP异常
        if ($this->isHttpException($exception)) {
            switch ($exception->getStatusCode()) {
                //404异常
                case 404:
                    return response([
                        'code' => 404,
                        'message' => 'Not Found'
                    ]);
                    break;
                    //405异常
                case 405:
                    return response([
                        'code' => 405,
                        'message' => 'Method not allowed'
                    ]);
                    break;
                    //其他异常
                default:
                    return response([
                        'code' => 500,
                        'message' => 'Server Error'
                    ]);
            }
        }
        return parent::render($request, $exception);
    }
}
```

至此，针对于请求方法的限制已经实现了，以目前的学识感觉这种方法最靠谱，如果有更好的方法，还请各位前辈指路。

