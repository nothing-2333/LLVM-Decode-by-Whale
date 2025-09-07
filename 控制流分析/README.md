**鲸书第七章**

## cfg
在 llvm 源码中找到 `llvm/lib/Analysis/CFG.cpp` 与 `llvm/include/llvm/Analysis/CFG.h`

因为文件都不长, 先直接贴源码 + 注释, 然后在分析.
- CFG.h:
  ```cpp
  #ifndef LLVM_ANALYSIS_CFG_H
  #define LLVM_ANALYSIS_CFG_H

  #include "llvm/ADT/GraphTraits.h"   // 引入 LLVM 抽象数据类型中的图遍历工具
  #include "llvm/ADT/SmallPtrSet.h"   // 引入 LLVM 轻量级指针集合容器
  #include <utility>                  // 引入标准库中的配对（pair）工具

  namespace llvm {  // LLVM 项目核心命名空间

  // 前向声明：避免循环依赖，提前声明后续用到的核心类
  class BasicBlock;    // 基本块（控制流图的基本单元）
  class DominatorTree; // 支配树（用于分析基本块间的支配关系）
  class Function;      // 函数（LLVM IR 中的顶层代码单元）
  class Instruction;   // 指令（LLVM IR 中的运算/操作单元）
  class LoopInfo;      // 循环信息（用于分析函数中的循环结构）
  template <typename T> class SmallVectorImpl; // 轻量级向量容器（LLVM 自定义实现）

  /// 分析指定函数，找出函数中所有的“循环回边”并返回结果。
  /// 相较于计算“支配树”和“循环信息”，此分析的时间/空间开销更低（更轻量）。
  ///
  /// 分析结果将以 <起点块, 终点块> 的配对形式存入 Result 容器中。
  void FindFunctionBackedges(
      const Function &F,  // 待分析的函数（const 修饰，表示不修改函数内容）
      SmallVectorImpl<std::pair<const BasicBlock *, const BasicBlock *>> &
          Result);  // 存储结果的容器（引用传递，接收分析出的循环回边）

  /// 搜索基本块 BB 的指定“后继块”，并返回该后继块在“终止指令的后继列表”中的索引位置。
  /// 注意：若传入的“后继块”并非 BB 的实际后继，则调用此函数属于错误用法。
  unsigned GetSuccessorNumber(const BasicBlock *BB,  // 待查询的基本块
                            const BasicBlock *Succ); // 待定位的后继基本块

  /// 若指定的边是“临界边”，则返回 true。
  /// 临界边的定义：从“有多个后继的基本块”指向“有多个前驱的基本块”的控制流边。
  ///
  /// 重载1：通过“终止指令 + 后继索引”指定边
  bool isCriticalEdge(const Instruction *TI,        // 基本块的终止指令（如分支、跳转指令）
                      unsigned SuccNum,             // 后继块在终止指令中的索引
                      bool AllowIdenticalEdges = false); // 是否允许“相同边”（默认不允许）

  /// 重载2：通过“终止指令 + 后继块指针”指定边
  bool isCriticalEdge(const Instruction *TI,        // 基本块的终止指令
                      const BasicBlock *Succ,       // 目标后继基本块
                      bool AllowIdenticalEdges = false); // 是否允许“相同边”（默认不允许）

  /// 判断指令“To”是否可从指令“From”到达（路径不经过 ExclusionSet 中的任何基本块），
  /// 若无法确定（分析存在保守性），则返回 true。
  ///
  /// 核心功能：判断在单个函数内是否存在从“From 指令所在块”到“To 指令所在块”的控制流路径。
  /// 返回规则：仅当能证明“执行 From 后绝对无法执行 To”时返回 false；否则（包括不确定的情况）返回 true（保守策略）。
  ///
  /// 算法特性：此函数的时间复杂度与控制流图（CFG）中的基本块数量呈线性关系，
  /// 从 From 所在块开始向下遍历后继块以寻找 To 所在块，且遍历深度设有固定阈值。
  /// 优化方式：传入 DominatorTree（DT）或 LoopInfo（LI）可加速分析：
  /// - LI（循环信息）：将任意规模的循环视为“单个块”处理，降低循环内遍历成本；
  /// - DT（支配树）：当找到“支配 To 所在块的基本块”时可终止搜索，对分支密集型代码优化明显；
  /// 适用场景：DT 对循环外的分支代码更有效，LI 对含循环的代码更有效，二者互补。
  bool isPotentiallyReachable(
      const Instruction *From,  // 起点指令（分析的起始点）
      const Instruction *To,    // 终点指令（分析的目标点）
      const SmallPtrSetImpl<BasicBlock *> *ExclusionSet = nullptr, // 禁止经过的块集合（可选，默认无）
      const DominatorTree *DT = nullptr, // 支配树（可选，用于加速分析）
      const LoopInfo *LI = nullptr);     // 循环信息（可选，用于加速分析）

  /// 判断基本块“To”是否可从基本块“From”到达，若无法确定则返回 true。
  ///
  /// 核心功能：判断在单个函数内是否存在从 From 到 To 的控制流路径。
  /// 返回规则：仅当能证明“到达 From 后绝对无法到达 To”时返回 false；否则返回 true（保守策略）。
  bool isPotentiallyReachable(
      const BasicBlock *From,  // 起点基本块
      const BasicBlock *To,    // 终点基本块
      const SmallPtrSetImpl<BasicBlock *> *ExclusionSet = nullptr, // 禁止经过的块集合（可选）
      const DominatorTree *DT = nullptr, // 支配树（可选，加速分析）
      const LoopInfo *LI = nullptr);     // 循环信息（可选，加速分析）

  /// 判断是否存在至少一条从“Worklist 中的块”到“StopBB”的路径（不经过 ExclusionSet 中的块），
  /// 若无法确定则返回 true。
  ///
  /// 核心功能：判断在单个函数内，是否存在从 Worklist 中任意一个块到 StopBB 的控制流路径，
  /// 且路径不经过 ExclusionSet 中的任何块。
  /// 返回规则：仅当能证明“到达 Worklist 中任意块后，绝对无法到达 StopBB”时返回 false；否则返回 true（保守策略）。
  bool isPotentiallyReachableFromMany(
      SmallVectorImpl<BasicBlock *> &Worklist, // 起点块集合（待遍历的起始块列表）
      const BasicBlock *StopBB,                // 终点基本块（目标块）
      const SmallPtrSetImpl<BasicBlock *> *ExclusionSet, // 禁止经过的块集合（必选）
      const DominatorTree *DT = nullptr,       // 支配树（可选，加速分析）
      const LoopInfo *LI = nullptr);           // 循环信息（可选，加速分析）

  /// 若 \p RPOTraversal 所代表的控制流图（CFG）是“不可约的”，则返回 true。
  ///
  /// 核心功能：这是一个基于“循环信息分析”检测 CFG 不可约性的通用实现。
  /// 适用场景：通过提供“反向后序遍历（RPOTraversal）”和“循环信息（LI）”，
  /// 可用于任何类型的 CFG（如 Loop、MachineLoop、Function、MachineFunction 等）。
  /// 使用建议：仅当“循环信息分析”已可用时，才推荐使用此函数；若循环信息未提前计算，
  /// 不建议为调用此函数而显式计算（存在更高效的不可约性检测算法，如 T1/T2 算法、Tarjan 算法，无需依赖循环信息）。
  ///
  /// 调用要求：
  ///   1) 必须为 NodeT 类型实现 GraphTraits 模板：用于获取 NodeT 的后继节点（遍历依赖）；
  ///   2) \p RPOTraversal 必须是目标 CFG 的“有效反向后序遍历”，且支持 begin()/end() 迭代器接口；
  ///   3) \p LI 必须是有效的 LoopInfoBase 实例，且包含 CFG 的最新循环分析结果。
  ///
  /// 算法原理：此算法利用 \p LI 中已计算的“可约循环回边”信息：
  /// 在反向后序遍历过程中，若发现一条“回边”，则检查该回边是否属于循环信息中记录的“可约回边”；
  /// 若不属于，则判定 CFG 是不可约的。
  /// 示例：对于下方的典型不可约 CFG（A->B、A->C、B->C、C->B、C->D），
  /// 循环信息中不会记录任何循环，因此当检查 B 与 C 之间的双向回边时，函数会返回“不可约”。
  ///
  /// （不可约 CFG 结构示意图）
  ///    A
  ///  /   \
  /// B<- ->C  （B 与 C 之间存在双向回边，构成不可约结构）
  ///       |
  ///       D
  ///
  template <class NodeT, class RPOTraversalT, class LoopInfoT,
            class GT = GraphTraits<NodeT>>  // 模板参数：节点类型、遍历类型、循环信息类型、图工具（默认 GraphTraits<NodeT>）
  bool containsIrreducibleCFG(RPOTraversalT &RPOTraversal, const LoopInfoT &LI) {
    /// 辅助lambda函数：根据 LI 判断边（\p Src, \p Dst）是否为“可约循环回边”。
    /// 判定逻辑：是否存在一个包含 Src 的循环，且该循环的“循环头”是 Dst。
    auto isProperBackedge = [&](NodeT Src, NodeT Dst) {
      // 从 Src 所在的最内层循环开始，逐层向上检查父循环
      for (const auto *Lp = LI.getLoopFor(Src); Lp; Lp = Lp->getParentLoop()) {
        if (Lp->getHeader() == Dst)  // 若当前循环的头是 Dst，则为可约回边
          return true;
      }
      return false;  // 未找到匹配的循环，不是可约回边
    };

    SmallPtrSet<NodeT, 32> Visited;  // 记录已遍历的节点（避免重复处理，初始容量32）
    // 遍历反向后序序列中的每个节点
    for (NodeT Node : RPOTraversal) {
      Visited.insert(Node);  // 标记当前节点为已遍历
      // 遍历当前节点的所有后继节点（通过 GT::child_begin/child_end 获取后继）
      for (NodeT Succ : make_range(GT::child_begin(Node), GT::child_end(Node))) {
        // 若后继节点未被遍历，则跳过（非回边，无需检查）
        if (!Visited.count(Succ))
          continue;
        // 若后继节点已被遍历，则当前边（Node->Succ）是回边：
        // 检查该回边是否为“可约回边”，若不是则 CFG 不可约，直接返回 true
        if (!isProperBackedge(Node, Succ))
          return true;
      }
    }

    // 遍历完成后未发现不可约回边，返回 false（CFG 是可约的）
    return false;
  }

  }

  #endif
  ```
- CFG.cpp:
  ```cpp
  #include "llvm/Analysis/CFG.h"
  #include "llvm/Analysis/LoopInfo.h"
  #include "llvm/IR/Dominators.h"
  #include "llvm/Support/CommandLine.h"

  using namespace llvm;

  // 可达性分析中探索的最大基本块数量上限。
  // 保持较小值以限制编译时间，尤其适用于该分析的客户端（如捕获跟踪）反复调用的场景。
  static cl::opt<unsigned> DefaultMaxBBsToExplore(
      "dom-tree-reachability-max-bbs-to-explore", cl::Hidden,
      cl::desc("Max number of BBs to explore for reachability analysis"), // 可达性分析中探索的最大基本块数量
      cl::init(32));  // 默认值为32

  /// FindFunctionBackedges - 分析指定函数，找出函数中所有的循环回边并返回。
  /// 相较于计算支配树和循环信息，此分析的开销较低。
  ///
  /// 分析结果将以 <起点块, 终点块> 的形式添加到 Result 中。
  void llvm::FindFunctionBackedges(const Function &F,
      SmallVectorImpl<std::pair<const BasicBlock*,const BasicBlock*> > &Result) {
    const BasicBlock *BB = &F.getEntryBlock();  // 获取函数的入口基本块
    if (succ_empty(BB))  // 若入口块没有后继，则直接返回（无循环回边）
      return;

    SmallPtrSet<const BasicBlock*, 8> Visited;  // 记录已访问的基本块
    // 访问栈：存储 <当前块, 待处理的后继迭代器>，用于深度优先遍历
    SmallVector<std::pair<const BasicBlock *, const_succ_iterator>, 8> VisitStack;
    SmallPtrSet<const BasicBlock*, 8> InStack;  // 记录当前遍历路径中的块（用于检测回边）

    Visited.insert(BB);  // 标记入口块为已访问
    VisitStack.push_back(std::make_pair(BB, succ_begin(BB)));  // 初始化访问栈
    InStack.insert(BB);  // 将入口块加入当前路径

    do {
      std::pair<const BasicBlock *, const_succ_iterator> &Top = VisitStack.back();
      const BasicBlock *ParentBB = Top.first;  // 当前处理的块
      const_succ_iterator &I = Top.second;     // 当前待处理的后继迭代器

      bool FoundNew = false;  // 是否找到未访问的后继块
      while (I != succ_end(ParentBB)) {
        BB = *I++;  // 取当前后继块并移动迭代器
        // 若该块未被访问过
        if (Visited.insert(BB).second) {
          FoundNew = true;
          break;
        }
        // 若该块在当前路径中（InStack中），则构成回边
        if (InStack.count(BB))
          Result.push_back(std::make_pair(ParentBB, BB));
      }

      if (FoundNew) {
        // 发现未访问的块，深入遍历：将其加入当前路径和访问栈
        InStack.insert(BB);
        VisitStack.push_back(std::make_pair(BB, succ_begin(BB)));
      } else {
        // 无新块可访问，回溯：从当前路径移除并弹出栈顶
        InStack.erase(VisitStack.pop_back_val().first);
      }
    } while (!VisitStack.empty());  // 栈为空时遍历结束
  }

  /// GetSuccessorNumber - 搜索基本块 BB 的指定后继块 Succ，
  /// 返回其在终止指令的后继列表中的索引位置。
  /// 注意：若 Succ 不是 BB 的后继，则调用此函数属于错误。
  unsigned llvm::GetSuccessorNumber(const BasicBlock *BB,
      const BasicBlock *Succ) {
    const Instruction *Term = BB->getTerminator();  // 获取基本块的终止指令（分支/跳转指令）
  #ifndef NDEBUG
    unsigned e = Term->getNumSuccessors();  // 调试模式下获取后继总数（用于断言）
  #endif
    for (unsigned i = 0; ; ++i) {
      assert(i != e && "未找到指定的边！");  // 确保不越界
      if (Term->getSuccessor(i) == Succ)
        return i;  // 返回匹配的索引
    }
  }

  /// isCriticalEdge - 若指定的边是临界边，则返回 true。
  /// 临界边定义：从具有多个后继的块到具有多个前驱的块的边。
  bool llvm::isCriticalEdge(const Instruction *TI, unsigned SuccNum,
                            bool AllowIdenticalEdges) {
    assert(SuccNum < TI->getNumSuccessors() && "无效的边索引！");
    return isCriticalEdge(TI, TI->getSuccessor(SuccNum), AllowIdenticalEdges);
  }

  bool llvm::isCriticalEdge(const Instruction *TI, const BasicBlock *Dest,
                            bool AllowIdenticalEdges) {
    assert(TI->isTerminator() && "必须是终止指令才能有后继！");
    if (TI->getNumSuccessors() == 1)  // 只有一个后继的块不可能产生临界边
      return false;

    // 断言：TI 所在的块确实是 Dest 的前驱（确保边存在）
    assert(is_contained(predecessors(Dest), TI->getParent()) &&
          "TI 所在块与 Dest 之间不存在边。");

    const_pred_iterator I = pred_begin(Dest), E = pred_end(Dest);

    // 若 Dest 有多个前驱，则可能是临界边
    assert(I != E && "无前置块，但存在指向该块的边？");
    const BasicBlock *FirstPred = *I;
    ++I;  // 跳过 TI 所在块的前驱（因为当前边就是 TI->Dest）

    if (!AllowIdenticalEdges)
      return I != E;  // 若还有其他前驱，则是临界边

    // 若 AllowIdenticalEdges 为 true，则仅当所有前驱都来自同一块时才不是临界边
    for (; I != E; ++I)
      if (*I != FirstPred)
        return true;  // 存在不同的前驱，是临界边
    return false;
  }

  // 辅助函数：LoopInfo 包含从基本块到其最内层循环的映射。
  // 查找包含 BB 的循环嵌套中最外层的循环。
  static const Loop *getOutermostLoop(const LoopInfo *LI, const BasicBlock *BB) {
    const Loop *L = LI->getLoopFor(BB);  // 获取 BB 所在的最内层循环
    return L ? L->getOutermostLoop() : nullptr;  // 返回最外层循环（若无则返回空）
  }

  bool llvm::isPotentiallyReachableFromMany(
      SmallVectorImpl<BasicBlock *> &Worklist, const BasicBlock *StopBB,
      const SmallPtrSetImpl<BasicBlock *> *ExclusionSet, const DominatorTree *DT,
      const LoopInfo *LI) {
    // 若终止块不可达，则它被所有块支配，无论是否存在路径
    if (DT && !DT->isReachableFromEntry(StopBB))
      DT = nullptr;

    // 若存在排除块，且可能位于支配路径中，则禁用支配树优化
    if (ExclusionSet && !ExclusionSet->empty())
      DT = nullptr;

    // 通常循环中的任何块都可从循环内其他块到达，但排除块可能分割循环体
    // 记录有"漏洞"（含排除块）的循环
    SmallPtrSet<const Loop *, 8> LoopsWithHoles;
    if (LI && ExclusionSet) {
      for (auto *BB : *ExclusionSet) {
        if (const Loop *L = getOutermostLoop(LI, BB))
          LoopsWithHoles.insert(L);
      }
    }

    // 获取终止块所在的最外层循环
    const Loop *StopLoop = LI ? getOutermostLoop(LI, StopBB) : nullptr;

    unsigned Limit = DefaultMaxBBsToExplore;  // 探索块数量上限
    SmallPtrSet<const BasicBlock*, 32> Visited;  // 已访问的块

    do {
      BasicBlock *BB = Worklist.pop_back_val();  // 从工作列表取一个块
      if (!Visited.insert(BB).second)  // 若已访问则跳过
        continue;
      if (BB == StopBB)  // 找到终止块，返回可达
        return true;
      if (ExclusionSet && ExclusionSet->count(BB))  // 若为排除块则跳过
        continue;
      if (DT && DT->dominates(BB, StopBB))  // 若 BB 支配终止块，则可达
        return true;

      const Loop *Outer = nullptr;
      if (LI) {
        Outer = getOutermostLoop(LI, BB);  // 获取当前块所在的最外层循环
        // 若循环有漏洞（含排除块），则不能假设循环内块相互可达，需清除 Outer 以处理后继
        if (LoopsWithHoles.count(Outer))
          Outer = nullptr;
        // 若当前块与终止块在同一最外层循环，则可达
        if (StopLoop && Outer == StopLoop)
          return true;
      }

      if (!--Limit) {  // 达到探索上限，保守返回可达
        // 无法确定是否可达，保守起见返回 true（可能存在路径）
        return true;
      }

      if (Outer) {
        // 同一循环内的块相互可达，可直接跳至循环出口块（优化遍历）
        Outer->getExitBlocks(Worklist);
      } else {
        // 无有效循环信息，添加所有后继块继续遍历
        Worklist.append(succ_begin(BB), succ_end(BB));
      }
    } while (!Worklist.empty());  // 工作列表为空时结束

    // 已穷尽所有可能路径，确定不可达
    return false;
  }

  bool llvm::isPotentiallyReachable(
      const BasicBlock *A, const BasicBlock *B,
      const SmallPtrSetImpl<BasicBlock *> *ExclusionSet, const DominatorTree *DT,
      const LoopInfo *LI) {
    assert(A->getParent() == B->getParent() && "此分析仅适用于同一函数内！");

    if (DT) {
      // 若 A 可达但 B 不可达，则返回 false
      if (DT->isReachableFromEntry(A) && !DT->isReachableFromEntry(B))
        return false;
      // 若无排除块，可通过支配关系快速判断
      if (!ExclusionSet || ExclusionSet->empty()) {
        // 若 A 是入口块且 B 可达，则 A 到 B 一定可达
        if (A->isEntryBlock() && DT->isReachableFromEntry(B))
          return true;
        // 若 B 是入口块且 A 可达，则 A 到 B 一定不可达（入口块是起点）
        if (B->isEntryBlock() && DT->isReachableFromEntry(A))
          return false;
      }
    }

    // 初始化工作列表，从 A 开始遍历
    SmallVector<BasicBlock*, 32> Worklist;
    Worklist.push_back(const_cast<BasicBlock*>(A));

    return isPotentiallyReachableFromMany(Worklist, B, ExclusionSet, DT, LI);
  }

  bool llvm::isPotentiallyReachable(
      const Instruction *A, const Instruction *B,
      const SmallPtrSetImpl<BasicBlock *> *ExclusionSet, const DominatorTree *DT,
      const LoopInfo *LI) {
    assert(A->getParent()->getParent() == B->getParent()->getParent() &&
          "此分析仅适用于同一函数内！");

    if (A->getParent() == B->getParent()) {
      // 同一基本块内的特殊情况：需判断指令顺序或循环关系
      BasicBlock *BB = const_cast<BasicBlock *>(A->getParent());

      // 若块在循环中，则块内任何指令都可通过回边相互到达
      if (LI && LI->getLoopFor(BB) != nullptr)
        return true;

      // 若 A 在 B 之前（或 A 与 B 是同一指令），则 B 可达
      if (A == B || A->comesBefore(B))
        return true;

      // 入口块不可能在循环中（无前置块），若 A 在 B 之后则不可达
      if (BB->isEntryBlock())
        return false;

      // 否则，从当前块的后继开始遍历
      SmallVector<BasicBlock*, 32> Worklist;
      Worklist.append(succ_begin(BB), succ_end(BB));
      if (Worklist.empty()) {
        // 无后继，证明不可达
        return false;
      }

      // 检查是否可从后继块到达 B 所在的块
      return isPotentiallyReachableFromMany(Worklist, B->getParent(),
                                            ExclusionSet, DT, LI);
    }

    // 不同块的情况：直接检查块之间的可达性
    return isPotentiallyReachable(
        A->getParent(), B->getParent(), ExclusionSet, DT, LI);
  }
  ```

构建 cfg 在 `clang/lib/CodeGen/CodeGenFunction.h` 中完成, 上边的代码提供的功能是为了进行控制流分析