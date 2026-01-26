---
{"dg-publish":true,"permalink":"/Program/Godot/UI 系统/","noteIcon":"","created":"2026-01-24T11:47:43.799+08:00"}
---

# UI 系统

Godot 的 UI 系统基于 Control 节点，使用锚点和边距进行布局。

## Control 节点基础

所有 UI 节点都继承自 `Control`。

### 基本属性

```gdscript
extends Control

# 位置和大小
position = Vector2(100, 100)
size = Vector2(200, 50)

# 锚点（控制相对位置）
anchors_preset = PRESET_CENTER

# 边距
offset_left = 0
offset_right = 100
offset_top = 0
offset_bottom = 50
```

## 常用 UI 节点

### Label (标签)

```gdscript
extends Label

func _ready() -> void:
    text = "Hello, World!"
    horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
    vertical_alignment = VERTICAL_ALIGNMENT_CENTER
    font_size = 24
```

### Button (按钮)

```gdscript
extends Button

func _ready() -> void:
    text = "Click Me"

func _pressed() -> void:
    print("Button pressed!")
```

### LineEdit (文本输入框)

```gdscript
extends LineEdit

signal text_submitted(new_text: String)

func _ready() -> void:
    placeholder_text = "Enter your name..."

func _on_text_submitted(new_text: String) -> void:
    print("Entered: ", new_text)
    text_submitted.emit(new_text)
```

### ProgressBar (进度条)

```gdscript
extends ProgressBar

var current_value: float = 0.0
var max_value: float = 100.0

func update_progress(value: float) -> void:
    current_value = clamp(value, 0, max_value)
    value = (current_value / max_value) * 100
```

### TextureRect (图像显示)

```gdscript
extends TextureRect

func _ready() -> void:
    texture = preload("res://images/icon.png")
    expand_mode = TextureRect.EXPAND_FIT_WIDTH_PROPORTIONAL
```

### CheckBox (复选框)

```gdscript
extends CheckBox

signal toggled(is_checked: bool)

func _toggled(button_pressed: bool) -> void:
    print("Check box is: ", button_pressed)
```

## 布局容器

### VBoxContainer (垂直布局)

```gdscript
extends VBoxContainer

func _ready() -> void:
    separation = 10  # 子元素间距
    alignment = ALIGNMENT_CENTER
```

### HBoxContainer (水平布局)

```gdscript
extends HBoxContainer

func _ready() -> void:
    separation = 10
    alignment = ALIGNMENT_CENTER
```

### GridContainer (网格布局)

```gdscript
extends GridContainer

func _ready() -> void:
    columns = 3
    # 按行添加子节点，会自动换行
```

### MarginContainer (边距容器)

```gdscript
extends MarginContainer

func _ready() -> void:
    add_theme_constant_override("margin_left", 20)
    add_theme_constant_override("margin_right", 20)
    add_theme_constant_override("margin_top", 10)
    add_theme_constant_override("margin_bottom", 10)
```

### CenterContainer (居中容器)

```gdscript
extends CenterContainer

func _ready() -> void:
    # 子元素自动居中
```

## 锚点系统

锚点控制 UI 元素相对于父容器的位置。

### 锚点预设

```gdscript
# 全屏填充
anchors_preset = PRESET_FULL_RECT

# 左上角
anchors_preset = PRESET_TOP_LEFT

# 居中
anchors_preset = PRESET_CENTER

# 左居中
anchors_preset = PRESET_CENTER_LEFT

# 底部全宽
anchors_preset = PRESET_BOTTOM_WIDE
```

## UI 主题

### 使用内置主题

```gdscript
extends Control

func _ready() -> void:
    # 使用默认主题
    theme = Theme.new()
```

### 创建自定义主题

```gdscript
extends Control

@export var custom_theme: Theme

func _ready() -> void:
    theme = custom_theme
```

## UI 与 3D 游戏结合

### HUD (Heads-Up Display)

```gdscript
extends Control

@onready var health_bar = $HealthBar
@onready var score_label = $ScoreLabel

func update_health(current: int, max_value: int) -> void:
    health_bar.value = (float(current) / max_value) * 100

func update_score(score: int) -> void:
    score_label.text = "Score: " + str(score)
```

### 将 UI 覆盖在 3D 场景上

```gdscript
# Main 节点结构
# - World3D (Node3D)
#   - Camera3D
# - UI (Control) - 添加为同级，设置为 top level
```

## 对话系统示例

```gdscript
extends Control

@onready var dialog_label = $DialogBox/Label
@onready var continue_button = $DialogBox/Button

var dialogues: Array[String] = [
    "Welcome to the game!",
    "Use WASD to move.",
    "Press Space to jump."
]
var current_index: int = 0

func start_dialogue() -> void:
    visible = true
    show_dialogue()

func show_dialogue() -> void:
    if current_index < dialogues.size():
        dialog_label.text = dialogues[current_index]
    else:
        end_dialogue()

func _on_continue_button_pressed() -> void:
    current_index += 1
    show_dialogue()

func end_dialogue() -> void:
    visible = false
```

## UI 动画

### 使用 Tween 缓动

```gdscript
extends Control

var tween: Tween

func fade_in() -> void:
    tween = create_tween()
    tween.tween_property(self, "modulate:a", 1.0, 0.5)

func fade_out() -> void:
    tween = create_tween()
    tween.tween_property(self, "modulate:a", 0.0, 0.5)
```

## 响应式设计

```gdscript
extends Control

func _ready() -> void:
    # 根据屏幕大小调整
    var screen_size = get_viewport_rect().size
    if screen_size.x < 800:
        scale_ui_for_mobile()

func scale_ui_for_mobile() -> void:
    $HealthBar.scale = Vector2(0.8, 0.8)
    $ScoreLabel.add_theme_font_size_override("font_size", 20)
```

## 鼠标光标

```gdscript
extends Control

func _ready() -> void:
    # 隐藏默认光标
    Input.set_mouse_mode(Input.MOUSE_MODE_HIDDEN)

    # 显示自定义光标
    Input.set_custom_mouse_cursor(preload("res://cursors/custom.png"))
```

## 虚拟摇杆（移动端）

```gdscript
extends Control

var touch_start: Vector2
var is_dragging: bool = false

signal joystick_moved(direction: Vector2)

func _input(event: InputEvent) -> void:
    if event is InputEventScreenTouch:
        if event.pressed:
            touch_start = event.position
            is_dragging = true
        else:
            is_dragging = false
            joystick_moved.emit(Vector2.ZERO)

    elif event is InputEventScreenDrag and is_dragging:
        var direction = (event.position - touch_start).normalized()
        joystick_moved.emit(direction)
```
