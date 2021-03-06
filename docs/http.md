---
layout: default
title: HTTP(PSR7)
anchor: http
---

# HTTP(PSR7) {#http-title}

Wandu Http는 PSR-7를 구현한 컴포넌트입니다. 그리고 PSR-7에서 제시한 규격 이외에 Session, Cookie, Paramater(ParsedBody 및 QueryParams) 등을 제공하고 있습니다.

## Basic Usage

일반적인 PHP 사용환경에서는 다음과 같은 순서로 동작시킬 수 있습니다.

```php
<?php
use Wandu\Http\Cookie\CookieJarFactory;
use Wandu\Http\Factory\ResponseFactory;
use Wandu\Http\Factory\ServerRequestFactory;
use Wandu\Http\Factory\UploadedFileFactory;
use Wandu\Http\Sender\ResponseSender;
use Wandu\Http\Session\Adapter\FileAdapter;
use Wandu\Http\Session\SessionFactory;

// Factory 객체들을 생성합니다.
$requestFactory = new ServerRequestFactory(new UploadedFileFactory());
$cookieManager = new CookieJarFactory();
$sessionManager = new SessionFactory(new FileAdapter(__DIR__ . '/_sess'));
$responseFactory = new ResponseFactory();
$responseSender = new ResponseSender();

// 전역변수에서 ServerRequestInterface 객체를 생성합니다.
$request = $requestFactory->createFromGlobals();

// ServerRequestInterface를 이용하여 CookieJarInterface 객체를 생성합니다.
$cookie = $cookieManager->fromServerRequest($request);

// CookieJarInterface를 이용하여 SessionInterface 객체를 생성합니다.
$session = $sessionManager->fromCookieJar($cookie);

// Closure 내부에서 echo로 출력된 내용을 ResponseInterface로 변형합니다.
$response = $responseFactory->capture(function () use ($cookie, $session) {
    $cookie->set('is_wandu_http', true);
    $session->set('is_login', true);

    echo "<h3>Cookie!!</h3>";
    echo "<pre>";
    print_r($cookie->toArray());
    echo "</pre>";
    echo "<h3>Session!!</h3>";
    echo "<pre>";
    print_r($session->toArray());
    echo "</pre>";
});

// 위에서 처리된 SessionInterface를 CookieJarInterface에 반영시킵니다.
$sessionManager->toCookieJar($session, $cookie);

// 위에서 처리된 CookieJarInterface를 ResponseInterface에 반영시킵니다.
$response = $cookieManager->toResponse($cookie, $response);

// 해당 ResponseInterface를 PHP에서 처리합니다.
$responseSender->sendToGlobal($response);

```

It's so simple! :D

## Documents

PSR-7 구현체에 대한 설명을 보고 싶으시다면 하위에 [PSR-7 Implementations](#psr7-implementations) 항목 부터 읽어주시면 됩니다. PSR-7을 이미 어느정도 아신다는 가정하에 **Wandu Http**에서만 제공하는 편리한 내용을 먼저 설명하겠습니다.

### File Uploader

Wandu Http는 PSR7에서 사용하기 쉽게 몇가지 기능들을 제공하고 있습니다. 그 중 하나가 바로 File Uploader입니다. PSR-7에서는 업로드 객체를 만드는 내용까지만 명시되어있습니다. 하지만 Wandu에서는 해당 업로더를 제공하고 있습니다.

**Example.**

```php
<?php
use Wandu\Http\File\Uploader;

$uploader = new Uploader(__DIR__ . '/files');
$result = $uploader->uploadFiles($request->getUploadedFiles());

// uploaded files' name return.
// ex. ['thumbnail' => '/your/path/files/edd93193acab5bedf1c27a8efc095e7ba0a79945.jpg']
print_r($result);
```

### Cookie

#### CookieJar

`Wandu\Http\Cookie\CookieJar` is implementation of `Wandu\Http\Contracts\CookieJarInterface`

```php
<?php
namespace Wandu\Http\Contracts;

use Wandu\Http\Contracts\ParameterInterface;

interface CookieJarInterface extends ArrayAccess, IteratorAggregate, ParameterInterface
{

    /* Methods */

    public CookieJarInterface set(string $name, string $value, DateTime $expire = null)

    public CookieJarInterface remove(string $name)


    /* Inherited methods (from ParameterInterface) */

    public ParameterInterface? setFallback(ParameterInterface $fallback)

    public array toArray()

    public mixed get(string $key, mixed $default = null)

    public boolean has(string $key)
}
```

#### CookieJarFactory

This is useful for bringing a cookie jar object(`CookieJarInterface`) from the server request object(`ServerRequestInterface`). And a cookie jar object brought from the server request object must apply to the response object(`ResponseInterface`).

```php
<?php
namespace Wandu\Http\Cookie;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Wandu\Http\Contracts\CookieJarInterface;

class CookieJarFactory
{

    /* Methods */

    public CookieJarInterface fromServerRequest(ServerRequestInterface $request)

    public ResponseInterface toResponse(CookieJarInterface $cookieJar, ResponseInterface $response)
}
```

**Example.**

```php
<?php
use Wandu\Http\Cookie\CookieJarFactory;
use Wandu\Http\Psr\Response;
use Wandu\Http\Psr\ServerRequest;

$cookieFactory = new CookieJarFactory();
$cookie = $cookieFactory->fromServerRequest($request); // request is a ServerRequest object.

// -- start controller --
//$cookie->set('cookie3', 'new33!!');
//$cookie->set('cookie4', 'new444');

$response = new Response();
$response = $cookieFactory->toResponse($cookie, $response);
```

### Session

#### Session

`Wandu\Http\Session\Session` is implementation of `Wandu\Http\Contracts\SessionInterface`

```php
<?php
namespace Wandu\Http\Contracts;

use Wandu\Http\Contracts\ParameterInterface;

interface SessionInterface extends ArrayAccess, ParameterInterface
{

    /* Methods */

    public string getId()

    public SessionInterface set(string $name, mixed $value)

    public SessionInterface flash(string $name, mixed $value)

    public array getRawParams()

    public SessionInterface remove(string $name)
    

    /* Inherited methods (from ParameterInterface) */

    public ParameterInterface? setFallback(ParameterInterface $fallback)

    public array toArray()

    public mixed get(string $key, mixed $default = null)

    public boolean has(string $key)
}
```

#### SessionFactory

This is useful for bringing a session object(`SessionInterface`) from the cookie jar  object(`CookieJarInterface`). And a session object brought from the cookie jar object must apply to the same cookie jar object(`CookieJarInterface`).

```php
<?php
namespace Wandu\Http\Session;

use DateTime;
use Wandu\Http\Contracts\CookieJarInterface;
use Wandu\Http\Contracts\SessionAdapterInterface;
use Wandu\Http\Contracts\SessionInterface;

class SessionFactory
{

    /* Methods */

    public __construct(SessionAdapterInterface $adapter, array $config = [])

    public SessionInterface fromCookieJar(CookieJarInterface $cookieJar)

    public SessionInterface toCookieJar(SessionInterface $session, CookieJarInterface $cookieJar)
}
```

**Example.**

```php
<?php
use Wandu\Http\Contracts\CookieJarInterface;
use Wandu\Http\Session\SessionFactory;
use Wandu\Http\Session\Adapter\FileAdapter;

$sessionManager = new SessionFactory(new FileAdapter(__DIR__ . '/_sess')); // default Adapter
$session = $sessionManager->fromCookieJar($cookie); // $cookie is the instance of CookieJarInterface

//$session->set('sess3', '!!');
//$session->set('sess4', '??');

$sessionManager->toCookieJar($session, $cookie);
```

### Session Adapter

There are three adapters.

1. `File Adapter`
1. `Redis Adapter`
1. `Global Adapter`
 
#### File Adapter

**Example.**

```php
<?php
use Wandu\Http\Session\SessionFactory;
use Wandu\Http\Session\Adapter\FileAdapter;

$sessionManager = new SessionFactory(new FileAdapter(__DIR__ . '/_sess'));
```

#### Redis Adapter

**Example.**

```php
<?php
use Wandu\Http\Session\SessionFactory;
use Wandu\Http\Session\Adapter\RedisAdapter;

$redisClient = new \Predis\Client();
$sessionManager = new SessionFactory(new RedisAdapter($redisClient));
```

#### Global Adapter

**refactoring only**

리팩토링 할 때만 사용하여 주십시오. 그 외에는 권장하지 않습니다.

**Example.**

```php
<?php
use Wandu\Http\Session\SessionFactory;
use Wandu\Http\Session\Adapter\GlobalAdapter;

$sessionManager = new SessionFactory(new GlobalAdapter()); // get session by $_SESSION
```

### Parameter

#### Parameter

`ServerRequestInterface`::`getParsedBody()` and `getQueryParams()` return array. If you want to use these array as a object, use `Parameter` class.

```php
<?php
namespace Wandu\Http\Contracts;

interface ParameterInterface
{

    /* Methods */

    public ParameterInterface? setFallback(ParameterInterface $fallback)

    public array toArray()

    public mixed get(string $key, mixed $default = null)

    public boolean has(string $key)
}

interface QueryParamsInterface extends ParameterInterface
{
}

interface ParsedBodyInterface extends ParameterInterface
{
}
```

**Example.**

```php
<?php
use Wandu\Http\Parameters\Parameter;

$parsedBody = new Parameter($request->getParsedBody());

$userId = $parsedBody->get('user_id', 0); // return user_id, or 0.
```

```php
<?php
use Wandu\Http\Parameters\Parameter;

$queryParams = new Parameter($request->getQueryParams());

$userId = $queryParams->get('user_id', 0); // return user_id, or 0.
```

### Psr7 Implementations

#### Stream

There are very useful 4 streams.

1. `Default Stream`
1. `String Stream`
1. `PHP Input Stream`
1. `Generator Stream`

##### Default Stream

```php
<?php
namespace Wandu\Http\Psr;

use Psr\Http\Message\StreamInterface;

class Stream implements StreamInterface
{

    public __construct(string $stream = 'php://memory', string $mode = 'r')
    
    /* Inherited methods (from StreamInterface) */
    ...
}
```

**Example.**

```php
<?php
use Wandu\Http\Psr\Stream;

// read / write
$stream = new Stream('php://memory', 'w+');

// from http request body (do not use. use PHP Input Stream)
$stream = new Stream('php://input');
```

##### String Stream

```php
<?php
namespace Wandu\Http\Psr\Stream;

use Psr\Http\Message\StreamInterface;

class StringStream implements StreamInterface
{

    public __construct(string $context = '')
    
    /* Inherited methods (from StreamInterface) */
    ...
}
```

**Example.**

```php
<?php
use Wandu\Http\Psr\Stream\StringStream;

$stream = new StringStream('hello world!!');
echo $stream->__toString(); // "hello world!!"
```

##### PHP Input Stream

```php
<?php
namespace Wandu\Http\Psr\Stream;

use Psr\Http\Message\StreamInterface;

class PhpInputStream implements StreamInterface
{
    /* Inherited methods (from StreamInterface) */
    ...
}
```

**Example.**

```php
<?php
use Wandu\Http\Psr\Stream\PhpInputStream;

$stream = new PhpInputStream();

echo $stream->__toString(); // print php://input.
echo $stream->__toString(); // also print php://input.
```

##### Generator Stream

It's very useful to print **big** csv, xml, json and so on.

```php
<?php
namespace Wandu\Http\Psr\Stream;

use Closure;
use Psr\Http\Message\StreamInterface;

class GeneratorStream implements StreamInterface
{

    public __construct(Closure $handler)

    /* Inherited methods (from StreamInterface) */
    ...
}
```

**Example.**

```php
<?php
use Wandu\Http\Psr\Stream\GeneratorStream;

$stream = new GeneratorStream(function () {
    for ($i = 0; $i < 10; $i++) {
        yield sprintf("%02d\n", $i);
    }
});

echo $stream->__toString(); // print '00\n01\n ... 08\n09\n'
echo $stream->__toString(); // also print '00\n01\n ...08\n09\n'
```


#### Response

.. 작성중..

#### ServerRequest

.. 작성중..

#### UploadedFile

.. 작성중..

#### Uri

**Wandu\Http\Psr\Uri::join**

It executes like urljoin function in python.

```php
<?php
$uri = new Uri('http://wani.kr/hello/world');
$uriToJoin = new Uri('../other-link');

$uri->join($uriToJoin); // http://wani.kr/other-link
```

If you want to see more detail test cases, see
[this page](https://github.com/Wandu/Http/blob/master/tests/UriTest.php#L430).

**Example.**

```php
<?php
// case in-sensitive
new Uri('http://blog.wani.kr');
new Uri('http://BLOG.WANI.KR');
new Uri('https://blog.wani.kr');

// no scheme
new Uri('//blog.wani.kr');
new Uri('://blog.wani.kr');
new Uri('blog.wani.kr');

// real path
new Uri('/abc/def');

// relative path
new Uri('hello/world');

// user info
new Uri('http://wan2land@blog.wani.kr');
new Uri('http://wan2land:hello@blog.wani.kr');

// utf-8
new Uri('/hello/enwl dfk/-_-/한글'); // getPath -> '/hello/enwl%20dfk/-_-/%ED__%EA%B8_'

// query and fragment
new Uri('http://blog.wani.kr?hello=world&abc=def');
new Uri('http://blog.wani.kr/path/name?hello=world#fragment');
```

### Factory Classes

#### UploadedFileFactory

```php
<?php
namespace Wandu\Http\Factory

class UploadedFileFactory
{
    /**
     * @param array $files
     * @return \Psr\Http\Message\UploadedFileInterface[]|array
     */
    public function createFromFiles(array $files);
}
```

**Example.**

```php
<?php
use Wandu\Http\Factory\UploadedFileFactory;

$factory = new UploadedFileFactory();
$treeOfFiles = $factory->fromFiles($_FILES);
$treeOfFiles; // array of UploadedFile object.
```

#### ServerRequestFactory

```php
<?php
namespace Wandu\Http\Factory;

class ServerRequestFactory
{
    /**
     * @param \Wandu\Http\Factory\UploadedFileFactory $fileFactory
     */
    public function __construct(UploadedFileFactory $fileFactory);

    /**
     * @return \Psr\Http\Message\ServerRequestInterface
     */
    public function createFromGlobals();

    /**
     * @param string $body
     * @return \Psr\Http\Message\ServerRequestInterface
     */
    public function createFromSocketBody($body);

    /**
     * @param array $server
     * @param array $get
     * @param array $post
     * @param array $cookies
     * @param array $files
     * @param \Psr\Http\Message\StreamInterface $stream
     * @return \Psr\Http\Message\ServerRequestInterface
     */
    public function create(
        array $server,
        array $get,
        array $post,
        array $cookies,
        array $files,
        StreamInterface $stream = null
    );
}
```

**Example.**

```php
<?php
use Wandu\Http\Factory\ServerRequestFactory;
use Wandu\Http\Factory\UploadedFileFactory;

$requestFactory = new ServerRequestFactory(new UploadedFileFactory());
$serverRequest = $requestFactory->createFromGlobals();

$serverRequest; // instanceof ServerRequestInterface
```

#### ResponseSender

```php
<?php
namespace Wandu\Http\Sender;

class ResponseSender
{
    /**
     * @param \Psr\Http\Message\ResponseInterface $response
     */
    public function sendToGlobal(ResponseInterface $response);

    /**
     * @param \Psr\Http\Message\ResponseInterface $response
     * @return string
     */
    public function parseToSocketBody(ResponseInterface $response);
}
```

**Example.**

```php
<?php
use Wandu\Http\Sender\ResponseSender;

$sender = new ResponseSender;

$response = ''; // Psr7 Response Interface

$sender->sendToGlobal($response);
```
