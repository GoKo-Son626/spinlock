<!--
 * @Date: 2025-03-26
 * @LastEditors: GoKo-Son626
 * 
 * @LastEditTime: 2025-03-26
 * @FilePath: /qspinlock/CSR.md
 * @Description: 
-->
主要是为 RISC-V 架构 添加 Sxcsrind ISA 扩展 的 CSR（Control and Status Register）定义。

**Sxcsrind 是一个扩展 CSR 访问的 ISA 机制，用于 间接访问 M/S/VS 模式的 CSR。**
主要添加了新的 CSR 定义：

这些 CSR 允许 间接访问（Indirect Access）M-mode（机器模式）、S-mode（监督模式）和 VS-mode（虚拟监督模式）的任意 CSR。

CSR包括：
CSR_SIREG2, CSR_SIREG3, CSR_SIREG4, CSR_SIREG5, CSR_SIREG6
CSR_VSIREG2, CSR_VSIREG3, CSR_VSIREG4, CSR_VSIREG5, CSR_VSIREG6
CSR_MIREG2, CSR_MIREG3, CSR_MIREG4, CSR_MIREG5, CSR_MIREG6
这些 CSR 提供了一种标准方式来在不同的特权模式下间接访问寄存器，增强了灵活性和可扩展性。

- RISC-V 的 Sxcsrind ISA 扩展 主要是为了提供 更灵活的 CSR 访问方式，减少直接操作 CSR 的指令集开销，提高系统在复杂应用场景下的适应性。
- 例如，在虚拟化环境或安全相关场景下，某些 CSR 可能不允许直接访问，因此通过 间接访问机制 使得系统更易于管理和控制。