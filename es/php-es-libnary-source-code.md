## php Elasticsearch类库源码分析记录

git仓库地址： https://github.com/elastic/elasticsearch-php

1. 首先查看文档，正常请求方式如下：

```
use Elasticsearch\ClientBuilder;

require 'vendor/autoload.php';

$client = ClientBuilder::create()->build();
```

我们看到首先调用`ClientBuilder`类， 因此我们以该类为起点，切入代码分析

`ClientBuilder`该类的属性值:

```
 /** @var Transport */
    private $transport; //定义网络传输类

    /** @var callback */
    private $endpoint; //定义操作类型， 比如是query, info 还是Explain 

    /** @var NamespaceBuilderInterface[] */
    private $registeredNamespacesBuilders = []; //注册操作类型所属

    /** @var  ConnectionFactoryInterface */
    private $connectionFactory;  //网络连接工厂， 主要用于生成transport类

    private $handler; //定义curl请求句柄

    /** @var  LoggerInterface */
    private $logger; //定义日志类

    /** @var  LoggerInterface */
    private $tracer; //定义日志类

    /** @var string */
    private $connectionPool = '\Elasticsearch\ConnectionPool\StaticNoPingConnectionPool'; //连接池类

    /** @var  string */
    private $serializer = '\Elasticsearch\Serializers\SmartSerializer'; //序列化类

    /** @var  string */
    private $selector = '\Elasticsearch\ConnectionPool\Selectors\RoundRobinSelector'; ／／连接选择器类，比如有多个host, 定义切换分方法

    /** @var  array */
    private $connectionPoolArgs = [
        'randomizeHosts' => true // 连接池随机切换host, 默认打开
    ];

    /** @var array */
    private $hosts; //host列表

    /** @var array */
    private $connectionParams; //连接参数

    /** @var  int */
    private $retries; //重试参数

    /** @var bool */
    private $sniffOnStart = false; 调试参数

    /** @var null|array  */
    private $sslCert = null; // https模式

    /** @var null|array  */
    private $sslKey = null; // https模式

    /** @var null|bool|string */
    private $sslVerification = null; //https模式

    /** @var bool  */
    private $allowBadJSON = false; //json使用格式
```

对属性列表总结：
主要 定义网络连接， 日志类

然后我们来看build 方法

```
/**
     * @return Client
     */
    public function build()
    {
        if(!defined('JSON_PRESERVE_ZERO_FRACTION') && $this->allowBadJSON === false) {
            throw new RuntimeException("Your version of PHP / json-ext does not support the constant 'JSON_PRESERVE_ZERO_FRACTION',".
            " which is important for proper type mapping in Elasticsearch. Please upgrade your PHP or json-ext.\n".
            "If you are unable to upgrade, and are willing to accept the consequences, you may use the allowBadJSONSerialization()".
            " method on the ClientBuilder to bypass this limitation.");
        }

        $this->buildLoggers();

        if (is_null($this->handler)) {
            $this->handler = ClientBuilder::defaultHandler();
        }

        $sslOptions = null;
        if (isset($this->sslKey)) {
            $sslOptions['ssl_key'] = $this->sslKey;
        }
        if (isset($this->sslCert)) {
            $sslOptions['cert'] = $this->sslCert;
        }
        if (isset($this->sslVerification)) {
            $sslOptions['verify'] = $this->sslVerification;
        }

        if (!is_null($sslOptions)) {
            $sslHandler = function (callable $handler, array $sslOptions) {
                return function (array $request) use ($handler, $sslOptions) {
                    // Add our custom headers
                    foreach ($sslOptions as $key => $value) {
                        $request['client'][$key] = $value;
                    }

                    // Send the request using the handler and return the response.
                    return $handler($request);
                };
            };
            $this->handler = $sslHandler($this->handler, $sslOptions);
        }

        if (is_null($this->serializer)) {
            $this->serializer = new SmartSerializer();
        } elseif (is_string($this->serializer)) {
            $this->serializer = new $this->serializer;
        }

        if (is_null($this->connectionFactory)) {
            if (is_null($this->connectionParams)) {
                $this->connectionParams = [];
            }

            // Make sure we are setting Content-Type and Accept (unless the user has explicitly
            // overridden it
            if (isset($this->connectionParams['client']['headers']) === false) {
                $this->connectionParams['client']['headers'] = [
                    'Content-Type' => ['application/json'],
                    'Accept' => ['application/json']
                ];
            } else {
                if (isset($this->connectionParams['client']['headers']['Content-Type']) === false) {
                    $this->connectionParams['client']['headers']['Content-Type'] = ['application/json'];
                }
                if (isset($this->connectionParams['client']['headers']['Accept']) === false) {
                    $this->connectionParams['client']['headers']['Accept'] = ['application/json'];
                }
            }

            $this->connectionFactory = new ConnectionFactory($this->handler, $this->connectionParams, $this->serializer, $this->logger, $this->tracer);
        }

        if (is_null($this->hosts)) {
            $this->hosts = $this->getDefaultHost();
        }

        if (is_null($this->selector)) {
            $this->selector = new Selectors\RoundRobinSelector();
        } elseif (is_string($this->selector)) {
            $this->selector = new $this->selector;
        }

        $this->buildTransport();

        if (is_null($this->endpoint)) {
            $serializer = $this->serializer;

            $this->endpoint = function ($class) use ($serializer) {
                $fullPath = '\\Elasticsearch\\Endpoints\\' . $class;
                if ($class === 'Bulk' || $class === 'Msearch' || $class === 'MsearchTemplate' || $class === 'MPercolate') {
                    return new $fullPath($serializer);
                } else {
                    return new $fullPath();
                }
            };
        }

        $registeredNamespaces = [];
        foreach ($this->registeredNamespacesBuilders as $builder) {
            /** @var $builder NamespaceBuilderInterface */
            $registeredNamespaces[$builder->getName()] = $builder->getObject($this->transport, $this->serializer);
        }

        return $this->instantiate($this->transport, $this->endpoint, $registeredNamespaces);
    }
```

还是初始化各种类对象， 最后一句是构建client对象， 我们在看下instantiate方法

```
/**
     * @param Transport $transport
     * @param callable $endpoint
     * @param Object[] $registeredNamespaces
     * @return Client
     */
    protected function instantiate(Transport $transport, callable $endpoint, array $registeredNamespaces)
    {
        return new Client($transport, $endpoint, $registeredNamespaces);
    }
```

最后我们看到 生成最终的client 对象， 将transport, endpoint, registeredNamespaces传给client
, 下面我们通过调用client 对象， 来完成各种操作。

可以理解为 client对象为 该类库的代理者， 各种操作必须经过client对象， 类似一些PHP框架中的index.php入口文件

主要依赖关系

```
client -----> 生成client对象， 以下是参数列表
	1. transport ------>网络连接相关， 下面是参数列表
	  a. $this->retries ---->变量
	  b. $this->sniffOnStart --->变量
	  c. $this->connectionPool --->连接池对象
	    1. connections ---> host连接对象数组, 多个host,则多个连接对象
	      connectionFactory::create---->Connection对象
	        1. handler ---->来自工厂
	        2. hostDetails ----->这里额外传的 host信息
	        3. connectionParams --->来自工厂
	        4. serializer ---->来自工厂
	        5. logger -------> 来自工厂
	        6. tracer -------> 来自工厂
	    2. selector ------>默认生成RoundRobinSelector对象
	    3. connectionFactory ---------->connection工厂方法， 主要为了生成Connection对象
	      a. handler ---->curl handler句柄
	      b. connectionParams ----> 主要设置header信息， 比如
	      c. serializer ------>序列化对象， 对象-->json
	      d. logger ------> 输出日志相关 依赖psr/log库
	      e. tracer ------> 也是日志相关， 我理解和logger用途类似
	    4. connectionPoolArgs -----> 连接池参数列表 目前只有 'randomizeHosts' 参数
	  d. $this->logger -----> 日志相关
	2. endpoint -------->主要是将ES操作结构对象化， 比如get/query/
	   构造工厂函数 '\\Elasticsearch\\Endpoints\\' . $class
	3. registeredNamespaces
		对es操作分类， 比如node， cluster等
```

我们再来分析下transport类

```
class Transport
{
    /**
     * @var AbstractConnectionPool
     */
    public $connectionPool;

    /**
     * @var LoggerInterface
     */
    private $log;

    /** @var  int */
    public $retryAttempts = 0;

    /** @var  Connection */
    public $lastConnection;

    /** @var int  */
    public $retries;

    /**
     * Transport class is responsible for dispatching requests to the
     * underlying cluster connections
     *
     * @param $retries
     * @param bool $sniffOnStart
     * @param ConnectionPool\AbstractConnectionPool $connectionPool
     * @param \Psr\Log\LoggerInterface $log    Monolog logger object
     * 构造函数，初始化
     */
    public function __construct($retries, $sniffOnStart = false, AbstractConnectionPool $connectionPool, LoggerInterface $log)
    {
        $this->log            = $log;
        $this->connectionPool = $connectionPool;
        $this->retries        = $retries;

        if ($sniffOnStart === true) {
            $this->log->notice('Sniff on Start.');
            $this->connectionPool->scheduleCheck();
        }
    }

    /**
     * Returns a single connection from the connection pool
     * Potentially performs a sniffing step before returning
     *
     * @return ConnectionInterface Connection
     * 获取连接对象
     */

    public function getConnection()
    {
        return $this->connectionPool->nextConnection();
    }

    /**
     * Perform a request to the Cluster
     * 
     * @param string $method     HTTP method to use
     * @param string $uri        HTTP URI to send request to
     * @param null $params     Optional query parameters
     * @param null $body       Optional query body
     * @param array $options
     *
     * @throws Common\Exceptions\NoNodesAvailableException|\Exception
     * @return FutureArrayInterface
     * 执行http请求
     */
    public function performRequest($method, $uri, $params = null, $body = null, $options = [])
    {
        try {
            $connection  = $this->getConnection();
        } catch (Exceptions\NoNodesAvailableException $exception) {
            $this->log->critical('No alive nodes found in cluster');
            throw $exception;
        }

        $response             = array();
        $caughtException      = null;
        $this->lastConnection = $connection;

        $future = $connection->performRequest(
            $method,
            $uri,
            $params,
            $body,
            $options,
            $this
        );

        $future->promise()->then(
            //onSuccess
            function ($response) {
                $this->retryAttempts = 0;
                // Note, this could be a 4xx or 5xx error
            },
            //onFailure
            function (\Exception $response) {
                // Ignore 400 level errors, as that means the server responded just fine
                $code = $response->getCode();
                if (!(isset($code) && $code >=400 && $code < 500)) {
                    // Otherwise schedule a check
                    $this->connectionPool->scheduleCheck();
                }
            });

        return $future;
    }

    /**
     * @param FutureArrayInterface $result  Response of a request (promise)
     * @param array                $options Options for transport
     *
     * @return callable|array
     * 返回结果 或者返回future对象
     */
    public function resultOrFuture($result, $options = [])
    {
        $response = null;
        $async = isset($options['client']['future']) ? $options['client']['future'] : null;
        if (is_null($async) || $async === false) {
            do {
                $result = $result->wait();
            } while ($result instanceof FutureArrayInterface);

            return $result;
        } elseif ($async === true || $async === 'lazy') {
            return $result;
        }
    }

    /**
     * @param $request
     *
     * @return bool
     * 校验是否重试 //todo 命名是isShouldRetry比较好
     */
    public function shouldRetry($request)
    {
        if ($this->retryAttempts < $this->retries) {
            $this->retryAttempts += 1;

            return true;
        }

        return false;
    }

    /**
     * Returns the last used connection so that it may be inspected.  Mainly
     * for debugging/testing purposes.
     *
     * @return Connection
     * 获取最后一次连接对象
     */
    public function getLastConnection()
    {
        return $this->lastConnection;
    }
}
```
我们可以看到 Transport 类，有主要以下功能

1. 重试次数，尝试重试次数限制
2. 通过操作connectionPool对象来执行connection对象

下面我们看下 `connectionPool`类

```
class StaticNoPingConnectionPool extends AbstractConnectionPool implements ConnectionPoolInterface
{
    /**
     * @var int
     * ping时间
     */
    private $pingTimeout    = 60;

    /**
     * @var int
     * 最大ping时间
     */
    private $maxPingTimeout = 3600;

    /**
     * {@inheritdoc}
     */
    public function __construct($connections, SelectorInterface $selector, ConnectionFactoryInterface $factory, $connectionPoolParams)
    {
        parent::__construct($connections, $selector, $factory, $connectionPoolParams);
    }

    /**
     * @param bool $force
     *
     * @return Connection
     * @throws \Elasticsearch\Common\Exceptions\NoNodesAvailableException
     * //获取连接实例
     */
    public function nextConnection($force = false)
    {
        $total = count($this->connections);
        while ($total--) {
            /** @var Connection $connection */
            $connection = $this->selector->select($this->connections);
            if ($connection->isAlive() === true) {
                return $connection;
            }

            if ($this->readyToRevive($connection) === true) {
                return $connection;
            }
        }

        throw new NoNodesAvailableException("No alive nodes found in your cluster");
    }

    public function scheduleCheck()
    {
    }

    /**
     * @param \Elasticsearch\Connections\Connection $connection
     *
     * @return bool
     * //是否准备接收
     */
    private function readyToRevive(Connection $connection)
    {
        $timeout = min(
            $this->pingTimeout * pow(2, $connection->getPingFailures()),
            $this->maxPingTimeout
        );

        if ($connection->getLastPing() + $timeout < time()) {
            return true;
        } else {
            return false;
        }
    }
}
```
`connectionPool`类主要职责：

1. 通过selector获取下一个对象 (我的常规思维一般是通过一个方法获取)
2. 返回connection对象


分析到这里， 我们发现 `Transport` `connectionPool `类，都没有进行网络请求， 我理解是做一些代理相关事情，单一指责， 真正处理网络请求，是在connection类中， 下面我们分析`connection`类

```
class Connection implements ConnectionInterface
{
    /** @var  callable */
    //curl句柄对象guzzp
    protected $handler;

    /** @var SerializerInterface */
    //序列化类对象
    protected $serializer;

    /**
     * @var string
     */
    //传输协议
    protected $transportSchema = 'http';    // TODO depreciate this default

    /**
     * @var string
     */
    // host地址
    protected $host;

    /**
     * @var string || null
     */
    //路径地址
    protected $path;

    /**
     * @var LoggerInterface
     */
    //log日志
    protected $log;

    /**
     * @var LoggerInterface
     */
    //log 日志相关
    protected $trace;

    /**
     * @var array
     */
    //connectionParams参数相关
    protected $connectionParams;

    /** @var  array */
    //头部信息
    protected $headers = [];

    /** @var bool  */
    protected $isAlive = false;

    /** @var float  */
    private $pingTimeout = 1;    //TODO expose this

    /** @var int  */
    private $lastPing = 0;

    /** @var int  */
    private $failedPings = 0;

    private $lastRequest = array();

    /**
     * Constructor
     *
     * @param $handler
     * @param array $hostDetails
     * @param array $connectionParams Array of connection-specific parameters
     * @param \Elasticsearch\Serializers\SerializerInterface $serializer
     * @param \Psr\Log\LoggerInterface $log              Logger object
     * @param \Psr\Log\LoggerInterface $trace
     */
    public function __construct($handler, $hostDetails, $connectionParams,
                                SerializerInterface $serializer, LoggerInterface $log, LoggerInterface $trace)
    {
        //端口号
        if (isset($hostDetails['port']) !== true) {
            $hostDetails['port'] = 9200;
        }
        //初始化协议
        if (isset($hostDetails['scheme'])) {
            $this->transportSchema = $hostDetails['scheme'];
        }
		 //基础验证相关
        if (isset($hostDetails['user']) && isset($hostDetails['pass'])) {
            $connectionParams['client']['curl'][CURLOPT_HTTPAUTH] = CURLAUTH_BASIC;
            $connectionParams['client']['curl'][CURLOPT_USERPWD] = $hostDetails['user'].':'.$hostDetails['pass'];
        }

        if (isset($connectionParams['client']['headers']) === true) {
            $this->headers = $connectionParams['client']['headers'];
            unset($connectionParams['client']['headers']);
        }

        $host = $hostDetails['host'].':'.$hostDetails['port'];
        $path = null;
        if (isset($hostDetails['path']) === true) {
            $path = $hostDetails['path'];
        }
        $this->host             = $host;
        $this->path             = $path;
        $this->log              = $log;
        $this->trace            = $trace;
        $this->connectionParams = $connectionParams;
        $this->serializer       = $serializer;

        $this->handler = $this->wrapHandler($handler, $log, $trace);
    }

    /**
     * @param $method
     * @param $uri
     * @param null $params
     * @param null $body
     * @param array $options
     * @param \Elasticsearch\Transport $transport
     * @return mixed
     * //执行最终请求
     */
    public function performRequest($method, $uri, $params = null, $body = null, $options = [], Transport $transport = null)
    {
        if (isset($body) === true) {
            $body = $this->serializer->serialize($body);
        }

        $request = [
            'http_method' => $method,
            'scheme'      => $this->transportSchema,
            'uri'         => $this->getURI($uri, $params),
            'body'        => $body,
            'headers'     => array_merge([
                'Host'  => [$this->host]
            ], $this->headers)
        ];

        $request = array_replace_recursive($request, $this->connectionParams, $options);

        // RingPHP does not like if client is empty
        if (empty($request['client'])) {
            unset($request['client']);
        }

        $handler = $this->handler;
        $future = $handler($request, $this, $transport, $options);

        return $future;
    }

    /** @return string */
    public function getTransportSchema()
    {
        return $this->transportSchema;
    }

    /** @return array */
    public function getLastRequestInfo()
    {
        return $this->lastRequest;
    }

    private function wrapHandler(callable $handler, LoggerInterface $logger, LoggerInterface $tracer)
    {
        return function (array $request, Connection $connection, Transport $transport = null, $options) use ($handler, $logger, $tracer) {

            $this->lastRequest = [];
            $this->lastRequest['request'] = $request;

            // Send the request using the wrapped handler.
            $response =  Core::proxy($handler($request), function ($response) use ($connection, $transport, $logger, $tracer, $request, $options) {

                $this->lastRequest['response'] = $response;

                if (isset($response['error']) === true) {
                    if ($response['error'] instanceof ConnectException || $response['error'] instanceof RingException) {
                        $this->log->warning("Curl exception encountered.");

                        $exception = $this->getCurlRetryException($request, $response);

                        $this->logRequestFail(
                            $request['http_method'],
                            $response['effective_url'],
                            $request['body'],
                            $request['headers'],
                            $response['status'],
                            $response['body'],
                            $response['transfer_stats']['total_time'],
                            $exception
                        );

                        $node = $connection->getHost();
                        $this->log->warning("Marking node $node dead.");
                        $connection->markDead();

                        // If the transport has not been set, we are inside a Ping or Sniff,
                        // so we don't want to retrigger retries anyway.
                        //
                        // TODO this could be handled better, but we are limited because connectionpools do not
                        // have access to Transport.  Architecturally, all of this needs to be refactored
                        if (isset($transport) === true) {
                            $transport->connectionPool->scheduleCheck();

                            $neverRetry = isset($request['client']['never_retry']) ? $request['client']['never_retry'] : false;
                            $shouldRetry = $transport->shouldRetry($request);
                            $shouldRetryText = ($shouldRetry) ? 'true' : 'false';

                            $this->log->warning("Retries left? $shouldRetryText");
                            if ($shouldRetry && !$neverRetry) {
                                return $transport->performRequest(
                                    $request['http_method'],
                                    $request['uri'],
                                    [],
                                    $request['body'],
                                    $options
                                );
                            }
                        }

                        $this->log->warning("Out of retries, throwing exception from $node");
                        // Only throw if we run out of retries
                        throw $exception;
                    } else {
                        // Something went seriously wrong, bail
                        $exception = new TransportException($response['error']->getMessage());
                        $this->logRequestFail(
                            $request['http_method'],
                            $response['effective_url'],
                            $request['body'],
                            $request['headers'],
                            $response['status'],
                            $response['body'],
                            $response['transfer_stats']['total_time'],
                            $exception
                        );
                        throw $exception;
                    }
                } else {
                    $connection->markAlive();

                    if (isset($response['body']) === true) {
                        $response['body'] = stream_get_contents($response['body']);
                        $this->lastRequest['response']['body'] = $response['body'];
                    }

                    if ($response['status'] >= 400 && $response['status'] < 500) {
                        $ignore = isset($request['client']['ignore']) ? $request['client']['ignore'] : [];
                        $this->process4xxError($request, $response, $ignore);
                    } elseif ($response['status'] >= 500) {
                        $ignore = isset($request['client']['ignore']) ? $request['client']['ignore'] : [];
                        $this->process5xxError($request, $response, $ignore);
                    }

                    // No error, deserialize
                    $response['body'] = $this->serializer->deserialize($response['body'], $response['transfer_stats']);
                }
                $this->logRequestSuccess(
                    $request['http_method'],
                    $response['effective_url'],
                    $request['body'],
                    $request['headers'],
                    $response['status'],
                    $response['body'],
                    $response['transfer_stats']['total_time']
                );

                return isset($request['client']['verbose']) && $request['client']['verbose'] === true ? $response : $response['body'];

            });

            return $response;
        };
    }

    /**
     * @param string $uri
     * @param array $params
     *
     * @return string
     */
    private function getURI($uri, $params)
    {
        if (isset($params) === true && !empty($params)) {
            array_walk($params, function (&$value, &$key) {
                if ($value === true) {
                    $value = 'true';
                } else if ($value === false) {
                    $value = 'false';
                }
            });

            $uri .= '?' . http_build_query($params);
        }

        if ($this->path !== null) {
            $uri = $this->path . $uri;
        }

        return $uri;
    }

    /**
     * Log a successful request
     *
     * @param string $method
     * @param string $fullURI
     * @param string $body
     * @param array  $headers
     * @param string $statusCode
     * @param string $response
     * @param string $duration
     *
     * @return void
     */
    public function logRequestSuccess($method, $fullURI, $body, $headers, $statusCode, $response, $duration)
    {
        $this->log->debug('Request Body', array($body));
        $this->log->info(
            'Request Success:',
            array(
                'method'    => $method,
                'uri'       => $fullURI,
                'headers'   => $headers,
                'HTTP code' => $statusCode,
                'duration'  => $duration,
            )
        );
        $this->log->debug('Response', array($response));

        // Build the curl command for Trace.
        $curlCommand = $this->buildCurlCommand($method, $fullURI, $body);
        $this->trace->info($curlCommand);
        $this->trace->debug(
            'Response:',
            array(
                'response'  => $response,
                'method'    => $method,
                'uri'       => $fullURI,
                'HTTP code' => $statusCode,
                'duration'  => $duration,
            )
        );
    }

    /**
     * Log a a failed request
     *
     * @param string $method
     * @param string $fullURI
     * @param string $body
     * @param array $headers
     * @param null|string $statusCode
     * @param null|string $response
     * @param string $duration
     * @param \Exception|null $exception
     *
     * @return void
     */
    public function logRequestFail($method, $fullURI, $body, $headers, $statusCode, $response, $duration, \Exception $exception)
    {
        $this->log->debug('Request Body', array($body));
        $this->log->warning(
            'Request Failure:',
            array(
                'method'    => $method,
                'uri'       => $fullURI,
                'headers'   => $headers,
                'HTTP code' => $statusCode,
                'duration'  => $duration,
                'error'     => $exception->getMessage(),
            )
        );
        $this->log->warning('Response', array($response));

        // Build the curl command for Trace.
        $curlCommand = $this->buildCurlCommand($method, $fullURI, $body);
        $this->trace->info($curlCommand);
        $this->trace->debug(
            'Response:',
            array(
                'response'  => $response,
                'method'    => $method,
                'uri'       => $fullURI,
                'HTTP code' => $statusCode,
                'duration'  => $duration,
            )
        );
    }

    /**
     * @return bool
     */
    public function ping()
    {
        $options = [
            'client' => [
                'timeout' => $this->pingTimeout,
                'never_retry' => true,
                'verbose' => true
            ]
        ];
        try {
            $response = $this->performRequest('HEAD', '/', null, null, $options);
            $response = $response->wait();
        } catch (TransportException $exception) {
            $this->markDead();

            return false;
        }

        if ($response['status'] === 200) {
            $this->markAlive();

            return true;
        } else {
            $this->markDead();

            return false;
        }
    }

    /**
     * @return array
     */
    public function sniff()
    {
        $options = [
            'client' => [
                'timeout' => $this->pingTimeout,
                'never_retry' => true
            ]
        ];

        return $this->performRequest('GET', '/_nodes/', null, null, $options);
    }

    /**
     * @return bool
     */
    public function isAlive()
    {
        return $this->isAlive;
    }

    public function markAlive()
    {
        $this->failedPings = 0;
        $this->isAlive = true;
        $this->lastPing = time();
    }

    public function markDead()
    {
        $this->isAlive = false;
        $this->failedPings += 1;
        $this->lastPing = time();
    }

    /**
     * @return int
     */
    public function getLastPing()
    {
        return $this->lastPing;
    }

    /**
     * @return int
     */
    public function getPingFailures()
    {
        return $this->failedPings;
    }

    /**
     * @return string
     */
    public function getHost()
    {
        return $this->host;
    }

    /**
     * @return null|string
     */
    public function getUserPass()
    {
        if (isset($this->connectionParams['client']['curl'][CURLOPT_USERPWD]) === true) {
            return $this->connectionParams['client']['curl'][CURLOPT_USERPWD];
        }
        return null;
    }

    /**
     * @return null|string
     */
    public function getPath()
    {
        return $this->path;
    }

    /**
     * @param $request
     * @param $response
     * @return \Elasticsearch\Common\Exceptions\Curl\CouldNotConnectToHost|\Elasticsearch\Common\Exceptions\Curl\CouldNotResolveHostException|\Elasticsearch\Common\Exceptions\Curl\OperationTimeoutException|\Elasticsearch\Common\Exceptions\MaxRetriesException
     */
    protected function getCurlRetryException($request, $response)
    {
        $exception = null;
        $message = $response['error']->getMessage();
        $exception = new MaxRetriesException($message);
        switch ($response['curl']['errno']) {
            case 6:
                $exception = new CouldNotResolveHostException($message, null, $exception);
                break;
            case 7:
                $exception = new CouldNotConnectToHost($message, null, $exception);
                break;
            case 28:
                $exception = new OperationTimeoutException($message, null, $exception);
                break;
        }

        return $exception;
    }

    /**
     * Construct a string cURL command
     *
     * @param string $method HTTP method
     * @param string $uri    Full URI of request
     * @param string $body   Request body
     *
     * @return string
     */
    private function buildCurlCommand($method, $uri, $body)
    {
        if (strpos($uri, '?') === false) {
            $uri .= '?pretty=true';
        } else {
            str_replace('?', '?pretty=true', $uri);
        }

        $curlCommand = 'curl -X' . strtoupper($method);
        $curlCommand .= " '" . $uri . "'";

        if (isset($body) === true && $body !== '') {
            $curlCommand .= " -d '" . $body . "'";
        }

        return $curlCommand;
    }

    /**
     * @param $request
     * @param $response
     * @param $ignore
     * @throws \Elasticsearch\Common\Exceptions\AlreadyExpiredException|\Elasticsearch\Common\Exceptions\BadRequest400Exception|\Elasticsearch\Common\Exceptions\Conflict409Exception|\Elasticsearch\Common\Exceptions\Forbidden403Exception|\Elasticsearch\Common\Exceptions\Missing404Exception|\Elasticsearch\Common\Exceptions\ScriptLangNotSupportedException|null
     */
    private function process4xxError($request, $response, $ignore)
    {
        $statusCode = $response['status'];
        $responseBody = $response['body'];

        /** @var \Exception $exception */
        $exception = $this->tryDeserialize400Error($response);

        if (array_search($response['status'], $ignore) !== false) {
            return;
        }

        if ($statusCode === 400 && strpos($responseBody, "AlreadyExpiredException") !== false) {
            $exception = new AlreadyExpiredException($responseBody, $statusCode);
        } elseif ($statusCode === 403) {
            $exception = new Forbidden403Exception($responseBody, $statusCode);
        } elseif ($statusCode === 404) {
            $exception = new Missing404Exception($responseBody, $statusCode);
        } elseif ($statusCode === 409) {
            $exception = new Conflict409Exception($responseBody, $statusCode);
        } elseif ($statusCode === 400 && strpos($responseBody, 'script_lang not supported') !== false) {
            $exception = new ScriptLangNotSupportedException($responseBody. $statusCode);
        } elseif ($statusCode === 408) {
            $exception = new RequestTimeout408Exception($responseBody, $statusCode);
        } else {
            $exception = new BadRequest400Exception($responseBody, $statusCode);
        }

        $this->logRequestFail(
            $request['http_method'],
            $response['effective_url'],
            $request['body'],
            $request['headers'],
            $response['status'],
            $response['body'],
            $response['transfer_stats']['total_time'],
            $exception
        );

        throw $exception;
    }

    /**
     * @param $request
     * @param $response
     * @param $ignore
     * @throws \Elasticsearch\Common\Exceptions\NoDocumentsToGetException|\Elasticsearch\Common\Exceptions\NoShardAvailableException|\Elasticsearch\Common\Exceptions\RoutingMissingException|\Elasticsearch\Common\Exceptions\ServerErrorResponseException
     */
    private function process5xxError($request, $response, $ignore)
    {
        $statusCode = $response['status'];
        $responseBody = $response['body'];

        /** @var \Exception $exception */
        $exception = $this->tryDeserialize500Error($response);

        $exceptionText = "[$statusCode Server Exception] ".$exception->getMessage();
        $this->log->error($exceptionText);
        $this->log->error($exception->getTraceAsString());

        if (array_search($statusCode, $ignore) !== false) {
            return;
        }

        if ($statusCode === 500 && strpos($responseBody, "RoutingMissingException") !== false) {
            $exception = new RoutingMissingException($exception->getMessage(), $statusCode, $exception);
        } elseif ($statusCode === 500 && preg_match('/ActionRequestValidationException.+ no documents to get/', $responseBody) === 1) {
            $exception = new NoDocumentsToGetException($exception->getMessage(), $statusCode, $exception);
        } elseif ($statusCode === 500 && strpos($responseBody, 'NoShardAvailableActionException') !== false) {
            $exception = new NoShardAvailableException($exception->getMessage(), $statusCode, $exception);
        } else {
            $exception = new ServerErrorResponseException($responseBody, $statusCode);
        }

        $this->logRequestFail(
            $request['http_method'],
            $response['effective_url'],
            $request['body'],
            $request['headers'],
            $response['status'],
            $response['body'],
            $response['transfer_stats']['total_time'],
            $exception
        );

        throw $exception;
    }

    private function tryDeserialize400Error($response)
    {
        return $this->tryDeserializeError($response, 'Elasticsearch\Common\Exceptions\BadRequest400Exception');
    }

    private function tryDeserialize500Error($response)
    {
        return $this->tryDeserializeError($response, 'Elasticsearch\Common\Exceptions\ServerErrorResponseException');
    }

    private function tryDeserializeError($response, $errorClass)
    {
        $error = $this->serializer->deserialize($response['body'], $response['transfer_stats']);
        if (is_array($error) === true) {
            // 2.0 structured exceptions
            if (isset($error['error']['reason']) === true) {

                // Try to use root cause first (only grabs the first root cause)
                $root = $error['error']['root_cause'];
                if (isset($root) && isset($root[0])) {
                    $cause = $root[0]['reason'];
                    $type = $root[0]['type'];
                } else {
                    $cause = $error['error']['reason'];
                    $type = $error['error']['type'];
                }

                $original = new $errorClass($response['body'], $response['status']);

                return new $errorClass("$type: $cause", $response['status'], $original);
            } elseif (isset($error['error']) === true) {
                // <2.0 semi-structured exceptions
                $original = new $errorClass($response['body'], $response['status']);

                return new $errorClass($error['error'], $response['status'], $original);
            }

            // <2.0 "i just blew up" nonstructured exception
            // $error is an array but we don't know the format, reuse the response body instead
            return new $errorClass($response['body'], $response['status']);
        }

        // <2.0 "i just blew up" nonstructured exception
        return new $errorClass($response['body']);
    }
}

```

`Connection` 类主要功能

1. 完成最终网络请求



现在分析完 `ClientBuilder`类后， 重点看下`Client`类结构

属性列表如下：

```
/**
     * @var Transport 
     * 网络对象
     */
    public $transport;

    /**
     * @var array
     * 参数数组
     */
    protected $params;

    /**
     * @var IndicesNamespace
     * 索引组空间
     */
    protected $indices;

    /**
     * @var ClusterNamespace
     * 集群组空间
     */
    protected $cluster;

    /**
     * @var NodesNamespace
     * node组空间
     */
    protected $nodes;

    /**
     * @var SnapshotNamespace
     * snaphost组空间
     */
    protected $snapshot;

    /**
     * @var CatNamespace
     * cat组空间
     */
    protected $cat;

    /**
     * @var IngestNamespace
     * ingest组空间
     */
    protected $ingest;

    /**
     * @var TasksNamespace
     * task组空间
     */
    protected $tasks;

    /**
     * @var RemoteNamespace
     * remote组空间
     */
    protected $remote;

    /** @var  callback */
    // endpoint功能点构造函数
    protected $endpoints;

    /** @var  NamespaceBuilderInterface[] */
    //额外注册的命名空间
    protected $registeredNamespaces = [];
```

构造函数初始化

```
/**
     * Client constructor
     *
     * @param Transport $transport
     * @param callable $endpoint
     * @param AbstractNamespace[] $registeredNamespaces
     */
    public function __construct(Transport $transport, callable $endpoint, array $registeredNamespaces)
    {
        $this->transport = $transport;
        $this->endpoints = $endpoint;
        $this->indices   = new IndicesNamespace($transport, $endpoint); //初始化，对外暴露indices所属方法
        $this->cluster   = new ClusterNamespace($transport, $endpoint); //初始化，对外暴露indices所属方法， 下面的初始化都类似

        $this->nodes     = new NodesNamespace($transport, $endpoint);
        $this->snapshot  = new SnapshotNamespace($transport, $endpoint);
        $this->cat       = new CatNamespace($transport, $endpoint);
        $this->ingest    = new IngestNamespace($transport, $endpoint);
        $this->tasks     = new TasksNamespace($transport, $endpoint);
        $this->remote    = new RemoteNamespace($transport, $endpoint);
        $this->registeredNamespaces = $registeredNamespaces;
    }

```

```
/**
     * @param $params
     * @return array
     * 这里是因为不属于各个命名空间， 因此直接构造请求
     */
    public function info($params = [])
    {
        /** @var callback $endpointBuilder */
        $endpointBuilder = $this->endpoints;

        /** @var \Elasticsearch\Endpoints\Info $endpoint */
        $endpoint = $endpointBuilder('Info');
        $endpoint->setParams($params);

        return $this->performRequest($endpoint);
    }
```


我们看下这里的 `performRequest`方法

```
/**
     * @param $endpoint AbstractEndpoint
     * 
     * @throws \Exception
     * @return array
     * 直接通过transport请求 perfromRequest方法
     */
    private function performRequest(AbstractEndpoint $endpoint)
    {
        $promise =  $this->transport->performRequest(
            $endpoint->getMethod(),
            $endpoint->getURI(),
            $endpoint->getParams(),
            $endpoint->getBody(),
            $endpoint->getOptions()
        );

        return $this->transport->resultOrFuture($promise, $endpoint->getOptions());
    }
```

我们在看下 `search`方法

```
public function search($params = array())
    {
        $index = $this->extractArgument($params, 'index');
        $type = $this->extractArgument($params, 'type');
        $body = $this->extractArgument($params, 'body');

        /** @var callback $endpointBuilder */
        $endpointBuilder = $this->endpoints;

        /** @var \Elasticsearch\Endpoints\Search $endpoint */
        $endpoint = $endpointBuilder('Search');
        $endpoint->setIndex($index)
                 ->setType($type)
                 ->setBody($body);
        $endpoint->setParams($params);

        return $this->performRequest($endpoint);
    }
```

这里我们在看下Endpoint下面Search类

```
<?php

namespace Elasticsearch\Endpoints;

use Elasticsearch\Common\Exceptions\InvalidArgumentException;
use Elasticsearch\Common\Exceptions;

/**
 * Class Search
 *
 * @category Elasticsearch
 * @package  Elasticsearch\Endpoints
 * @author   Zachary Tong <zach@elastic.co>
 * @license  http://www.apache.org/licenses/LICENSE-2.0 Apache2
 * @link     http://elastic.co
 */
class Search extends AbstractEndpoint
{
    /**
     * @param array $body
     *
     * @throws \Elasticsearch\Common\Exceptions\InvalidArgumentException
     * @return $this
     */
    public function setBody($body)
    {
        if (isset($body) !== true) {
            return $this;
        }

        $this->body = $body;

        return $this;
    }

    /**
     * @return string
     * 获取URL地址
     */
    public function getURI()
    {
        $index = $this->index;
        $type = $this->type;
        $uri   = "/_search";

        if (isset($index) === true && isset($type) === true) {
            $uri = "/$index/$type/_search";
        } elseif (isset($index) === true) {
            $uri = "/$index/_search";
        } elseif (isset($type) === true) {
            $uri = "/_all/$type/_search";
        }

        return $uri;
    }

    /**
     * @return string[]
     * 白名单过滤 这里主要是setParams时用到
     */
    public function getParamWhitelist()
    {
        return array(
            'analyzer',
            'analyze_wildcard',
            'default_operator',
            'df',
            'explain',
            'from',
            'ignore_unavailable',
            'allow_no_indices',
            'expand_wildcards',
            'indices_boost',
            'lenient',
            'lowercase_expanded_terms',
            'preference',
            'q',
            'query_cache',
            'request_cache',
            'routing',
            'scroll',
            'search_type',
            'size',
            'slice',
            'sort',
            'source',
            '_source',
            '_source_exclude',
            '_source_include',
            'stats',
            'suggest_field',
            'suggest_mode',
            'suggest_size',
            'suggest_text',
            'timeout',
            'version',
            'fielddata_fields',
            'docvalue_fields',
            'filter_path',
            'terminate_after',
            'stored_fields',
            'batched_reduce_size',
            'typed_keys'
        );
    }

    /**
     * @return string
     */
    public function getMethod()
    {
        return 'GET';
    }
}

```
功能总结：
1. 将方法结构化为类对象， 构造对象属性
2. 主要是设置URI, method, body, 以及其他附属属性

我们在看下Namespaces下 类对象, `NodesNamespace`下面主要是node相关操作， 底层还是通过构造`endpoint`, 再进行网络调用。

```
/**
 * Class NodesNamespace
 *
 * @category Elasticsearch
 * @package  Elasticsearch\Namespaces\NodesNamespace
 * @author   Zachary Tong <zach@elastic.co>
 * @license  http://www.apache.org/licenses/LICENSE-2.0 Apache2
 * @link     http://elastic.co
 */
class NodesNamespace extends AbstractNamespace
{
    /**
     * $params['fields']        = (list) A comma-separated list of fields for `fielddata` metric (supports wildcards)
     *        ['metric_family'] = (enum) Limit the information returned to a certain metric family
     *        ['metric']        = (enum) Limit the information returned for `indices` family to a specific metric
     *        ['node_id']       = (list) A comma-separated list of node IDs or names to limit the returned information; use `_local` to return information from the node you're connecting to, leave empty to get information from all nodes
     *        ['all']           = (boolean) Return all available information
     *        ['clear']         = (boolean) Reset the default level of detail
     *        ['fs']            = (boolean) Return information about the filesystem
     *        ['http']          = (boolean) Return information about HTTP
     *        ['indices']       = (boolean) Return information about indices
     *        ['jvm']           = (boolean) Return information about the JVM
     *        ['network']       = (boolean) Return information about network
     *        ['os']            = (boolean) Return information about the operating system
     *        ['process']       = (boolean) Return information about the Elasticsearch process
     *        ['thread_pool']   = (boolean) Return information about the thread pool
     *        ['transport']     = (boolean) Return information about transport
     *
     * @param $params array Associative array of parameters
     *
     * @return array
     */
    public function stats($params = array())
    {
        $nodeID = $this->extractArgument($params, 'node_id');

        $metric = $this->extractArgument($params, 'metric');

        $index_metric = $this->extractArgument($params, 'index_metric');

        /** @var callback $endpointBuilder */
        $endpointBuilder = $this->endpoints;

        /** @var \Elasticsearch\Endpoints\Cluster\Nodes\Stats $endpoint */
        $endpoint = $endpointBuilder('Cluster\Nodes\Stats');
        $endpoint->setNodeID($nodeID)
                 ->setMetric($metric)
                 ->setIndexMetric($index_metric)
                 ->setParams($params);

        return $this->performRequest($endpoint);
    }

    /**
     * $params['node_id']       = (list) A comma-separated list of node IDs or names to limit the returned information; use `_local` to return information from the node you're connecting to, leave empty to get information from all nodes
     *        ['metric']        = (list) A comma-separated list of metrics you wish returned. Leave empty to return all.
     *        ['flat_settings'] = (boolean) Return settings in flat format (default: false)
     *        ['human']         = (boolean) Whether to return time and byte values in human-readable format.

     *
     * @param $params array Associative array of parameters
     *
     * @return array
     * 获取node信息
     */
    public function info($params = array())
    {
        $nodeID = $this->extractArgument($params, 'node_id');
        $metric = $this->extractArgument($params, 'metric');

        /** @var callback $endpointBuilder */
        $endpointBuilder = $this->endpoints;

        /** @var \Elasticsearch\Endpoints\Cluster\Nodes\Info $endpoint */
        $endpoint = $endpointBuilder('Cluster\Nodes\Info');
        $endpoint->setNodeID($nodeID)->setMetric($metric);
        $endpoint->setParams($params);

        return $this->performRequest($endpoint);
    }

    /**
     * $params['node_id']   = (list) A comma-separated list of node IDs or names to limit the returned information; use `_local` to return information from the node you're connecting to, leave empty to get information from all nodes
     *        ['interval']  = (time) The interval for the second sampling of threads
     *        ['snapshots'] = (number) Number of samples of thread stacktrace (default: 10)
     *        ['threads']   = (number) Specify the number of threads to provide information for (default: 3)
     *        ['type']      = (enum) The type to sample (default: cpu)
     *
     * @param $params array Associative array of parameters
     *
     * @return array
     */
    public function hotThreads($params = array())
    {
        $nodeID = $this->extractArgument($params, 'node_id');

        /** @var callback $endpointBuilder */
        $endpointBuilder = $this->endpoints;

        /** @var \Elasticsearch\Endpoints\Cluster\Nodes\HotThreads $endpoint */
        $endpoint = $endpointBuilder('Cluster\Nodes\HotThreads');
        $endpoint->setNodeID($nodeID);
        $endpoint->setParams($params);

        return $this->performRequest($endpoint);
    }

    /**
     * $params['node_id'] = (list) A comma-separated list of node IDs or names to perform the operation on; use `_local` to perform the operation on the node you're connected to, leave empty to perform the operation on all nodes
     *        ['delay']   = (time) Set the delay for the operation (default: 1s)
     *        ['exit']    = (boolean) Exit the JVM as well (default: true)
     *
     * @param $params array Associative array of parameters
     *
     * @return array
     */
    public function shutdown($params = array())
    {
        $nodeID = $this->extractArgument($params, 'node_id');

        /** @var callback $endpointBuilder */
        $endpointBuilder = $this->endpoints;

        /** @var \Elasticsearch\Endpoints\Cluster\Nodes\Shutdown $endpoint */
        $endpoint = $endpointBuilder('Cluster\Nodes\Shutdown');
        $endpoint->setNodeID($nodeID);
        $endpoint->setParams($params);

        return $this->performRequest($endpoint);
    }
}
```



### 总结

1. 类单一指责， 明确类的使用范围
2. 类层次感， 通过client类暴露 库接口, 收拢调用
3. 大部分代码都是重复类似的， 需要抓住重点， 比如Endpoints目录,namespaces目录都是重复代码



### 其他

1. 项目引入Guzzle 实现http调用， 其中使用promise来请求。 配合迁移js的promise 来加深理解
2.  下一步需要分析ES查询类库:https://github.com/ongr-io/ElasticsearchDSL
3. php symfony类库的序列化库：https://symfony.com/doc/current/components/serializer.html




