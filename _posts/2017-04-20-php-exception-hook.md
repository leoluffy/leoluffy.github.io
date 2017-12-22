---
layout: post
category : PHP
title: 'PHP设置全局异常处理钩子'
tagline: ""
tags : [PHP]
---
* auto-gen TOC:
{:toc}

# PHP设置全局异常处理钩子
* 最近需要上报项目里面的异常日志，方面定位和分析问题
* 翻阅相关资料后，决定采用设置全局异常处理钩子
* 业务层将需要上报的异常日志使用抛出异常的方式，便可以被捕捉到，并上报到日志搜集服务

<!--break-->

```
//设置全局异常处理钩子
set_exception_handler ( 'customException' );

//初始化日志上报配置
use MingChao\McWeb\AppLog;
AppLog::getInstance()->setAppId(METL_APP_ID);
AppLog::getInstance()->setRelatedAppId(RELATED_APP_ID);
AppLog::getInstance()->setSubApp(SUB_APP);
AppLog::getInstance()->setAppKey(METL_APP_KEY);
AppLog::getInstance()->setMetlVersion(METL_VERSION);
AppLog::getInstance()->setLogType(ERROR_LOG_TYPE);
AppLog::getInstance()->setMetlService(METL_LOG_URL);
AppLog::getInstance()->setLogSwitch(METL_LOG_SWITCH);

//自定义异常处理钩子
function customException($exception){
    do {
        $summary = $exception->getMessage();
        $trace = $exception->getTraceAsString();
        $file = $exception->getFile();
        $line = $exception->getLine();
        $trace = "$summary in $file:$line\nStack trace:\n$trace";
        AppLog::getInstance()->error($summary, $trace);
        _error($trace);
        if (empty(ini_get('display_errors'))){
            header('HTTP/1.1 500 Internal Server Error');
        } else {
            echo $trace;
        }
    } while($exception = $exception->getPrevious());
}

```

