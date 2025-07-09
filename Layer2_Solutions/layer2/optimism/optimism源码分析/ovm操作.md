在我们有了沙箱 OVM，我们需要将智能合约编译为 OVM 字节码。以下是我们的一些选择：

- 创建一种可编译为 OVM 的新智能合约语言：一种新的智能合约语言是一个很容易被忽略的想法，因为它需要从头开始重新做所有事情，我们已经同意我们不会在这里这样做。
- 将 EVM 字节码转译为 OVM 字节码：曾[尝试过](https://github.com/ethereum-optimism/optimism-monorepo/blob/2ca62fb41be6ef69b0c07a1bd5502ac425aaf341/packages/solc-transpiler/src/compiler.ts#L420-L496)但由于复杂性而放弃。
- 通过修改编译器以生成 OVM 字节码来支持 Solidity 和 Vyper。





> https://research.paradigm.xyz/optimism

https://github.com/ethereum-optimism/solidity/blob/df005f39493525b43f1153dff8da5910a2b83e34/libsolidity/codegen/CompilerContext.cpp#L64-L367