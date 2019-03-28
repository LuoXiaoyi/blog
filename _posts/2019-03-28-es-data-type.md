---
layout:    post
title:     ES 中数据类型使用
category:  ES
tags: ES数据类型
---

ES 的数据类型真的让人崩溃啊，对于初学者来说，简直就是噩梦一般，描述一下我踩的坑，主要是针对 List 的操作的，如下

{% highlight java %}
public static class NObj {
    String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

public static class Test {

    List<NObj> objs;
    List<String> test;
    String id;

    public List<String> getTest() {
        return test;
    }

    public void setTest(List<String> test) {
        this.test = test;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public List<NObj> getObjs() {
        return objs;
    }

    public void setObjs(List<NObj> objs) {
        this.objs = objs;
    }
}
{% endhighlight %}

在上面的 ```Test``` 类中包含有3种数据类型，分别是 ```List<NObj>``` 、```List<String>``` 、 ```String``` ，最开始，在 ES 的 mapping 中我给的数据类型分别是 object、object、keyword，在插入数据时，只要 ```List<String>``` 字段有值，就插不进数据，但是，```List<NObj>``` 的却可以，这很费解，毕竟在 java 中，String/NObj 都是对象，凭啥一个可以一个不行。呵呵...最后冷笑一声，就凭你是 ```Java``` ，我是 ```ES``` ，不服你就来治我啊？于是我找同事去讨论和翻阅有关文档，总结如下
* ```keyword``` 只能保存文本类型的对象，如 ```String```、```List<String>```等
* ```text``` 和 ```keyword``` 一样，只是 ```text``` 支持分词
* ```nested``` 用来保存内嵌对象，如 ```List<NObj>``` 这种数据
* ```object``` 用来保存对象，但是不能保存如 ```String```、```List<String>```这种，不过像```List<Nobj>```是可以保存的

这里不得不多提一句的是，在 ES 中的对象，都应该是类似 ```Java``` 中的 ```Map```一样是，类似
```
{
    "key":"value",
    "key2":"value2"
}
```
这和```Java```中的对象有区别，需要注意！如：```Java``` 中的 String 对象就不符合要求，所以不能被当做 ```ES``` 中的 ```object``` 来保存，这就可以理解，为啥 ```List<String>```、```String``` 不能保存为 ```ES``` 中的 ```object``` 了