# HanXiongnuWars Trial Bypass

《戎马丹心：汉匈决战》（Gloria Sinica: Han Xiongnu Wars）Wine/Linux 试用模式破解补丁。

## 问题

汉匈决战基于战团引擎（Warband Engine 0.910），启动时会弹出序列号验证窗口，试用模式下角色等级被限制在 8 级。

## 补丁内容

对 `HanXiongnuWars.exe`（v2.725）进行了以下二进制补丁：

| # | 文件偏移 | 原始字节 | 补丁字节 | 说明 |
|---|---------|---------|---------|------|
| 1 | `0x207067` | `0f 85 80 00 00 00` | `90 e9 80 00 00 00` | 等级上限弹窗 `jnz` → 无条件跳转 |
| 2 | `0x207e3b` | `74 4e` | `90 90` | 试用模式菜单检测 `jz` → NOP |
| 3 | `0x1b5e93` | `0f 84 69 01 00 00` | `90 90 90 90 90 90` | 试用模式长跳转 → NOP |
| 4 | `0x18b455` | `74 07` | `90 90` | 序列号验证条件跳转 A → NOP |
| 5 | `0x18deab` | `74 07` | `90 90` | 序列号验证条件跳转 B → NOP |
| 6 | `0x1bf51c` | `0f 85 85 0a 00 00` | `90 e9 85 0a 00 00` | 启动序列号弹窗 `jnz` → 无条件跳转 |

所有补丁均针对激活标志位 `0x8eb5a0`（`.data` 段全局变量）的条件分支。

## 使用方法

```bash
# 备份原始文件
cp HanXiongnuWars.exe HanXiongnuWars.exe.bak

# 用补丁文件替换
cp patch/HanXiongnuWars.exe /path/to/Gloria\ Sinica\ Han\ Xiongnu\ Wars/HanXiongnuWars.exe

# 启动游戏
cd "/path/to/Gloria Sinica Han Xiongnu Wars/"
wine HanXiongnuWars.exe
```

## 效果

- ✅ 启动时不再弹出序列号验证窗口
- ✅ 角色等级突破 8 级限制，可自由升级
- ✅ 游戏其他功能（战斗、菜单、存档）完全正常

## 适用版本

- 汉匈决战 v2.725（基于 Warband Engine 0.910）
- 测试环境：Kali Linux + Wine + DXVK 3.0.2 + NVIDIA GTX 1660 SUPER

## 技术细节

详见 [SKILL.md](SKILL.md)，包含完整的逆向工程方法论，可复用于其他战团引擎独立游戏。
