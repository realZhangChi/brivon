---
title: "C# 模式匹配全面指南：从基础到高级应用"
date: 2024-03-18
draft: false
tags: ["C#", ".NET", "模式匹配", "编程技巧"]
categories: ["编程语言"]
---

模式匹配是C#中一个强大而优雅的特性，它能让我们以更简洁、更直观的方式处理复杂的数据结构和类型检查。本文将深入探讨C#中的各种模式匹配技术，从基础概念到高级应用。

## 类型模式

### 基础类型匹配

最基本的类型模式匹配示例：

```csharp
public static string GetDataType(object data)
{
    return data switch
    {
        int => "这是一个整数",
        string => "这是一个字符串",
        DateTime => "这是一个日期",
        _ => "未知类型"
    };
}
```

### 类型模式与声明

结合类型检查和变量声明：

```csharp
public static decimal CalculatePrice(object product)
{
    if (product is Product { IsDiscounted: true } discountedProduct)
    {
        return discountedProduct.Price * 0.9m;
    }
    return product is Product p ? p.Price : 0m;
}
```

## 属性模式

### 基本属性匹配

使用属性模式简化对象属性检查：

```csharp
public class Order
{
    public decimal Total { get; set; }
    public bool IsVIP { get; set; }
    public string Status { get; set; }
}

public static string GetOrderPriority(Order order) =>
    order switch
    {
        { Total: > 1000, IsVIP: true } => "最高优先级",
        { Total: > 1000 } => "高优先级",
        { IsVIP: true } => "中等优先级",
        _ => "普通优先级"
    };
```

### 嵌套属性模式

处理复杂对象结构：

```csharp
public class Address
{
    public string Country { get; set; }
    public string City { get; set; }
}

public class Customer
{
    public string Name { get; set; }
    public Address ShippingAddress { get; set; }
}

public static decimal CalculateShippingCost(Customer customer) =>
    customer switch
    {
        { ShippingAddress: { Country: "中国", City: "北京" } } => 10.0m,
        { ShippingAddress: { Country: "中国" } } => 20.0m,
        { ShippingAddress: not null } => 50.0m,
        _ => throw new ArgumentException("无效的收货地址")
    };
```

## 位置模式

### 元组模式匹配

使用位置模式处理元组：

```csharp
public static string AnalyzePoint(Point point) =>
    (point.X, point.Y) switch
    {
        (0, 0) => "原点",
        (0, _) => "Y轴上的点",
        (_, 0) => "X轴上的点",
        (var x, var y) when x == y => "对角线上的点",
        _ => "普通点"
    };
```

### 解构模式

结合解构和模式匹配：

```csharp
public record Rectangle(double Width, double Height);

public static string ClassifyRectangle(Rectangle rect) =>
    rect switch
    {
        (0, 0) => "点",
        (var w, var h) when w == h => "正方形",
        (> 0, > 0) => "矩形",
        _ => "无效图形"
    };
```

## 关系模式

### 数值比较模式

使用关系运算符进行模式匹配：

```csharp
public static string GetTemperatureAlert(double temperature) =>
    temperature switch
    {
        < 0 => "严寒",
        <= 10 => "寒冷",
        <= 20 => "凉爽",
        <= 30 => "适中",
        <= 35 => "温暖",
        > 35 => "炎热",
        _ => "无效温度"
    };
```

### 组合关系模式

复杂条件的组合：

```csharp
public static string GetAgeGroup(int age) =>
    age switch
    {
        <= 0 => "无效年龄",
        > 0 and <= 2 => "婴儿",
        > 2 and <= 12 => "儿童",
        > 12 and <= 18 => "青少年",
        > 18 and <= 60 => "成年人",
        > 60 => "老年人"
    };
```

## 逻辑模式

### not 模式

使用not模式进行否定匹配：

```csharp
public static bool IsValidInput(string input) =>
    input is not (null or "" or " ");
```

### and 和 or 模式

组合多个条件：

```csharp
public static bool IsSpecialCase(object obj) =>
    obj is (string or int) and not null;
```

## 高级应用场景

### 列表模式

C# 11中的列表模式匹配：

```csharp
public static string AnalyzeSequence(int[] numbers) =>
    numbers switch
    {
        [] => "空序列",
        [1, 2, 3] => "特定序列[1,2,3]",
        [0, ..] => "以0开头的序列",
        [.., 0] => "以0结尾的序列",
        [var first, .., var last] => $"首元素为{first}，末元素为{last}的序列"
    };
```

### 递归模式

处理递归数据结构：

```csharp
public class BinaryTree
{
    public int Value { get; set; }
    public BinaryTree Left { get; set; }
    public BinaryTree Right { get; set; }
}

public static int CountNodes(BinaryTree tree) =>
    tree switch
    {
        null => 0,
        { Left: null, Right: null } => 1,
        { Left: var l, Right: var r } => 1 + CountNodes(l) + CountNodes(r)
    };
```

## 性能考虑

### 编译时类型检查

模式匹配在编译时进行类型检查，有助于避免运行时错误：

```csharp
public static decimal ProcessValue<T>(T value) =>
    value switch
    {
        int i => i * 2,
        decimal d => d * 1.5m,
        double d => (decimal)(d * 1.5),
        // 编译时会检查类型覆盖情况
        _ => throw new ArgumentException($"不支持的类型: {typeof(T)}")
    };
```

### 优化建议

1. 按照最具体到最一般的顺序排列模式
2. 避免过度复杂的模式组合
3. 考虑使用switch表达式而不是switch语句
4. 利用编译器优化

## 最佳实践

1. 使用模式匹配提高代码可读性
2. 合理组合不同类型的模式
3. 注意处理所有可能的情况
4. 保持模式简单明了
5. 适当使用when子句添加额外条件

## 总结

C#的模式匹配特性为我们提供了强大而灵活的工具，可以：
- 简化复杂的类型检查和分支逻辑
- 提高代码的可读性和维护性
- 减少代码冗余
- 提供编译时类型安全

通过合理使用模式匹配，我们可以写出更简洁、更优雅的代码。随着C#语言的发展，模式匹配特性还在不断增强，为我们提供更多可能性。

## 参考资源

- C#语言规范
- Microsoft文档
- .NET官方博客
- C#设计模式最佳实践指南
