+++
title = "What I’ve Been Reading Lately"
date = 2026-09-07

[taxonomies]
tags = ["books"]
+++

[Kubernetes in Action](https://www.manning.com/books/kubernetes-in-action-second-edition)，这本面向开发者讲解如何使用k8s的书, 感觉主要是覆盖了CKAD相关的知识吧,  图文并茂,  详尽的图示表格清晰地阐释了k8s 各种api object.  适合边读边做,  [clone 对应的git 仓库](https://github.com/luksa/kubernetes-in-action-2nd-edition/)，自己动手练习.

[Kubernetes Programming with Go](https://link.springer.com/book/10.1007/978-1-4842-9026-2) ,   一本讲解如何使用golang 编程k8s的入门书吧，从raw http restful api  到更高阶易用的client-go,  controller-runtime 等golang package的使用,   最后是使用Kubebuilder项目生成基本的编写crd 资源的controller模板.  

[Learning Domain-Driven Design](https://www.oreilly.com/library/view/learning-domain-driven-design/9781098100124/),   首次了解ddd开发相关的概念,  整体感觉偏概念讲解吧,  对于domain model 基本构建块:  value objects, entity,  aggregates, domain services,  application services, 这些较容易理解,  其它Event-Sourced Domain Model， Data mesh architecture 感觉模模糊糊吧.   书中代码例子使用C#,  对于我来说有些隔阂. 剩下就是对这张图印象深刻:

<img src="/imgs/decision-making-in-lddd.png">

以下几本书算是在阅读Learning Domain-Driven Design过程对一些主题如event sourcing, event driven architecture遇到困惑不满足的进一步探索:

[Building Event-Driven Microservices, 2nd Edition](https://www.oreilly.com/library/view/building-event-driven-microservices/9798341622180/),  这本书某种程度仿佛是对ddia(Designing Data-Intensive Applications)部分章节的详尽展开讨论, 尤其是Batch Processing,   Stream Processing.   这里的讨论的技术主要是在为大数据提供批处理,实时处理方面存在优势,  很多中小企业可能没有这个需求,  而且这个领域目前主要是JVM语言框架Spark, Flink,  Beam,  kafka 等Apache 开源项目主导,  作者称前三者为重量级框架，kafka为轻量级框架, 个人感觉kafka已经比较厚重了，相比nats jetstream,  不过kafka提供了更多的功能.  希望这个领域涌现更多c++, rust 系统编程语言编写或者重写的项目，如ScyllaDB,  RedPanda,   Apache Iggy等. 

[Domain-Driven Design with Golang](https://www.packtpub.com/en-in/product/domain-driven-design-with-golang-9781804619261),  [Event-Driven Architecture in Golang](https://www.packtpub.com/en-in/product/event-driven-architecture-in-golang-9781803232188),  [Microservices with Go, Second Edition](https://www.packtpub.com/en-in/product/microservices-with-go-9781836207320),   这三本算是对Learning Domain-Driven Design中一些主题如何使用golang 生态中如何具体实现demo 展示，发现packtpub网站上三本恰好是推荐读者打包购买的推荐组合.

[Think Distributed Systems](https://www.manning.com/books/think-distributed-systems) ,  不到200页的小书, 为看待分布式系统提供一个精炼抽象的视角,  不过详尽程度不如DDIA 从第六章到第十章内容.

看了三本rust相关的书:

[Beginning Axum](https://link.springer.com/book/10.1007/979-8-8688-2631-3),  不是什么特别的好书,只是使用axum 库的指南书, 以前只使用过Actix-web, Warp 这两个rust社区的web framework,   Axum 风格类似Actix-web, 跟tokio 运行时绑定更紧密，比warp 这种函数式库更流行.

[Advanced Rust](https://link.springer.com/book/10.1007/979-8-8688-2992-5),    感觉名不符实, 内容谈不上多高级，还不如看官方文档.

[Mastering Distributed Observability in Rust](https://www.packtpub.com/en-in/product/mastering-distributed-observability-in-rust-9781806671786),    主要是distributed tracing 框架OpenTelemetry 在rust后端环境的应用,   系统可观察性基石:  trace,  log,  metric,  profiling 都各自有独立展开,   但开发团队应该培育Observability-Driven Development (ODD) 开发纪律， 不然在一个跨多种编程语言多节点多组件的微服务架构中,  很难排查问题(功能bug, 性能降级, 安全事故), 维护比较可靠的后端服务.   [据说otel 相关的生态目前最成熟稳定的是golang,dotnet](https://matduggan.com/otel-isnt-going-well-and-i-made-a-spreadsheet-about-it/),    rust 相关的库还在beta 阶段.     

经典重读:

[Designing Data-Intensive Applications - Second Edition](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781098119058/), 简称DDIA, 经典的后端开发必读更新到第二版，后端开发涉及的可以不仅是CRUD风格的业务胶合层,  还有Event-Sourcing, EDA 风格， 本书讲的就是那些后端开发经常依赖的核心组件, 也就是相比业务胶合层更困难的各种不同类型的数据处理系统背后的设计基本原理,各种不同的设计取舍. 
