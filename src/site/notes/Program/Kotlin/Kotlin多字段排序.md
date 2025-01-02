---
{"dg-publish":true,"permalink":"/Program/Kotlin/Kotlin多字段排序/","noteIcon":""}
---

### 利用`pair` 进行排序
在 Kotlin 中，可以通过 `Pair` 对象的内置比较方式来进行排序。`Pair` 实际上实现了 `Comparable` 接口，因此它可以直接进行排序。`Pair` 会首先比较第一个元素（`first`），如果第一个元素相等，再比较第二个元素（`second`）。

#### 代码示例：根据 `Pair` 排序

假设你有一个 `List<Pair<Int, String>>`，你想根据 `Pair` 的第一个元素进行排序，若第一个元素相等，则根据第二个元素进行排序。

```kotlin
fun main() {
    val list = listOf(
        Pair(3, "apple"),
        Pair(1, "banana"),
        Pair(3, "cherry"),
        Pair(2, "date")
    )

    // 根据 Pair 的第一个元素排序，如果第一个元素相同，按第二个元素排序
    val sortedList = list.sorted()

    println(sortedList)
}
```

#### 解释

- `sorted()` 会默认按 `Pair` 的第一个元素 (`first`) 排序，如果 `first` 相等，才会按 `second` 排序。
- 因为 `Pair` 已经实现了 `Comparable` 接口，所以 `sorted()` 直接适用于 `Pair` 的排序。

#### 输出

```text
[(1, banana), (2, date), (3, apple), (3, cherry)]
```

### 自定义排序

如果你想要自定义排序的方式（例如，先按 `second` 排序，再按 `first` 排序），你可以使用 `sortedWith` 配合自定义比较器：
#### 代码示例：根据 `Compare` 排序

```kotlin
fun main() {
    val list = listOf(
        Pair(3, "apple"),
        Pair(1, "banana"),
        Pair(3, "cherry"),
        Pair(2, "date")
    )

    // 按照 Pair 的第二个元素排序，如果相同则按第一个元素排序
    val sortedList = list.sortedWith(compareBy({ it.second }, { it.first }))

    println(sortedList)
}
```

#### 解释

- `compareBy` 用于创建一个按多个条件排序的比较器。
- `it.second` 是首先按 `second` 排序，如果相同再按 `first` 排序。

#### 输出

```bash
[(1, banana), (3, apple), (3, cherry), (2, date)]
```

这种方式为你提供了更灵活的排序选项，适用于各种排序需求。