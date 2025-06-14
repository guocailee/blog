---
{"dg-publish":true,"permalink":"/Program/Mixed/Clean  Code/","noteIcon":"","created":"2025-03-06T21:28:25.978+08:00"}
---


Code is clean if it can be understood easily – by everyone on the team. Clean code can be read and enhanced by a developer other than its original author. With understandability comes readability, changeability, extensibility and maintainability.
>  简洁的代码应该易于理解 —— 团队中的每个人都能轻松理解。简洁的代码可以被其他开发者阅读和改进，而不仅仅是其原作者。理解性强的代码具有可读性、可变更性、可扩展性和可维护性。
_____________________________________

## General rules
1. Follow standard conventions.
2. Keep it simple stupid. Simpler is always better. Reduce complexity as much as possible.
3. Boy scout rule. Leave the campground cleaner than you found it.
4. Always find root cause. Always look for the root cause of a problem.

## 通用规则
1. 遵循标准规范。
2. 保持简单，越简单越好。尽量减少复杂性。
3. 童子军规则。离开时让营地比你到来时更干净。
4. 始终找到问题的根本原因。总是寻找问题的根本原因。

## Design rules
1. Keep configurable data at high levels.
2. Prefer polymorphism to if/else or switch/case.
3. Separate multi-threading code.
4. Prevent over-configurability.
5. Use dependency injection.
6. Follow Law of Demeter. A class should know only its direct dependencies.

## 设计规则
1. 将可配置的数据放在高层次。
2. 优先使用多态而不是 if/else 或 switch/case。
3. 将多线程代码分离。
4. 防止过度配置。
5. 使用依赖注入。
6. 遵循迪米特法则（最少知识原则）。一个类应该只知道它的直接依赖关系。

## Understandability tips
1. Be consistent. If you do something a certain way, do all similar things in the same way.
2. Use explanatory variables.
3. Encapsulate boundary conditions. Boundary conditions are hard to keep track of. Put the processing for them in one place.
4. Prefer dedicated value objects to primitive type.
5. Avoid logical dependency. Don't write methods which works correctly depending on something else in the same class.
6. Avoid negative conditionals.

## 理解性建议
1. 保持一致性。如果你以某种方式做某事，那么对所有类似的事情都使用相同的方式。
2. 使用解释性变量。
3. 封装边界条件。边界条件很难跟踪，应将其处理集中在一个地方。
4. 优先使用专用的值对象而不是原始类型。
5. 避免逻辑依赖。不要编写依赖于同一个类中其他部分的方法。
6. 避免负面条件。

## Names rules
1. Choose descriptive and unambiguous names.
2. Make meaningful distinction.
3. Use pronounceable names.
4. Use searchable names.
5. Replace magic numbers with named constants.
6. Avoid encodings. Don't append prefixes or type information.

## 命名规则
1. 选择描述性和明确的名称。
2. 具有有意义的区分。
3. 使用可发音的名称。
4. 使用可搜索的名称。
5. 用命名常量代替魔术数字。
6. 避免编码。不要添加前缀或类型信息。

## Functions rules
1. Small.
2. Do one thing.
3. Use descriptive names.
4. Prefer fewer arguments.
5. Have no side effects.
6. Don't use flag arguments. Split method into several independent methods that can be called from the client without the flag.

## 函数规则
1. 保持简短。
2. 只做一件事。
3. 使用描述性名称。
4. 优先使用更少的参数。
5. 不产生副作用。
6. 不要使用标志参数。将方法分成几个独立的方法，客户端可以不使用标志来调用。

## Comments rules
1. Always try to explain yourself in code.
2. Don't be redundant.
3. Don't add obvious noise.
4. Don't use closing brace comments.
5. Don't comment out code. Just remove.
6. Use as explanation of intent.
7. Use as clarification of code.
8. Use as warning of consequences.

## 注释规则
1. 始终尽量通过代码来解释自己。
2. 不要重复。
3. 不要添加明显的噪音。
4. 不要使用闭合括号注释。
5. 不要注释掉代码。直接删除。
6. 用于解释意图。
7. 用于澄清代码。
8. 用于警告后果。

## Source code structure
1. Separate concepts vertically.
2. Related code should appear vertically dense.
3. Declare variables close to their usage.
4. Dependent functions should be close.
5. Similar functions should be close.
6. Place functions in the downward direction.
7. Keep lines short.
8. Don't use horizontal alignment.
9. Use white space to associate related things and disassociate weakly related.
10. Don't break indentation.

## 源代码结构
1. 垂直分离概念。
2. 相关的代码应垂直密集出现。
3. 在使用附近声明变量。
4. 相关的函数应靠近。
5. 相似的函数应靠近。
6. 将函数放在向下的方向。
7. 保持行简短。
8. 不要使用水平对齐。
9. 使用空白来关联相关的事物并分离弱相关的事物。
10. 不要破坏缩进。

## Objects and data structures
1. Hide internal structure.
2. Prefer data structures.
3. Avoid hybrids structures (half object and half data).
4. Should be small.
5. Do one thing.
6. Small number of instance variables.
7. Base class should know nothing about their derivatives.
8. Better to have many functions than to pass some code into a function to select a behavior.
9. Prefer non-static methods to static methods.

## 对象和数据结构
1. 隐藏内部结构。
2. 优先使用数据结构。
3. 避免混合结构（半对象半数据）。
4. 应该小而简单。
5. 只做一件事。
6. 实例变量的数量应少。
7. 基类不应了解其派生类。
8. 比起传递代码到函数中选择行为，更好的是有多个函数。
9. 优先使用非静态方法而不是静态方法。

## Tests
1. One assert per test.
2. Readable.
3. Fast.
4. Independent.
5. Repeatable.

## 测试
1. 每个测试只包含一个断言。
2. 可读性强。
3. 运行速度快。
4. 互相独立。
5. 可重复执行。

## Code smells
1. Rigidity. The software is difficult to change. A small change causes a cascade of subsequent changes.
2. Fragility. The software breaks in many places due to a single change.
3. Immobility. You cannot reuse parts of the code in other projects because of involved risks and high effort.
4. Needless Complexity.
5. Needless Repetition.
6. Opacity. The code is hard to understand.

## 代码异味
1. 僵化性。软件难以更改。一个小的更改会引发一连串的后续更改。
2. 脆弱性。软件在多个地方因单个更改而崩溃。
3. 不可移植性。由于涉及的风险和高昂的成本，无法在其他项目中重用部分代码。
4. 无谓的复杂性。
5. 无谓的重复。
6. 模糊性。代码难以理解。
