# malloc

## 粗略的描述分配流程

1. 如果分配内存<512字节，则通过内存大小定位到smallbins对应的index上(floor(size/8))
   * 如果smallbins[index]为空，进入步骤3
   * 如果smallbins[index]非空，直接返回第一个chunk
2. 如果分配内存>512字节，则定位到largebins对应的index上 
   * 如果largebins[index]为空，进入步骤3
   * 如果largebins[index]非空，扫描链表，找到第一个大小最合适的chunk，如size=12.5K，则使用chunk B，剩下的0.5k放入unsorted_list中
3. 遍历unsorted_list，查找合适size的chunk，如果找到则返回；否则，将这些chunk都归类放到smallbins和largebins里面
4. index++从更大的链表中查找，直到找到合适大小的chunk为止，找到后将chunk拆分，并将剩余的加入到unsorted_list中
5. 如果还没有找到，那么使用top chunk
6. 或者，内存<128k，使用brk；内存>128k，使用mmap获取新内存

