# RecordofInterview


### Redis中的zset（有序集合）的底层实现原理

``` Redis的作者给出了一些解释```
> There are a few reasons:
> They are not very memory intensive. It's up to you basically. Changing parameters about the probability of a node to have a given number of levels will make then less memory intensive than btrees.
> A sorted set is often target of many ZRANGE or ZREVRANGE operations, that is, traversing the skip list as a linked list. With this operation the cache locality of skip lists is at least as good as with other kind of balanced trees.
> They are simpler to implement, debug, and so forth. For instance thanks to the skip list simplicity I received a patch (already in Redis master) with augmented skip lists implementing ZRANK in O(log(N)). It required little changes to the code.


#### 跳表和B+树

- 实现复杂度：

跳跃表: 实现相对简单直观，代码量少，维护成本低。它只需要维护随机层数和前后指针，插入、删除操作的逻辑清晰
B+树：实现复杂，需要处理节点分裂、合并、平衡等操作，代码复杂度高。Redis 的作者 Antirez 曾表示，更倾向于选择实现简单的方案，以减少潜在的 bug。

- 内存占用

跳跃表：每个节点的额外指针较少（通常为 2-3 个），内存占用相对可控。Redis 可以通过调整跳跃表的参数（如最大层数）来优化内存使用。
B + 树：每个节点需要维护多个子节点指针，且为了保持平衡，可能需要额外的元数据，内存占用相对较高

- 范围查询效率

跳跃表：支持双向遍历，可以在 O (log n) 时间内定位到范围起点，然后线性遍历后续节点，非常适合 Redis 中常见的 ZRANGE/ZREVRANGE 操作。
B + 树：虽然也支持范围查询，但需要通过中序遍历来实现，实现复杂度较高。此外，B + 树的节点访问可能涉及更多的随机 IO（在磁盘存储场景下更为明显）。

跳表： 一旦通过索引定位到范围的起始点（O(log N)），后续遍历整个范围只需要沿着最底层的链表线性扫描（O(M)， M是范围大小）。这非常高效，且代码简单直接。
B+树： 范围查询也很高效（O(log N + M)）。在磁盘数据库中，B+树叶子节点的链表结构是巨大的优势（顺序I/O）。但是，在纯内存环境中，这个优势被大大削弱了。 内存访问是随机的，遍历叶子节点链表与遍历跳表的底层链表在效率上差异不大，甚至跳表的简单链表遍历可能更“缓存友好”。

- 写入性能
  
跳跃表：插入和删除操作只需局部调整，不需要全局重构，且平均时间复杂度为 O (log n)，在高并发写入场景下表现稳定。
B + 树：插入和删除可能导致节点分裂或合并，需要全局调整，写入性能相对较低。

-  磁盘存储 vs 内存存储

B + 树：设计初衷是为磁盘存储优化，通过减少 IO 次数提高性能。但 Redis 是内存数据库，IO 不是瓶颈，因此 B + 树的优势无法充分发挥。
跳跃表：在内存中的随机访问性能优秀，且实现简单，更适合 Redis 的内存使用模式。

- 碎片化

跳表： 节点大小相对灵活（主要取决于随机生成的层数），内存分配通常是按需的单个节点。虽然指针会带来一些开销（每个节点有多个向前的指针和一个向后的指针），但在内存数据库环境下，其内存使用效率是可以接受的。内存碎片化问题相对可控。

B+树： 节点通常是固定大小的页（page）。在内存中，这意味着：

即使一个节点只存少量数据，也可能占用一个完整的页大小，造成内部碎片浪费。

频繁的插入删除可能导致节点频繁分裂合并，增加内存分配/释放次数，加剧外部碎片问题。

需要更复杂的内存管理策略来优化页的分配和回收

- 并发控制

跳表： 更容易实现高效的无锁（Lock-Free）或乐观锁并发控制。因为修改通常只影响局部几个节点及其指针，冲突域较小。Redis 在需要时（如集群模式下）可以更容易地在跳表基础上实现细粒度的并发控制。
B+树： 实现高效的无锁并发极其困难。树结构的平衡操作（分裂、合并）涉及多个节点和指针的大范围修改，需要复杂的锁协议（如 B-link 树）或范围锁，容易成为性能瓶颈或增加实现复杂性。在追求高性能的内存数据库中，这是一个重要考量。



总结：为什么是跳表而不是 B+树？

核心原因： 在纯内存操作的语境下，跳表在实现简单性、内存效率（尤其是避免固定页大小带来的碎片）、并发控制潜力以及范围查询性能方面，提供了更好的综合权衡。

B+树的优势（磁盘顺序访问）在内存中失效： B+树的主要优势在于其非常适合磁盘存储——它最小化磁盘 I/O 次数（通过树高）并最大化顺序 I/O（通过叶子节点链表）。然而，Redis 是一个内存数据库，访问模式是随机的 RAM 访问，B+树为磁盘优化的特性（如大节点、叶子链表）在内存中带来的收益有限，而其复杂性（实现、平衡维护、内存碎片、并发）则成为显著的负担。

简单来说：跳表是为内存中的有序数据结构量身定做的更简单、更灵活、并发性更好的解决方案，而 B+树是为磁盘优化的重型解决方案，其优势在内存环境中无法充分发挥，劣势却被放大。 因此，对于 Redis ZSET 的设计目标，跳表是更自然和高效的选择。
