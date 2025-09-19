---
title: macOS 一键切换 Karabiner Profile 和输入法
date: 2025-09-20 01:55:19
permalink: one-key-switch-karabine-profile-on-macos
tags:
  - Tips
---

## 为什么

我的电脑配置了交换 Command 键和 Control 键，同时使用 Programmer Dvorak 键位，就导致这是一台何晨光的电脑，谁都别想用，在我一个人使用的时候固然是非常方便的，但日常工作免不了问大佬问题，大佬可能直接上手操作电脑解决，这时候就尴尬了，才要慌乱的调整键位。

我用的键位映射软件是 Karabiner-Elements，软件本身就支持 Profile 的功能，允许在多个配置之间切换，我们只需要额外增加一个空白的 Profile 在需要的时候切换过去就好，然后再手动切换输入法。

但我们可以做到一键切换。

## 配置方式

在 Karabiner 中设置好我们自己定制的 Profile，我这里命名为 `ikvarxt`，原始没有任何改动的 Profile，命名为 `original`，我们需要在这两个 Profile 的 Complex Modifications 栏下，增加如下的自定义配置：

```json
{
  "description": "Switch between ikvarxt and original",
  "manipulators": [
    {
      "type": "basic",
      /* 设置一个 flag 标记,用于单按键切换 Profile */
      "conditions": [
        {
          "type": "variable_if",
          "name": "input_source switched",
          "value": 1
        }
      ],
      "from": {
        /* 我这里设置为按左下角的 global 键切换 */
        "apple_vendor_top_case_key_code": "keyboard_fn"
      },
      "to": [
        {
          "select_input_source": {
            /* 切换到 Programmer Dvorak 输入法 */
            "input_source_id": "com.apple.keyboardlayout.Programmer Dvorak",
            "language": "^en$"
          }
        },
        {
          "set_variable": {
            "name": "input_source switched",
            "value": 0
          }
        }
      ],
      "to_after_key_up": [
        /* 通过 Karabiner 的 cli 操作切换 Profile，主要改为你的 Profile 名称 */
        {
          "shell_command": "'/Library/Application Support/org.pqrs/Karabiner-Elements/bin/karabiner_cli' --select-profile 'ikvarxt'"
        }
      ]
    },
    {
      "type": "basic",
      "from": {
        "apple_vendor_top_case_key_code": "keyboard_fn"
      },
      "to": [
        {
          "select_input_source": {
            /* 切换为 ABC 输入法 */
            "input_source_id": "com.apple.keylayout.ABC",
            "language": "^en$"
          }
        },
        {
          "set_variable": {
            "name": "input_source switched",
            "value": 1
          }
        }
      ],
      "to_after_key_up": [
        {
          "shell_command": "'/Library/Application Support/org.pqrs/Karabiner-Elements/bin/karabiner_cli' --select-profile 'original'"
        }
      ]
    }
  ]
}
```

有几个需要注意的点：
1. 两个 Profile 都需要创建 Complex Modifications
2. 当前在使用的 input_source_id 可以通过 Karabiner-EventViewer 的 Variables 标签页查看
3. 按键的 keycode 也可以在 Karabiner-EventViewer 中查看，注意要复制完整的 keycode json 值替换 from 字段下的内容
3. 注意更改 karabiner_cli 命令的 Profile 参数为你的 Profile 名称

## 参考

- [Switching Between Profiles and Input Sources](https://www.reddit.com/r/Karabiner/comments/10xuiqb/switching_between_profiles_and_input_sources/)

