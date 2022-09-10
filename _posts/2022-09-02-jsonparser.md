---
layout: post
title:  "Awk Json Parser"
date:   "2022-09-02 10:00:00"
---

Awk主要是用于文本数据处理的，而且主要是CSV格式。如果你碰到要处理Json数据，

就会比较麻烦。Awk原生没有提供Json的解析方法，可以加一个rapidjson的exlib。

rapidjson我在logwatch的项目中使用过，性能极佳。也许有人会想为什么不用jq呢，

如果awk的运算逻辑比较复杂的话，用jq解析是不现实的，还是需要操作一个awk的原生

数组，才能完全发挥awk的能力。

Github上有一个基于awk实现的[JsonParser](https://github.com/step-/JSON.awk)。

本着造轮子要造多个的基本原则，我也来凑个热闹，做个精简版的。

### 元素

1. String

2. Number

3. Boolean

4. Array

5. Object(key-values)

### 逻辑关系

按照Lpeg表达式的风格来写一下

```

Element = (String or Number or Boolean or Array or Object)

Array = [ Element , Element ...]

Key = String 

Object = { Key : Element , Key : Element ...}

JSON = (Array or Object)

```

### 元素类型的判断依据

```

  if(_sp == "\""){return "string";}
  if(_sp ~ /[0-9\-]/){return "number";}
  if(_sp == "t" || _sp == "f"){return "boolean";}
  if(_sp == "["){return "array";}
  if(_sp == "{"){return "object";}

```

### 完整代码

一个80行的JsonParser，简单明了。

说一个Awk的大坑，awk的变量是不能声明为local的，所以使用变量要非常慎重。

一旦需要local var，就要在函数params里声明一个虚参。

```
function fatal(_sp){ print "fatal error !!! index : ("IDX") "_sp; exit 1; }
function get(_np){ return substr(JSON_STRING, IDX, _np); }
function step(_np){ IDX+=_np; }
function skip(_sp, _greed){
  if(get(length(_sp)) == _sp){ 
    step(length(_sp)); 
    if(_greed == "true"){ skip(_sp, _greed); }
  } 
}
function skipSpace(){ skip(" ", "true" ); }
function skipDot(){ skip(",", "false"); }
function check(_sp){ 
  if(get(length(_sp)) == _sp){ step(length(_sp)); return ; }
  fatal("check failed "_np);  
}
function keyfc(_key, _i){
  for(_i = IDX; ; _i++){ if(substr(JSON_STRING, _i, 1) == "\""){ break; } }
  _key = get(_i - IDX); step(_i - IDX + 1); return _key;
}
function strfc(_json, _key, _i){
  for(_i = IDX; ; _i++){ if(substr(JSON_STRING, _i, 1) == "\"" && substr(JSON_STRING, _i - 1, 1) != "\\"){ break; } }
  _json[_key] = get(_i - IDX); step(_i - IDX + 1);
}
function numfc(_json, _key, _i){
  for(_i = IDX; ; _i++){ if(substr(JSON_STRING, _i, 1) !~ /[0-9.\-]/){ break; } }
  _json[_key] = get(_i - IDX); step(_i - IDX);
}
function boolfc(_json, _key){
  if(get(4) == "true" ){ _json[_key] = "true";  step(4); return; }
  if(get(5) == "false"){ _json[_key] = "false"; step(5); return; }
  fatal("not boolean");
}
function arrfc(_json, _key){
  _key = 0;
  while(1 == 1){
    skipSpace();
    if(get(1) == "]"){ step(1); skipSpace(); break; }
    if(get(1) == ","){ _json[_key++] = ""; skipDot(); continue; }
    dispatch(_json, _key++); skipSpace();
    if(get(1) != "]"){ check(","); }
  }
}
function objfc(_json, _key){
  while(1 == 1){
    skipSpace();
    if(get(1) == "}"){ step(1); skipSpace(); break; }
    check("\""); _key = keyfc(); skipSpace(); check(":"); skipSpace();
    dispatch(_json, _key); skipSpace();
    if(get(1) != "}"){ check(","); }
  }
}
function jtype(_sp){
  if(_sp == "\""){return "string";}
  if(_sp ~ /[0-9\-]/){return "number";}
  if(_sp == "t" || _sp == "f"){return "boolean";}
  if(_sp == "["){return "array";}
  if(_sp == "{"){return "object";}
  fatal("bad format");
}
function dispatch(_json, _key, _sp){
  _sp = jtype(get(1));
  if(_sp == "array"){
    _json[_key]["_type_"] = "array";  step(1); arrfc(_json[_key]);
  }else if(_sp == "object"){
    _json[_key]["_type_"] = "object"; step(1); objfc(_json[_key]);
  }else if(_sp == "string"){
    step(1); strfc(_json, _key);
  }else if(_sp == "number"){
    numfc(_json, _key);
  }else if(_sp == "boolean"){
    boolfc(_json, _key);
  }
}
BEGIN{JSON_STRING = ""; IDX = 1; result["_type_"] = "object";}
{ 
  JSON_STRING = $0; skipSpace();
  rootType = jtype(get(1)); if(rootType != "array" && rootType != "object"){ fatal("not json"); }
  dispatch(result, "_root_");
} 

```
