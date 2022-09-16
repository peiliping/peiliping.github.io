---
layout: post
title:  "Awk Linear"
date:   "2022-01-11 10:00:00"
---

上期讲到了Linear的概念，这次我们就用Awk来实现一个Linear吧。


### 代码

下面这段代码是从TradingView的LR脚本中翻译过来的，也是我找到的比较简洁的一个版本。

```

function lr(_source, _result, _i){
  _result["len"] = length(_source);
  for(_i = 1; _i <= _result["len"]; _i++){
    _result["sumx"]+=_i;
    _result["sumy"]+=_source[_i];
    _result["sumxx"]+=(_i * _i);
    _result["sumxy"]+=(_i * _source[_i]);
  }
  _result["slope"] = (_result["len"] * _result["sumxy"] - _result["sumx"] * _result["sumy"]) / (_result["len"] * _result["sumxx"] - _result["sumx"] * _result["sumx"]);
  _result["intercept"] = _result["sumy"] / _result["len"] - _result["slope"] * _result["sumx"] / _result["len"] + _result["slope"];
 
  for(_i = 1; _i <= _result["len"]; _i++){
    _result["stdDevAcc"]+=((_i * _result["slope"] + _result["intercept"] - _source[_i]) * (_i * _result["slope"] + _result["intercept"] - _source[_i]));
  }
  _result["stdDev"] = sqrt(_result["stdDevAcc"] / (_result["len"] - 1));
  _result["M"] = _result["len"] * _result["slope"] + _result["intercept"];
  _result["U"] = _result["M"] + 2 * _result["stdDev"];
  _result["D"] = _result["M"] - 2 * _result["stdDev"];
}

```

返回结果一般主要是获取slope、intercept、stdDev，还有U、M、D。


### 关于awk加载functions的问题

如果你写了一个完整的awk脚本可以通过参数 -f 来执行script，例如： ```awk -f xxx.awkscript```

如果你只是向上面LR一样实现了一个函数，希望能够被复用的话，可以这样： 

``` awk -i lib.awkscript  '{ lr(source, result); print result["M"];}' ```

如果你有多个lib文件的话，可以多次使用 -i  ``` -i a.awk -i b.awk -i c.awk ```

还可以再script文件里使用 @include 进行文件或者目录的加载，有点像模板加载宏。


### 关于awk中Array的赋值问题

awk的Array和Lua的Table特性非常相似，但是Awk没有提供像ipairs这样的迭代工具。

Array也无法成为Awk函数的返回值，只能在参数中传递进来，参考awk中split的用法。

如果map["key"]也是Array的话，无法直接进行变量的赋值，比如：val = map["key"]。