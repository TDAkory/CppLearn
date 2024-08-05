# 温故知新

* 计算机核心：CPU、内存、IO
  * 北桥：高速数据传输
  * 南桥：低速数据传输
  * LBA(Logical Block Address)：描述计算机存储设备上数据所在区块的通用机制

* “Any problem in computer science can be solved by another layer of inderection”

* 分段和分页
  * 分段是分别加在不同程序的地址空间
  * 分页粒度更小，更好的利用了局部性原理
  * 虚拟地址空间的页是虚拟页，物理内存的页是物理页，磁盘上的页叫磁盘页
* MMU负责虚拟地址到物理地址的映射

* 进程、线程、协程