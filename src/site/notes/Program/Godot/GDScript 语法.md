---
{"dg-publish":true,"permalink":"/Program/Godot/GDScript 语法/","noteIcon":"","created":"2026-01-24T11:47:43.799+08:00"}
---

# GDScript 语法

GDScript 是 Godot 的官方脚本语言，语法类似 Python。

## 基本语法

### 变量声明

```gdscript
var name = "Player"        # 类型推断
var health: int = 100     # 显式类型
var speed: float = 5.0    # 浮点数
var is_active: bool = true
```

### 常量

```gdscript
const MAX_SPEED = 10.0
const GRAVITY = 9.8
```

## 数据类型

### 基础类型

```gdscript
var integer: int = 42
var floating: float = 3.14
var text: String = "Hello"
var boolean: bool = true
```

### 数组

```gdscript
var numbers = [1, 2, 3, 4, 5]
var names: Array[String] = ["Alice", "Bob"]  # 类型化数组
```

### 字典

```gdscript
var player_data = {
    "name": "Hero",
    "level": 5,
    "exp": 100
}
```

## 控制流

### 条件语句

```gdscript
if health > 0:
    print("Alive")
elif health == 0:
    print("Dead")
else:
    print("Invalid health")
```

### 循环

```gdscript
# for 循环
for i in range(10):
    print(i)

for item in array:
    print(item)

# while 循环
while health > 0:
    take_damage()
```

## 函数

```gdscript
# 简单函数
func greet(name: String) -> void:
    print("Hello, " + name)

# 带返回值的函数
func calculate_damage(base: int, multiplier: float) -> int:
    return int(base * multiplier)

# 可选参数
func spawn_enemy(count: int = 1) -> void:
    for i in range(count):
        create_enemy()
```

## 类与继承

```gdscript
extends CharacterBody2D  # 继承

class_name Player  # 定义类名，可用于类型提示

# 类变量
var speed: float = 200.0
var health: int = 100

# 虚函数重写
func _ready() -> void:
    print("Player is ready!")

func _process(delta: float) -> void:
    movement()
```

## 信号

```gdscript
# 定义信号
signal health_changed(new_health: int)
signal died

# 发射信号
signal health_changed.emit(health)
signal died.emit()

# 连接信号
ready():
    health_changed.connect(_on_health_changed)

func _on_health_changed(new_health: int) -> void:
    print("Health is now: ", new_health)
```

## 注解

```gdscript
@export var speed: float = 5.0  # 在编辑器中可编辑
@onready var enemy = get_node("Enemy")  # 等待节点 ready
```

## 向量运算

```gdscript
var position := Vector2(0, 0)
var direction := Vector2(1, 0)

position += direction * speed * delta
var distance := position.distance_to(target)
var angle := position.angle_to(target)
```

## 类型转换

```gdscript
var text = "123"
var number = int(text)  # 字符串转整数
var float_num = float(text)
```

## 常用内置函数

```gdscript
# 数学函数
abs(x), sqrt(x), pow(base, exponent)
sin(x), cos(x), tan(x)
min(a, b), max(a, b)
clamp(value, min, max)

# 类型检查
is_instance_of(node, Node)
typeof(value)
```
