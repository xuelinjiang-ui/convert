# 技术设计说明

## 1. 设计目标

本项目的目标是将多种明文和加密音频格式统一转换为 `mp3`，并尽量保持以下特性：

- 用户入口统一
- 各格式转换流程自动分发
- 尽可能本地实现解密，不依赖外部黑盒 EXE
- 对必须依赖官方客户端的格式，使用最小化宿主或桥接方案实现
- 在 `net472` 下稳定运行

## 2. 当前工程结构

### 2.1 主工程

- `WpfApp1`

职责：

- 用户界面
- 格式判断
- 统一转换编排
- 调用各解密服务和 `ffmpeg`

### 2.2 QQ 解密链路

- `QQMusicDecryptRunner`
- `QQMusicDecryptHook`

职责：

- 自动连接或启动 QQ 音乐
- 通过 EasyHook 注入到 QQ 音乐进程
- 在目标进程内调用 `QQMusicCommon.dll` 获取解密后的音频流

### 2.3 酷狗桥接链路

- `KugouInfraRunner`

职责：

- 以 `x86` 进程加载酷狗 `infra.dll`
- 读取 `KGMusicV3.db`
- 查询 `EncryptionKey`
- 将密钥返回主程序

### 2.4 第三方依赖目录

- `ThirdParty\EasyHook`
- `ThirdParty\Compat`

职责：

- 提供 EasyHook 本地二进制
- 提供 `System.Memory`、`System.Buffers`、`System.Runtime.CompilerServices.Unsafe`、`System.Numerics.Vectors` 等兼容依赖

## 3. 框架与位数设计

### 3.1 主程序

- `net472`
- `AnyCPU`
- `Prefer32Bit=false`

效果：

- 在 64 位 Windows 上按 64 位进程运行

### 3.2 QQ 辅助工程

- `QQMusicDecryptRunner`：`net472`
- `QQMusicDecryptHook`：`net472`

### 3.3 酷狗桥接工程

- `KugouInfraRunner`：`net472 + x86`

这样设计的原因是：

- 主程序需要保持现代桌面环境下的 64 位可用性
- 但用户机器上的酷狗 `infra.dll` 常见为 `x86`
- 64 位进程无法直接 `LoadLibrary` 32 位 DLL
- 因此使用单独的 `x86` runner 完成这一步

## 4. 主转换流程

主转换入口位于 `AudioConverterService`。

基本流程：

1. 校验输入文件存在
2. 识别扩展名
3. 根据扩展名分派到不同转换分支
4. 必要时先生成临时解密文件
5. 若中间结果不是 `mp3`，则调用 `ffmpeg` 转码
6. 将结果输出到桌面
7. 清理临时目录

涉及文件：

- `WpfApp1/Services/AudioConverterService.cs`
- `WpfApp1/Services/FfmpegConverter.cs`

## 5. 各格式实现逻辑

### 5.1 `flac` / `ogg`

实现方式：

- 直接通过 `ffmpeg` 转码

对应模块：

- `FfmpegConverter`

### 5.2 `ncm`

实现方式：

- 解析 NCM 魔数头
- 读取并异或 key block
- 用 AES-ECB 解出核心密钥
- 构建 RC4-like key box
- 解析 metadata block，提取格式信息
- 跳过封面数据
- 解密音频流并输出为临时 `mp3` 或 `flac`
- 如结果不是 `mp3`，再转码

对应模块：

- `WpfApp1/Services/NcmDecoder.cs`

### 5.3 `kwm`

实现方式：

- 校验 `yeelion-kuwo-tme` 文件头
- 扫描重复 key block
- 提取或恢复 32 字节密钥
- 循环异或解密正文
- 探测真实明文格式
- 如不是 `mp3`，再调用 `ffmpeg`

对应模块：

- `WpfApp1/Services/KwmDecryptService.cs`

### 5.4 `kgma`

实现方式：

- 校验酷狗加密头
- 识别 `mode 3`
- 从头部提取 `cryptoKey`
- 通过 `MD5` 派生 `slotBox` 和 `fileBox`
- 用 `fileBox + slotBox + offset collapse` 规则逐字节解密
- 探测输出格式
- 如不是 `mp3`，再转码

对应模块：

- `WpfApp1/Services/KggDecryptService.cs`

### 5.5 `kgg`

实现方式：

- 校验酷狗加密头
- 识别 `mode 5`
- 从文件头提取 `audioHash`
- 查询酷狗数据库中的 `EncryptionKey`
- 用 `QMC2` 解密器恢复明文音频
- 探测输出格式
- 如不是 `mp3`，再转码

对应模块：

- `WpfApp1/Services/KggDecryptService.cs`
- `KugouInfraRunner/Program.cs`
- `WpfApp1/Services/Qmc2Crypto.cs`

### 5.6 `mgg` / `mflac`

实现方式：

- 主程序拉起 `QQMusicDecryptRunner`
- Runner 查找或启动 QQ 音乐
- 注入 `QQMusicDecryptHook`
- 在 QQ 进程内调用 `QQMusicCommon.dll`
- 读取解密后的明文流
- 探测输出格式
- 如不是 `mp3`，再转码

对应模块：

- `WpfApp1/Services/QqMusicMggDecryptService.cs`
- `QQMusicDecryptRunner/Program.cs`
- `QQMusicDecryptHook/InjectionEntryPoint.cs`
- `QQMusicDecryptHook/QqMusicCommonDecryptor.cs`

## 6. FFmpeg 设计

`ffmpeg` 统一负责非 `mp3` 明文音频向 `mp3` 的最终转码。

参数策略：

- `libmp3lame`
- `192k`
- 保留元数据映射

额外兼容策略：

- 若输入或输出路径含中文或特殊字符，先复制到 ASCII 临时目录再转码

对应模块：

- `WpfApp1/Services/FfmpegConverter.cs`

## 7. QQ 解密设计

### 7.1 为什么需要注入

QQ 音乐加密格式的稳定解密方式依赖 QQ 自身进程中的 `QQMusicCommon.dll`，因此不能只靠独立离线算法完成。

### 7.2 Runner 的职责

- 自动寻找已运行的 QQ 音乐
- 未运行时尝试后台启动
- 创建命名管道
- 注入 Hook
- 接收解密结果

### 7.3 Hook 的职责

- 在 QQ 进程内调用解密接口
- 将结果路径和格式通过管道回传

## 8. 酷狗桥接设计

### 8.1 问题背景

在用户机器上，酷狗安装目录中的 `infra.dll` 通常是 `x86`。

如果主程序在 64 位系统上以 64 位进程运行，则：

- 无法直接加载 32 位 `infra.dll`
- 会出现“加载酷狗 infra.dll 失败”

### 8.2 解决方案

新增独立工程：

- `KugouInfraRunner`

它固定以 `x86` 运行，负责：

- 加载 `infra.dll`
- 解析 sqlite 导出函数
- 打开并解锁 `KGMusicV3.db`
- 查询指定 `audioHash` 的 `EncryptionKey`
- 把密钥回传给主程序

这样主程序仍保持 64 位运行，不需要因为酷狗链路回退为 32 位。

## 9. 临时文件与清理策略

中间解密文件统一写入：

- `%TEMP%\WpfApp1\decoded\{guid}`

`ffmpeg` ASCII 中转文件写入：

- `%TEMP%\WpfApp1\ffmpeg\{guid}`

设计原则：

- 主业务结果优先
- 清理失败不覆盖主错误

## 10. 错误处理策略

### 10.1 输入校验

- 文件不存在时立即失败
- 不支持扩展名时立即返回

### 10.2 解密结果校验

- 解密后先确认临时明文文件存在
- 若路径返回值异常，会尝试按同名基础名在输出目录中自动回找

### 10.3 依赖缺失处理

- 缺少 `ffmpeg.exe`
- 缺少 QQ Runner 文件
- 缺少客户端 DLL
- 缺少数据库

都会返回明确错误信息

## 11. 运行依赖总结

必须依赖：

- `.NET Framework 4.7.2`
- `Tools\ffmpeg.exe`

按格式依赖：

- `ncm`：无客户端依赖
- `kwm`：无客户端依赖
- `kgma`：无客户端依赖
- `kgg`：依赖酷狗安装、`infra.dll`、`KGMusicV3.db`
- `mgg/mflac`：依赖 QQ 音乐安装、Hook/Runner 链路

## 12. 当前限制

- `kgg` 依赖客户端数据库中已存在该文件对应密钥
- `mgg/mflac` 依赖 QQ 客户端内部接口可用
- 各厂商客户端更新后，内部 DLL 导出签名可能变化，需要适配

## 13. 后续可扩展方向

- 增加格式批量转换日志
- 增加更明确的 UI 进度展示
- 为 `kgg` 和 `mgg` 增加更细粒度错误码
- 引入更完整的自动化样本回归测试

## 14. 适配性评估

从工程实现角度看，当前各格式的用户电脑适配性可分为三个层级。

### 14.1 高适配性格式

- `flac`
- `ogg`
- `ncm`
- `kwm`
- `kgma`

原因：

- 主要依赖本地算法或 `ffmpeg`
- 不强依赖第三方客户端内部导出函数
- 不依赖客户端数据库中的动态密钥记录
- 受用户电脑软件版本差异影响较小

### 14.2 中等适配性格式

- `kgg`

原因：

- 解密正文的算法本身是稳定的
- 但密钥获取依赖酷狗客户端环境
- 依赖 `infra.dll`、`KGMusicV3.db`、本地密钥记录

当前已做的稳定性增强：

- 主程序保持 `AnyCPU + Prefer32Bit=false`
- 新增 `KugouInfraRunner` 以 `x86` 方式桥接加载酷狗 `infra.dll`
- 避免了 64 位主程序直接加载 32 位 DLL 的失败场景

剩余风险：

- 不同酷狗版本的数据库结构或导出行为变化
- 用户数据库中不存在目标文件密钥

### 14.3 相对较低适配性格式

- `mgg`
- `mflac`

原因：

- 依赖 QQ 音乐客户端进程和内部 DLL
- 依赖 EasyHook 注入链路稳定
- 依赖 QQ 客户端内部实现保持兼容
- 受客户端版本、系统权限、安全软件拦截影响更明显

### 14.4 结论

如果只从“满足运行前提后在陌生用户电脑上的成功概率”来看，当前兼容性排序大致为：

- 高：`flac / ogg / ncm / kwm / kgma`
- 中：`kgg`
- 中偏低：`mgg / mflac`

因此，当前方案整体适配性属于“中等偏高”，但其中真正的环境敏感点主要集中在：

- 酷狗 `kgg` 的密钥读取链路
- QQ `mgg/mflac` 的注入与客户端内部解密链路
