# Link

- [图解 Linux 程序的链接原理](https://juejin.cn/post/6844903544043077646)
- [Executable and Linkable Format](chrome-extension://ikhdkkncnoglghljlkmcimlnlhkeamad/pdf-viewer/web/viewer.html?file=http%3A%2F%2Fwww.skyfree.org%2Flinux%2Freferences%2FELF_Format.pdf#=&zoom=100) 是描述 ELF 文件格式的文档，只有 60 页但面面俱到地讲解了静态链接和动态加载的原理，是了解 ELF 文件的必读材料。然而这个版本已经略有过时，如果按照这个文档去分析现在的 ELF 文件，会发现一些新的属性在文档中是缺失的。尽管如此，因为这个本手册比较薄，所以它的可读性很好，建议在阅读更详细的文档前先读一下这个文档。
- [Oracle ELF Application Binary Interface](https://docs.oracle.com/cd/E23824_01/html/819-0690/glcfv.html#scrolltoc) 是描述 ELF 文件格式最详细和最新的文档，适合当作手册来查阅。
- [Eli Bendersky 的 Load-Time Relocation of Shared Libraries](https://eli.thegreenplace.net/2011/08/25/load-time-relocation-of-shared-libraries/)是讲解共享对象加载原理的文章。
- 