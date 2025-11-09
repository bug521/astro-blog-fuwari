---
title: "Go知识总结"
published: 2025-11-10
tags: ["技术", "教程"]
description: "go知识汇总"
category: "编程技术"
slug: "go-knowledge-summary"
---

## 一、Go语言基础

### 1.1 语言特性

- Go的设计理念与特点

- 与其他语言的对比(C/C++、Java、Python)

- Go的优势与应用场景

### 1.2 基本语法

- 变量声明与类型系统

- 常量与iota

- 基本数据类型(int、float、string、bool)

- 复合类型(数组、切片、map、结构体)

- 指针的使用

- 类型转换与类型断言

### 1.3 流程控制

- if-else条件语句

- for循环(包括range)

- switch-case语句

- defer、panic、recover机制

## 二、函数与方法

### 2.1 函数基础

- 函数声明与调用

- 多返回值

- 命名返回值

- 可变参数函数

- 匿名函数与闭包

### 2.2 方法

- 方法的定义与接收者

- 值接收者vs指针接收者

- 方法集(Method Set)

## 三、接口与面向对象

### 3.1 接口

- 接口的定义与实现

- 空接口(interface{})

- 类型断言与类型switch

- 接口的组合

- 常用标准接口(io.Reader、io.Writer、error等)

### 3.2 面向对象特性

- 封装

- 组合vs继承

- 多态的实现

## 四、并发编程

### 4.1 Goroutine

- goroutine的创建与使用

- goroutine的调度原理(GMP模型)

- goroutine泄漏问题

### 4.2 Channel

- channel的创建与使用

- 有缓冲vs无缓冲channel

- channel的关闭与检测

- 单向channel

- select语句

### 4.3 并发模式

- Worker Pool模式

- Pipeline模式

- Fan-in/Fan-out模式

- Context的使用

### 4.4 并发安全

- 竞态条件(Race Condition)

- 互斥锁(Mutex)与读写锁(RWMutex)

- 原子操作(sync/atomic)

- sync.WaitGroup

- sync.Once

- sync.Pool

## 五、内存管理

### 5.1 内存分配

- 堆与栈

- 逃逸分析

- 内存对齐

### 5.2 垃圾回收

- GC的工作原理

- 三色标记算法

- 写屏障

- GC的优化与调优

## 六、标准库

### 6.1 常用包

- fmt - 格式化输入输出

- strings - 字符串处理

- strconv - 类型转换

- time - 时间处理

- math - 数学计算

- regexp - 正则表达式

### 6.2 文件与IO

- os包 - 文件操作

- io包 - IO接口

- bufio包 - 缓冲IO

- ioutil包 - IO工具函数

### 6.3 网络编程

- net包 - TCP/UDP编程

- net/http包 - HTTP服务器与客户端

- net/url包 - URL解析

### 6.4 编码与解码

- encoding/json

- encoding/xml

- encoding/base64

## 七、错误处理

- error接口

- 自定义错误

- errors.New vs fmt.Errorf

- 错误包装(Go 1.13+)

- panic与recover的使用场景

## 八、测试

### 8.1 单元测试

- testing包的使用

- 表驱动测试

- 测试覆盖率

### 8.2 基准测试

- 性能测试

- 内存分配测试

### 8.3 其他测试

- 示例测试

- 模糊测试(Go 1.18+)

- Mock与接口测试

## 九、工具链

### 9.1 常用命令

- go build - 编译

- go run - 运行

- go test - 测试

- go mod - 模块管理

- go fmt - 代码格式化

- go vet - 代码检查

- go doc - 文档查看

### 9.2 性能分析

- pprof工具

- CPU性能分析

- 内存分析

- goroutine分析

- trace工具

## 十、反射

- reflect包的使用

- Type与Value

- 反射的三大法则

- 反射的性能影响

- 反射的应用场景

## 十一、泛型(Go 1.18+)

- 泛型的基本语法

- 类型参数与类型约束

- 泛型函数与泛型类型

- 常用泛型约束(comparable、any)

## 十二、Web开发

### 12.1 HTTP服务

- 创建HTTP服务器

- 路由处理

- 中间件模式

- 模板渲染(html/template)

### 12.2 常用框架

- Gin框架

- Echo框架

- Fiber框架

## 十三、数据库操作

- database/sql包

- 连接池管理

- 预处理语句

- 事务处理

- ORM框架(GORM、XORM)

## 十四、微服务与分布式

- gRPC

- Protocol Buffers

- 服务发现

- 负载均衡

- 分布式追踪

## 十五、性能优化

- 代码优化技巧

- 内存优化

- 并发优化

- 缓存策略

- 常见性能陷阱

## 十六、最佳实践

- 代码组织与项目结构

- 命名规范

- 错误处理最佳实践

- 并发编程最佳实践

- 代码审查要点

## 十七、常见面试题

### 17.1 基础概念

- Go的优缺点

- 值类型vs引用类型

- make vs new

- 切片的底层实现

- map的底层实现

### 17.2 并发相关

- goroutine与线程的区别

- channel的实现原理

- 如何避免goroutine泄漏

- 如何实现并发安全

### 17.3 高级话题

- 接口的底层实现

- GC的触发时机

- 内存泄漏的排查

- 性能调优经验

