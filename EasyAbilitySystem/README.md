# EasyAbilitySystem 设计

目标是
1. 简化版的 GAS
2. 不包括网络同步
3. 纯蓝图，无需新建 C++
4. 方便配表

## AttributeSet

由于项目不大，使用多个 `AttributeSet` 的情况比较小，所以直接定义全局使用同一个 `AttributeSet`

为了方便蓝图使用，将属性定义为枚举使用，在 `EasyAbilitySystemSettings` 中配置定义属性的蓝图枚举


