### RagnarSDK 埋点库

### 简介
> * 系统逐渐复杂后，多会依赖多个接口进行工作。但是由于网络等原因线上出现故障不能很快发现、定位、排查，依赖越多系统的调试愈发困难。
> * 支持在线调试，在线调整分级日志级别，错误发现及警报，性能统计跟踪，接口依赖关系等功能。
> * 后续项目都会使用此SDK，之前Ragnar版本将不再提交新特性仅维护

### 安装及植入说明
PHP5.5以上版本方可使用，并安装bcmath扩展

#### Nginx配置

将此项目内的weiboad_fastcgi_pararms拷贝到nginx的conf目录内，在nginx配置内include weiboad_fastcgi_pararms   
 
配置例如：

```
    server{
        listen 80;
        charset utf-8;
        root /path/xxx/xxx/src/public;
        server_name xxx.com;
        
        location /{
            index index.php index.html index.htm;
            if (-f $request_filename) {
                break;
            }
            if (-d $request_filename) {
                break;
            }
            if ($request_filename !~ (\.css|images|index\.php.*) ) {
                rewrite ^/(.*)$ /index.php/$1 last;
                break;
            }
        }
    
        location ~ /index.php/ {
            fastcgi_index index.php;
            fastcgi_pass 127.0.0.1:9000;
            include fastcgi_params;
            include weiboad_fastcgi_params; # 这里
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_read_timeout 600;
        }
    
        location ~ \.php$ {
            fastcgi_index index.php;
            fastcgi_pass 127.0.0.1:9000;
            include fastcgi_params;
            include weiboad_fastcgi_params; # 这里
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_read_timeout 600;
        }
    }
```

配置好后需要reload加载此配置

PHP入口加入：
```
  requrire_once("ragnarsdk/src/MidTool.php");
  requrire_once("ragnarsdk/src/RagnarConst.php");
  requrire_once("ragnarsdk/src/RagnarSDK.php");
  requrire_once("ragnarsdk/src/Traceid.php");
  requrire_once("ragnarsdk/src/Util.php");

```

即可使用，具体使用方法请参考script内demo.php及README


#### Ragnar 埋点库植入说明

在项目框架初始化入口初始化Ragnar

```
    //若需要临时禁用Ragnar可以取消下面一行代码注释
    //\Adinf\RagnarSDK\RagnarSDK::disable();
    
    //是否开启xhprof日志统计，如果开启会影响性能，开发时可以使用
    //参数为指定超过多长时间才记录xhprof日志，若为0则全量记录
    \Adinf\RagnarSDK\RagnarSDK::startXhprof(0);
    
    //是否使用随机方式启动xhprof
    //\Adinf\RagnarSDK\RagnarSDK::startRandXhprof(1000) //随机 千分之一机会开启xhprof
        
    //默认开启info日志级别,低于此级别的日志不会被记录,建议将此日志集成到框架内分级日志内
    \Adinf\RagnarSDK\RagnarSDK::setLogLevel(\Adinf\RagnarSDK\RagnarConst::LOG_TYPE_INFO); 
    
    //这个函数一定要在所有shutdown之后执行，否则会少记录一些内容
    //ragnar_projectname为日志输出子路径目录名称，每个项目建议设置一个独立的名称
    \Adinf\RagnarSDK\RagnarSDK::init("ragnar_projectname");
     
    //设置要索引的日志附加数据，在搜索内能看到，不建议加太多
    //\Adinf\RagnarSDK\RagnarSDK::setMeta(123, "", array("extrakey" => "extraval"));
    
    //Ragnar 分级日志写入示范
    \Adinf\RagnarSDK\RagnarSDK::RecordLog(\Adinf\RagnarSDK\RagnarConst::LOG_TYPE_INFO, __FILE__, __LINE__, "module1_msg", "i wish i can fly!");
    \Adinf\RagnarSDK\RagnarSDK::RecordLog(\Adinf\RagnarSDK\RagnarConst::LOG_TYPE_INFO, __FILE__, __LINE__, "module2_msg", "i wish i'm rich!");
    
    //Ragnar 性能日志手动性能埋点示范  ragnar_test 建议格式 curl mysql 等 （curl mysql在下面已经定义了格式，请参考如下使用）
    $digpooint = \Adinf\RagnarSDK\RagnarSDK::digLogStart(__FILE__,__LINE__,"ragnar_test");
    
        //run something.....
    
    \Adinf\RagnarSDK\RagnarSDK::digLogEnd($digpooint,array("happy"=>1));
    

```

### Ragnar 日志功能介绍
> * 日志分级：在项目中直接使用Ragnar业务日志功能支持在线调试，可以通过密文更改本次请求接口的业务日志级别。
> * 日志索引：所有符合业务级别的日志都会被记录到系统内，通过本次接口的Traceid即可查看到所有此次接口相关调用及级别相关日志输出信息。
> * 错误警报：一些在程序运行过程中产生错误，可以通过设置日志级别发送错误信息到管理平台，平台提供“去重”后的错误信息及警报邮件短信功能。
> * 性能日志：RagnarSDK自带性能埋点函数(digstart/digend)，通过埋点函数可以针对一些关键功能点的运行耗时进行监控，平台会对监控接口进行汇总统计

#### 日志类型及级别介绍
> * LOG_TYPE_TRACE 最低级别，用来输出变量内容，仅用于调试开发或者在线调试
> * LOG_TYPE_DEBUG Debug日志，辅助debug信息
> * LOG_TYPE_NOTICE 警告通知类日志信息
> * LOG_TYPE_INFO  可阅读的一些提示信息
> * LOG_TYPE_ERROR 系统错误信息日志，此日志信息一旦产生会在平台错误统计内去重列出
> * LOG_TYPE_EMEGENCY 系统警报信息日志，此日志信息一旦产生会在平台错误统计内去重列出，并发送邮件短信通知
> * LOG_TYPE_EXCEPTION 系统异常信息日志，此日志信息一旦产生会在平台错误统计内去重列出

> * LOG_TYPE_XHPROF PHP性能日志插件产生的日志，分级日志写入功能不会输出此类日志，只有Ragnar自行产生
> * LOG_TYPE_PERFORMENCE 所有性能埋点函数产生的日志信息，使用的时候请使用示范的标准格式，否则平台渲染会出现问题


#### Curl 埋点建议
curl埋点建议，key请沿用否则会在ragnar展示有问题，如果按照这个方式埋点，可以记录curl请求时间，结果，以及调用链关系（如果对方api也装了这个埋点库的话能看到调用链）

```
    //curl字符串不要改
    $digpooint = \Adinf\RagnarSDK\RagnarSDK::digLogStart(__FILE__, __LINE__, "curl");
    
    //curl init 略 
    
    $nextrpcidheader = \Adinf\RagnarSDK\RagnarSDK::getCurlChildCallParam($digpooint);//这里很关键
    curl_setopt($this->ch, CURLOPT_HTTPHEADER, $nextrpcidheader);
    
    $result = //curl exec 略
    
    $ext = array("errorno" => $errno, "error" => curl_error($this->ch));
    $info = curl_getinfo($this->ch);
    
    
    \Adinf\RagnarSDK\RagnarSDK::digLogEnd($digpooint, array(
                "url" => $info['url'], "method" => self::get_method(),
                "param" => array("post" => $this->post_fields, "get" => $this->query_fields),
                "info" => $info,
                "error" => $ext,
                "result" => $result,//长度请勿超过20480,超过请切断
    );

```

#### Mysql 埋点 
Mysql埋点建议，请沿用key及常量字段，按以下方式埋点后可以记录每个sql的执行情况及性能，并且能够清晰记录异常
```
    //这个放在query操作上出现exception情况，用于记录异常信息
    \Adinf\RagnarSDK\RagnarSDK::RecordLog(\Adinf\Ragnar\Ragnar::LOG_TYPE_EXCEPTION, __FILE__, __LINE__, "mysql", array("fun" => "query", "sql" => $sql, "error" => $ex->getMessage()));
    
    //这个放在查询前,用于性能监控
    $digpooint = \Adinf\RagnarSDK\RagnarSDK::digLogStart(__FILE__, __LINE__, "mysql");
    
    //do some sql execute
    
    //这个放在查询后,用于性能监控
    \Adinf\RagnarSDK\RagnarSDK::digLogEnd($digpooint, array("sql" => $sql, "data" => "sql的参数", "op" => "select\delete\update\...", "fun" => "execute_sql"));
    
    //如果查询后错误
    if(error){
        \Adinf\RagnarSDK\RagnarSDK::RecordLog(\Adinf\RagnarSDK\RagnarSDK::LOG_TYPE_EXCEPTION, __FILE__, __LINE__, "mysql", array("fun" => "execute", "sql" => $sql, "error" => $error));
    }
```

### 在线调试
在请求埋点RagnarSDK的接口时，Header头内带上以下Header可以产生对应的效果，于此同时，接口返回的http请求会带Header如下

```
    X-RAGNAR-TRACEID   此次请求的唯一标识uuid 不建议请求时带这个
    X-RAGNAR-RPCID     此次请求的调用层级计数器
    X-RAGNAR-LOGLEVEL  此次要记录的日志等级，可以实时更改此次请求输出的日志级别
    X-RAGNAR-XHPROF    此次是否开启xhprof记录，若为1则开启记录，所有依赖的埋点Ragnar接口也会同时记录
    X-RAGNAR-XHPROFTIME 可选：此次运行超过多少秒记录xhprof日志，如果超过这个秒数
```
