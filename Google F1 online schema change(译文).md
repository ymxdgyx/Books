<center><font size=6><b>F1中在线，异步的模式变更</b></font></center>

## 摘要

我们在共享数据、无状态服务、没有全局成员身份的全球分布式数据库管理系统提出了一个模式演变的协议。我们协议是*异步的*-它允许在数据库系统不同的服务器上、不同的时间，过渡到一个新的模式-并且也是*在线的*-在模式变更时所有的服务器可以访问并且更新所有的数据。我们提供了一个正式的模型来确定在这种情况下模式更改的正确性，并且我们证明了许多常见的模式更改可能会导致异常和数据库损坏。我们通过用一系列模式更改来替换导致损坏的模式更改来避免这些问题，只要所有服务器在任何时候都不落后一个模式版本，就可以保证避免损坏数据库。最后，我们讨论了F1中协议的实际实现，F1是一种用于存储Google AdWords数据的数据库管理系统。

## 1.介绍

模式演变（在不丢失数据的情况下更改数据库定义的能力）已经在数据库研究社区中进行了二十多年的研究[17]。在本文中，我们描述了Google F1数据库管理系统[21]在新环境中支持模式演变的技术。F1是一个全球分布式关系型数据库管理系统，提供强一致、高可用、并且允许用户通过SQL进行查询。Spanner经过一段时间的发展，提供了自己的关系抽象层，其中包括F1提供的某些功能。 但是，F1是围绕Spanner的早期版本开发的，因此，其实实现细节，设计目标和假设与Spanner不同。 我们只关注Spanner与我们用于模式演变的协议相关的功能，因此将Spanner视为键值存储。

F1的主要特征会影响架构更改：

**大规模分布**F1实例由数百个单独的F1服务器组成，它们在分布于全球许多数据中心的共享计算机集群上运行。

**关系模式**每个F1服务器都有一个关系模式的副本，该副本描述了表，列，索引和约束。 对模式的任何修改都需要更改分布式模式以更新到所有服务器上

**共享数据存储**所有数据中心中的所有F1服务器都可以访问Spanner中存储的所有数据。 F1服务器之间没有数据分区。

**无状态服务器**F1服务器必须容忍机器故障，抢占和失去对网络资源的访问。 为了解决这个问题，F1服务器基本上是无状态的-客户端可以连接到任何F1服务器，即使对于同一事务一部分的不同语句也是如此。

**无全局成员身份**由于F1服务器是无状态的，因此F1无需实现全局成员资格协议。 这意味着没有确定当前正在运行的F1服务器的可靠机制，并且不可能进行显式全局同步。

F1架构的这些方面以及它作为Google AdWords的主要数据存储的角色，对架构更改过程施加了一些约束：

**完整的数据可用性**F1是Google业务基础架构的关键部分。 因此，由F1管理的数据的可用性至关重要-可以衡量AdWords F1实例的任何停机时间都直接影响Google的收入，并且对架构的修改不能使系统离线。
此外，由于这种收入损失，在架构更改期间使数据库的一部分脱机也是不可接受的（例如，锁定列以建立索引）。

**对性能的影响最小化**AdWords F1实例在Google中需要访问AdWords数据的许多不同团队和项目之间共享。F1模式迅速变化以支持这些团队创建的新功能和业务需求。

通常，每周都会应用多个模式更改。 因为模式更改是频繁的，所以它们对用户操作的响应时间的影响必须最小。

**异步模式更改**由于F1中没有全局成员身份，因此我们无法在所有F1服务器之间同步模式更改。 换句话说，不同的F1服务器可能会在不同时间转换为使用新架构。

这些要求以多种方式影响了我们的架构更改过程的设计。 首先，由于所有数据必须尽可能地可用，因此我们不限制对正在进行重组的数据的访问。 其次，由于架构更改必须对用户事务产生最小的影响，因此尽管我们不会自动重写查询以使其符合所使用的架构，但我们允许事务跨越任意数量的模式更改。 最后，在单个F1服务器上异步应用模式更改意味着可以同时使用多个版本的模式（请参见图1）。

因为每个服务器都具有对所有数据的共享访问权，所以使用不同架构版本的服务器可能会破坏数据库。 考虑从模式S1到模式S2的模式更改，该更改在表R上添加了索引I。假设两个不同的服务器M1和M2执行以下操作序列：

  1. 服务器M2,使用模式S2,插入一行数据r到表R中，因为S2包含索引I,服务器M2也插入一条与r相对应的索引到键-值存储中。

  2. 服务器M1，使用模式S1,删除r，因为S1没有包含索引I,M1从键-值存储中删除r，但是没有相应的删除r在I中对应的索引。

第二次删除使数据库损坏。 例如，仅索引扫描将返回不正确的结果，其中包括已删除行r的列值。

我们的协议试图通过解决在所有服务器之间共享数据访问的分布式数据库系统中的异步，在线模式演变的一般问题来防止这种损坏。 我们不仅考虑更改逻辑模式（例如添加或删除列），还考虑更改物理模式（例如添加或删除辅助索引）。通过确保在任何给定时间使用不超过两个模式版本，并且这些模式版本具有特定的属性，我们的协议可以通过以下方式启用分布式架构更改：不需要全局成员资格，节点之间的隐式或显式同步，也不需要在架构更改完成后保留旧的架构版本。 本文的主要贡献是：

- 描述了我们在键值存储之上构建全球规模的分布式DBMS时所做的设计选择及其对架构更改操作的影响。
- 在共享访问数据的系统中，用于异步执行模式更改的协议，同时避免导致数据损坏的异常。 该协议允许在架构更改期间通过用户事务对数据库进行并行修改。
- 提议的模式更改协议的正式模型和正确性证明。
- 讨论了为实施协议对系统所做的更改。
- 评估该协议在Google AdWords使用的生产系统中的实施和效果。

本文的其余部分安排如下。 在第2节中，我们将解释键值存储的接口以及F1的关系模式和操作的高级设计。在第3节中，我们为支持的模式更改提供一个模型，并说明如何设计它们以防止可能破坏数据库的各种异常。接下来，我们将在第4节中介绍在我们的生产F1系统中如何实现这些架构更改，该系统已经为Google的AdWords流量提供了超过一年的服务，我们将在第5节中提供一些有关系统性能和整体用户体验的信息。最后，我们在第6节中讨论相关工作，并在第7节中进行总结。

## 2.背景
F1提供有关存储为键和键值存储中的值的数据的关系视图。 在本节中，我们将键值存储提供的接口与其实现分开，并说明如何将传统的关系数据库功能映射到此唯一设置中。

## 3.模式变更

There is nothing as practical as a good theory.
