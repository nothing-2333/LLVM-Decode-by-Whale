## 简介
支配树是一个重要的结构, 在鲸书, 虎书, 橡书中皆有讲解. 涉及到了很多概念: 支配, 必经节点, 直接必经节点, 严格必经节点...

首先认真读三本书有关控制流分析部分, 然后在研究 Lengauer-Tarjan 算法, 在虎书 310 页, 鲸书 134 页均有讲解, 同样有很多 bolg:
- https://blog.csdn.net/u012061345/article/details/129832025
- https://www.cnblogs.com/meowww/p/6475952.html
- ......

## Lengauer-Tarjan 算法在 llvm 中的实现
`llvm/include/llvm/Support/GenericDomTreeConstruction.h` 文件实现了 llvm 中用于构建支配树的 **Semi-NCA 算法**, 一种简化版的 Lengauer-Tarjan 算法