---
title: Better Coder
date: 2018-7-23 15:26:55
tags: [Code]
---



什么是优质的代码，如何编写出优质的代码，在Coding的时候要做什么才能让自己的代码更为优雅简洁，本文**落足实际**，为编写出优秀的业务的代码而服务。

### 表格驱动法

核心：**拆分业务逻辑和具体数据**

```java
// 同样一段业务代码 如果在需要增加数据的时候，代码就会特别冗杂
function contry_initial($country){
    if ($country==="China" ){
       return "CHN";
    }else if($country==="America"){
       return "USA";
    }else if($country==="Japan"){
      return "JPN";
    }else{
       return "OTHER";
    }
}
// 然而将数据和逻辑分离之后，数据可以单独增加，不影响业务逻辑，代码的可重用性更高。
function contry_initial($country){
  $countryList=[
      "China"=> "CHN",
      "America"=> "USA",
      "Japan"=> "JPN",
    ];

    if(in_array($country, array_keys($countryList))) {
        return $countryList[$country];
    }
    return "Other";

}
```

在实际的多人开发的项目中，表格驱动法可以使得在添加相对应功能的时候，只需要修改数据而非变动逻辑，使得代码的维护性大大增强，风格统一。数据易于变动，逻辑相对而言更为稳定。

***

### 相关引用

[表格驱动法](https://www.zhihu.com/question/37943171/answer/119525120)