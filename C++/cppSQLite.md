# C++实现一个简易的关系型数据库

- WLS2-Ubuntu22.04LTS
- gcc version 11.3.0 (Ubuntu 11.3.0-1ubuntu1~22.04)
- ruby 3.0.2p107 (2021-07-07 revision 0db68f0233) [x86_64-linux-gnu]
- RSpec 3.12


**整体架构：**
![[Pasted image 20221215202211.png]]

**前端 (front-end)**
- 分词器 (tokenizer)
- 解析器 (parser)
- 代码生成器 (code generator)

**后端 (back-end)**
- 虚拟机 (virtual machine)
- B树 (B-tree)
- 分页 (pager)
- 操作系统层接口 (os interface)

## REPL
