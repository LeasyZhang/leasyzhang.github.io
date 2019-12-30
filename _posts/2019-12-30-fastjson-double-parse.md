---
layout:     post
title:      "fastjson 解析double类型数字变成科学计数法的解决方法"
subtitle:   "fastjson 自定义解析格式"
date:       2019-12-30
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - fastjson
    - Java
---

> "fastjson解析大数字"

最近在使用fastjson的时候，发现在解析Double类型的数据时，如果数字小数点过多，fastjson会解析成科学计数法的形式。

```java
static class Stock {
    private Double price;

    public Double getPrice() {
        return price;
    }

    public void setPrice(Double price) {
        this.price = price;
    }
}

public static void main(String[] args) {
    Stock stock = new Stock();
    stock.setPrice(71094751259.14613451346151435145);

    System.out.println(JSON.toJSONString(stock));
}
```

上面这段代码的输出是

```json
{
    "price":7.109475125914613E10
}
```

这个问题有两种解决方法

- 使用BigDecimal类型的数值

```java
static class Stock {
    private BigDecimal price;

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }
}

public static void main(String[] args) {
    Stock stock = new Stock();
    stock.setPrice(new BigDecimal(71094751259.14613451346151435145));

    System.out.println(JSON.toJSONString(stock));
}
```

输出是

```json
{
    "price":71094751259.14613451346151435145
}
```

- 使用自定义的Double序列化方法

fastjson解析Double数值的时候使用的是com.alibaba.fastjson.serializer.DoubleSerializer，这个类有两个构造方法，一个是不带参数的默认构造方法，一个是带上DecimalFormat参数的构造方法。
*但是这个方法不能保留所有的小数位，只会保留部分小数*

```java
public DoubleSerializer(){

}

public DoubleSerializer(DecimalFormat decimalFormat){
    this.decimalFormat = decimalFormat;
}
```

默认情况下使用的是不带参数的构造方法，这个时候解析Double类型的数据会使用com.alibaba.fastjson.util.RyuDouble#toString(double, char[], int)这个方法，这个方法解析小数点过多的数值会转换成科学计数法的形式。

所以可以在使用的时候使用带有DecimalFormat参数的构造方法。

```java
Stock stock = new Stock();
stock.setPrice(71094751259.14613451346151435145);

SerializeConfig config = SerializeConfig.getGlobalInstance();
config.put(Double.class, new DoubleSerializer("#.#####"));

System.out.println(JSON.toJSONString(stock));
```

输出是

```java
{
    "price":71094751259.14613
}
```

#### 总结

这篇文章介绍了如何解决fastjson解析Double数据输出科学计数法的问题，第一个是用BigDecimal类型，这样可以保留所有的小数点位数，第二个是自定义DoubleSerializer，这种方法只会保留一部分小数点。
