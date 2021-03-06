## Buffer Pool

​		为了缓存磁盘中的页，在`MySQL`服务器启动的时候就向操作系统申请了一片连续的内存，他们给这片内存起了个名，叫做`Buffer Pool`（中文名是`缓冲池`），用以节省 磁盘IO的开销

## Buffer Pool 结构

​		`Buffer Pool`中默认的缓存页大小和在磁盘上默认的页大小是一样的，都是`16KB`。为了更好的管理这些在`Buffer Pool`中的缓存页，为每一个缓存页都创建了一些所谓的**`控制信息`**，这些控制信息包括该页所属的表空间编号、页号、缓存页在`Buffer Pool`中的地址、链表节点信息、一些锁信息以及`LSN`信息等。

​		每个缓存页对应的控制信息占用的内存大小是相同的，称为一个`控制块`，控制块和缓存页是一一对应的，它们都被存放到 Buffer Pool 中，其中控制块被存放到 Buffer Pool 的前边，缓存页被存放到 Buffer Pool 后边，所以整个`Buffer Pool`对应的内存空间看起来就是这样的：

![img](InnoDB的BufferPool.assets/1693e86e2b9d6dd1.png)

​		剩余的那点儿空间不够一对控制块和缓存页的大小时，这部分空间就被称为`碎片`。

## Free 链表

​		当启动`MySQL`服务器的时候，需要完成对`Buffer Pool`的初始化过程，就是先向操作系统申请`Buffer Pool`的内存空间，然后把它划分成若干对控制块和缓存页。但是此时并没有真实的磁盘页被缓存到`Buffer Pool`中，之后随着程序的运行，会不断的有磁盘上的页被缓存到`Buffer Pool`中。

​		把所有空闲的缓存页对应的控制块作为一个节点放到一个链表中，这个链表被称作`free链表`（空闲链表）。刚刚完成初始化的`Buffer Pool`中所有的缓存页都是空闲的，所以每一个缓存页对应的控制块都会被加入到`free链表`中，假设该`Buffer Pool`中可容纳的缓存页数量为`n`，那增加了`free链表`的效果图就是这样的：

![img](InnoDB的BufferPool.assets/1693e86e300173c1.png)

​		每当需要从磁盘中加载一个页到`Buffer Pool`中时，就从`free链表`中取一个空闲的缓存页，并且把该缓存页对应的`控制块`的信息填上，然后把该缓存页对应的`free链表`节点从链表中移除。

## 缓存页的 Hash 处理

​		当需要访问某个页中的数据时，就会把该页从磁盘加载到`Buffer Pool`中，如果该页已经在`Buffer Pool`中的话直接使用就可以了。如何知道该页在不在`Buffer Pool`中呢？其实是根据`**表空间号 + 页号**`来定位一个页的，也就相当于`表空间号 + 页号`是一个`key`，`缓存页`就是对应的`value`，通过哈希表结构能够快速查找到一个页是否在`Buffer Pool`中

## Flush 链表

​		如果修改了`Buffer Pool`中某个缓存页的数据，这样的缓存页也被称为`脏页`。最简单的做法就是每发生一次修改就立即同步到磁盘上对应的页上，但是频繁的往磁盘中写数据会严重的影响程序的性能。所以**每次修改缓存页后，并不着急立即把修改同步到磁盘上，而是在未来的某个时间点进行同步**，所以，需要创建一个存储脏页的链表，凡是修改过的缓存页对应的控制块都会作为一个节点加入到一个链表中，因为这个链表节点对应的缓存页都是需要被刷新到磁盘上的，所以也叫`flush链表`。链表的构造和`free链表`差不多。

## LRU 链表

​		`Buffer Pool`对应的内存大小毕竟是有限的，如果需要缓存的页占用的内存大小超过了`Buffer Pool`大小，就需要把某些旧的缓存页从`Buffer Pool`中移除，然后再把新的页放进来。

​		回到设立`Buffer Pool`的初衷，就是想减少和磁盘的`IO`交互，最好每次在访问某个页的时候它都已经被缓存到`Buffer Pool`中了。假设一共访问了`n`次页，那么被访问的页已经在缓存中的次数除以`n`就是所谓的`缓存命中率`，我们的期望就是让`缓存命中率`越高越好～ 从这个角度出发，需要 LRU 链表来实现这种调度：

- 如果该页不在`Buffer Pool`中，在把该页从磁盘加载到`Buffer Pool`中的缓存页时，就把该缓存页对应的`控制块`作为节点塞到链表的头部。
- 如果该页已经缓存在`Buffer Pool`中，则直接把该页对应的`控制块`移动到`LRU链表`的头部

### LRU 存在的问题

- **预读**策略：`InnoDB`认为执行当前的请求可能之后会读取某些页面，就预先把它们加载到`Buffer Pool`中。根据触发方式的不同，`预读`又可以细分为下边两种：

  - 线性预读：InnoDB提供了一个系统变量`innodb_read_ahead_threshold`，如果顺序访问了某个区的页面超过这个系统变量的值，就会触发一次`异步`读取下一个区中全部的页面到`Buffer Pool`的请求，注意`异步`读取意味着从磁盘中加载这些被预读的页面并不会影响到当前工作线程的正常执行。
  - 随机预读：如果`Buffer Pool`中已经缓存了某个区的13个连续的页面，不论这些页面是不是顺序读取的，都会触发一次`异步`读取本区中所有其的页面到`Buffer Pool`的请求。InnoDB提供了`innodb_random_read_ahead`系统变量，它的默认值为`OFF`，也就意味着`InnoDB`并不会默认开启随机预读的功能

  **如果预读到`Buffer Pool`中的页成功的被使用到，那就可以极大的提高语句执行的效率。如果用不到，则这些预读的页都会放到`LRU`链表的头部，但是如果此时`Buffer Pool`的容量不太大而且很多预读的页面都没有用到的话，这就会导致处在`LRU链表`尾部的一些缓存页会很快的被淘汰掉，也就是所谓的`劣币驱逐良币`，会大大降低缓存命中率。**

- 全表扫描的查询：如果没有建立合适的索引或者压根儿没有WHERE子句的查询，则会触发全表扫描，假设这个表中记录非常多的话，若把它们统统都加载到`Buffer Pool`中，这也就意味着`Buffer Pool`中的所有页都被换了一次血，其他查询语句在执行时又得执行一次从磁盘加载到`Buffer Pool`的操作。而这种全表扫描的语句执行的频率也不高，每次执行都要把`Buffer Pool`中的缓存页换一次血，这严重的影响到其他查询对 `Buffer Pool`的使用，从而大大降低了缓存命中率。

### 解决方案

把这个`LRU链表`按照**一定比例**分成两截，分别是：

- 一部分存储使用频率非常高的缓存页，所以这一部分链表也叫做`热数据`，或者称`young区域`。
- 另一部分存储使用频率不是很高的缓存页，所以这一部分链表也叫做`冷数据`，或者称`old区域`。

![img](InnoDB的BufferPool.assets/1693e86e2a3fffa3.png)

有了这个划分，就可以针对上边提到的两种可能降低缓存命中率的情况进行优化了：

- 针对预读的页面可能不进行后续访问情况的优化：当磁盘上的某个页面在初次加载到Buffer Pool中的某个缓存页时，该缓存页对应的控制块会被放到old区域的头部。这样针对预读到`Buffer Pool`却不进行后续访问的页面就会被逐渐从`old`区域逐出，而不会影响`young`区域中被使用比较频繁的缓存页。
- 针对全表扫描时，**短时间内访问大量使用频率非常低的页面**情况的优化：在进行全表扫描时，**虽然首次被加载到`Buffer Pool`的页被放到了`old`区域的头部，但是后续会被马上访问到，每次进行访问的时候又会把该页放到`young`区域的头部，这样仍然会把那些使用频率比较高的页面给顶下去**。（因为`InnoDB`规定每次去页面中读取一条记录时，都算是访问一次页面，而一个页面中可能会包含很多条记录，也就是说读取完某个页面的记录就相当于访问了这个页面好多次）。但是全表扫描有一个特点，那就是它的**执行频率非常低**，而且在执行全表扫描的过程中，即使某个页面中有很多条记录，也就是去**多次访问这个页面所花费的时间也是非常少的**。所以只需要在对某个处在`old`区域的缓存页进行第一次访问时就在它对应的控制块中记录下来这个访问时间，**如果后续的访问时间与第一次访问的时间在某个时间间隔内，那么该页面就不会被从old区域移动到young区域的头部，否则将它移动到young区域的头部**。上述的这个间隔时间是由系统变量`innodb_old_blocks_time`控制的

### 更进一步优化

​		对于`young`区域的缓存页来说，每次访问一个缓存页就要把它移动到`LRU链表`的头部，这样的开销也不可忽略，毕竟在`young`区域的缓存页都是热点数据，也就是可能被经常访问的，为了解决这个问题其实还可以提出一些优化策略：比如**只有被访问的缓存页位于`young`区域的`1/4`的后边，才会被移动到`LRU链表`头部**，这样就可以降低调整`LRU链表`的频率，从而提升性能（也就是说如果某个缓存页对应的节点在`young`区域的`1/4`中，再次访问该缓存页时也不会将其移动到`LRU`链表头部）

## 刷新脏页到磁盘

后台有专门的线程每隔一段时间负责把脏页刷新到磁盘，这样可以不影响用户线程处理正常的请求。主要有两种刷新路径：

- 从`LRU链表`的冷数据中刷新一部分页面到磁盘：

  后台线程会定时从`LRU链表`尾部开始扫描一些页面，扫描的页面数量可以通过系统变量`innodb_lru_scan_depth`来指定，如果从里边儿发现脏页，会把它们刷新到磁盘。这种刷新页面的方式被称之为`BUF_FLUSH_LRU`。

- 从`flush链表`中刷新一部分页面到磁盘：

  后台线程也会定时从`flush链表`中刷新一部分页面到磁盘，刷新的速率取决于当时系统是不是很繁忙。这种刷新页面的方式被称之为`BUF_FLUSH_LIST`。

## 设置多个 Buffer Pool

​		`Buffer Pool`本质是`InnoDB`向操作系统申请的一块连续的内存空间，在多线程环境下，访问`Buffer Pool`中的各种链表都需要加锁处理，在`Buffer Pool`特别大而且多线程并发访问特别高的情况下，单一的`Buffer Pool`可能会影响请求的处理速度。所以在`Buffer Pool`特别大的时候，可以把它们拆分成若干个小的`Buffer Pool`，每个`Buffer Pool`都称为一个`实例`，它们都是独立的，独立的去申请内存空间，独立的管理各种链表，所以在多线程并发访问时并不会相互影响，从而提高并发处理能力。我们可以在服务器启动的时候通过设置`innodb_buffer_pool_instances`的值来修改`Buffer Pool`实例的个数。

![img](InnoDB的BufferPool.assets/1693e86e2abd79c1.png)

