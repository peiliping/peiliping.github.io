---
layout: post
title:  "Cmd Binance Client"
date:   "2022-02-19 10:00:00"
---

无意间在Binance的ApiDoc中看到了基于Curl的示例代码，于是便想封装一个基于Curl的

命令行客户端 。获取一般行情的Api接口直接使用Curl就可以，难题主要是获取个人数据

和交易行为接口,需要对参数进行签名。


### Client

```

ApiKey="zzzzzzzzzzzzzzzzzzzz"
SecretKey="xxxxxxxxxxxxxxxxx"

HttpCode=""
HttpResult=""

## 对参数数据进行签名
signature(){
  echo -n "$1" | openssl dgst -sha256 -hmac "${SecretKey}" | awk '{print "signature="$2}'
}

## 添加时间校对参数
addTimeParams(){
  local tsParams="recvWindow=10000&timestamp="`date +%s`"000"
  if [ -z "$1" ] ;then
    echo $tsParams
  else
    echo $1"&"$tsParams
  fi
}

## 发送带签名的请求
sendRequest(){
  HttpCode=""
  HttpResult=""
  local method=$1
  local host=$2
  local path=$3
  local params=$4
  params=`addTimeParams ${params}`
  local sig=`signature ${params}`
  params=$params"&"$sig
  local result=`curl -s -w "\t%{http_code}" -H "X-MBX-APIKEY: ${ApiKey}" -X ${method} $host""$path"?"$params`
  HttpCode=`echo $result | awk '{ print $NF}'`
  HttpResult=`echo $result | awk '{ print substr($0,1,length($0)-4); }'`
  if [ $HttpCode -eq 200 ];then
    return 0
  else
    echo $HttpResult
    return 1
  fi
}

```

签名的方法其实很简单，需要注意的是echo的参数-n，是不换行输出的意思。

如果不加这个参数，那么签名数据就无法通过Server端的校验了。

时间校对参数这里只是简单的用了当前时间，如果是谨慎设计的话，应该使用业务时间。

比如获取行情信息后下单，那么下单的校对时间应该是获取行情的时间。

如何获得Http的Code？curl默认是返回Http Response Body的，通过-w参数，

可以获取httpcode，会输出到最后，通过awk将其切分成Code和Result变量。

根据HttpCode来确定函数sendRequest的return值(0/1)。

我们来调用一下这个Client。

```

  local host="https://xxxxx.com"
  local path="/sapi/v3/asset/getUserAsset"
  sendRequest "POST" $host $path
  [[ $? -gt  0 ]] && return 1
  echo $HttpResult

```

判断sendRequest的return值，如果大于0，则是请求失败，程序终止。

[[ $? -gt  0 ]] && return 1 是一种if语句的简洁写法。


### Shell中如何识别Number类型

网上可以搜到很多种，我比如喜欢下面的这个方法

```

  expr $1 "+" 1 &> /dev/null
  if [ $? -eq 0 ];then
    echo "orderId"
  else
    echo "origClientOrderId"
  fi

```
