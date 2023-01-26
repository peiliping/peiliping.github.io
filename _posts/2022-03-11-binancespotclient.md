---
layout: post
title:  "Binance Spot Client"
date:   "2022-03-11 10:00:00"
---

上期我们把BinanceApi的请求签名进行了封装，接下来我们就在这个基础上，实现核心功能。

### Client

```

getAssets(){
  local path="/sapi/v3/asset/getUserAsset"
  sendRequest "POST" $Host $path
  [[ $? -gt  0 ]] && return 1
  echo $HttpResult | awk -i ${AwkLib}/json.awk '
  {
    parserJson($0, json);
    for(i=0; i<length(json)-1; i++){
      print json[i]["asset"]"\t"json[i]["free"]"\t"json[i]["locked"];
    }
  }' | sort -k2nr
}

getOpenOrders(){
  local path="/api/v3/openOrders"
  sendRequest "GET" $Host $path
  [[ $? -gt  0 ]] && return 1
  echo $HttpResult | awk -i ${AwkLib}/json.awk '{parserJson($0, json);printJson(json);}'
}

getOrder(){
  local path="/api/v3/order"
  local symbol=`echo $1 | tr a-z A-Z`
  local key=`getOrderIdType $2`
  local value=$2
  local params="symbol=${symbol}&${key}=${value}"
  sendRequest "GET" $Host $path $params
  [[ $? -gt  0 ]] && return 1
  echo $HttpResult | awk -i ${AwkLib}/json.awk '{parserJson($0, json);printJson(json);}'
}

cancelOpenOrder(){
  local path="/api/v3/order"
  local symbol=`echo $1 | tr a-z A-Z`
  local key=`getOrderIdType $2`
  local value=$2
  local params="symbol=${symbol}&${key}=${value}"
  sendRequest "DELETE" $Host $path $params
  [[ $? -gt  0 ]] && return 1
  echo $HttpResult | awk -i ${AwkLib}/json.awk '{parserJson($0, json);printJson(json);}'
}

marketTrade(){
  local path="/api/v3/order"
  local symbol=`echo $1 | tr a-z A-Z`
  local side=$2
  local quantity=$3
  local params="symbol=${symbol}&side=${side}&type=MARKET&quantity=${quantity}"
  sendRequest "POST" $Host $path $params
  [[ $? -gt  0 ]] && return 1
  echo $HttpResult | awk -i ${AwkLib}/json.awk '{parserJson($0, json);printJson(json);}'
}

limitTrade(){
  local path="/api/v3/order"
  local symbol=`echo $1 | tr a-z A-Z`
  local side=$2
  local price=$3
  local quantity=$4
  local params="symbol=${symbol}&side=${side}&type=LIMIT&timeInForce=GTC&price=${price}&quantity=${quantity}"
  sendRequest "POST" $Host $path $params
  [[ $? -gt  0 ]] && return 1
  echo $HttpResult | awk -i ${AwkLib}/json.awk '{parserJson($0, json);printJson(json);}'
}

```

总的来说还是比较简单的，预处理一下参数，解析一下返回结果的json，按照一定的格式输出。

U本位合约的Api也是差不多的情况，逻辑基本相同。
