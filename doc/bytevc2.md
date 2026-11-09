# ByteVC2 / BVC2 解码接入

## 实现范围

本次直接修改原源码目录，不使用新的 worktree，不生成或替换预编译库：

- `D:/MyCode/MJC/FongMi/FFmpeg`：`release-8.1-fongmi`。
- `D:/MyCode/MJC/media3`：`release-1.11.1-fongmi`。
- `D:/MyCode/MJC/TVBoxOSC`：仅补 `FfmpegLibrary` 的 `video/x-bytevc2` → `bvc2` 映射。

`format_supported=YES` 只是播放器的格式支持判定，不是已经解码出视频帧。上一次在 JNI 中复用普通 VVC decoder、转换为 Annex B 的实验路径没有完成实播验收；本次改为 FFmpeg 正式注册的独立 BVC2 路径。

数据路径：

```text
MP4 bvc2 视频轨
  → Media3 video/x-bytevc2，保留四字节 NAL 长度前缀和原始参数集
  → media3ext 查找 avcodec_find_decoder_by_name("bvc2")
  → AV_CODEC_ID_BVC2 / ff_bvc2_decoder
  → 专用 BVC2 CBS：固定 NALFF + 四字节长度，不解析 VVCC、不转 Annex B
  → 复用 VVC 视频帧解码核心
```

已有 VVC/HEVC 路径保持不变。没有增加 H.264 转码或整集缓存；此处实现的是解码，不是 BVC2 编码器或所有容器的重新封装支持。

## 组件与许可

BVC2 拆包及 decoder 接入参考：

- `zfdang/hongguo-open-player@81831d8e3a66b219ea11e5dabe77ac13e01ddd0e` 中的 BVC2 补丁。
- 上游 `wxyzh/ffmpeg-bvc2@04f87d26d67162c0f73c3d9bd1870fd5900f3aee`。
- 解码补丁源自 Paul B Mahol 的 librempeg，按 GPLv3 提供；启用和分发时须遵守对应许可。

`bvc2_decoder` 依赖 `gplv3`，并选择 `cbs_bvc2` 和 `vvc_decoder`。后者会带入 VVC 的现有实现及架构优化对象，避免只复制通用 C 对象而漏掉 ARM64 NEON 等依赖。不要同时显式关闭 VVC decoder。

## FFmpeg 编译

保留原有 Android 工具链、ABI、共享库和优化参数，确保包含：

```sh
--enable-gpl --enable-version3 --enable-decoder=bvc2,vvc
```

如果原脚本使用 `--disable-everything` 或 decoder 白名单，必须将以上组件启用；不要照搬桌面转码项目的 `--disable-asm`。不需要为解码引入 libx264 或开启 AAC 编码器。

MP4 解封装需保留 MOV demuxer。BVC2 标签已映射到独立 codec ID；`glbl` 和 `bv2C` 配置 payload 原样进入 extradata，不能改名为 `vvcC`。

已知边界：FFmpeg 沿用 `mov_read_glbl` 的现有“保留首份有效 extradata”策略，而 Media3 优先 `bv2C`。若同一轨同时带两种且内容不同，两条解封装路径可能选用不同配置；目前没有真实文件证明触发此情况，不将其认定为本次错误原因。Media3 → JNI 的主播放链不经过 FFmpeg MOV demuxer。

配置后应检查生成文件：

```text
CONFIG_BVC2_DECODER=1 / yes
CONFIG_VVC_DECODER=1 / yes
CONFIG_CBS_BVC2=1 / yes
CONFIG_CBS_H266=1 / yes
CONFIG_GPLV3=1 / yes
```

如果同时构建了命令行程序，可通过 `ffmpeg -hide_banner -decoders` 确认存在 `bvc2`。仅在 `configure --list-decoders` 中看到名称，不能证明最终库已启用它；运行时应以 `avcodec_find_decoder_by_name("bvc2")` 为准。

## Media3 和 TVBox 集成

1. 编译 Media3 的 `lib-common`、`lib-container`、`lib-extractor`，更新 TVBox 实际使用的三个 AAR；只改 Java 源文件不会改变已经发布的本地 Maven 产物。
2. 将同一次、同一 ABI 编译的 FFmpeg 头文件及配套 `libavcodec`、`libavformat`、`libavutil`、`libswscale`、`libswresample` 同步到 TVBox 的相应 include / jniLibs 位置，避免混用新旧头文件或不同 ABI 的库。
3. 重新构建 `media3ext` 和 TVBox APK。原 TVBox 源码按 decoder 名直接查询，不提供 `bvc2` → `vvc` 冒充支持的回退；缺少真正 BVC2 decoder 时不应判定可用。
4. 使用 Exo/Media3 并允许 FFmpeg 软件解码。上一次实验 APK 仍包含旧的兼容层，不能作为本次源修改的验证产物；若手工移植过实验代码，不要继续对 BVC2 sample 或 extradata 做 Annex-B 转换。

本次没有移植 IJK 自带的旧 FFmpeg 内核。更新这里的共享 FFmpeg 库，不等于已经更新 IJK 的独立 `libijkffmpeg.so`。

## 当前验证边界

- Media3 原目录的 `MimeTypesTest` 12 项、`BoxParserTest` 11 项、`ByteVc2Mp4ExtractorTest` 10 项，共 33 项通过，零失败、零跳过；只运行单测，没有打包 AAR 或 APK。
- 原生 BVC2 补丁通过当前源码的上下文检查，`configure --list-decoders` 能识别 `bvc2`，脚本语法与补丁空白检查通过。跨文件只读审查未发现确定的配置/注册接线错误，不等于编译或实播验收。
- 本机的 Android 交叉配置检查停在 `Host compiler lacks C11 support`；未通过变更源码或假造编译器绕过这一错误。
- 未编译/替换 FFmpeg 库、Media3 AAR 或 APK，未安装应用、读取账号缓存或发出新的播放接口请求。
- 尚未用真实 ByteVC2 文件验证输出帧。若新构建仍失败，需要 `Caused by` 及 `avcodec_open2` / `avcodec_send_packet` 等具体错误；只提供顶层 renderer 错误和 `format_supported=YES` 无法区分参数集、解码、解密或 Surface 输出问题。日志不要包含完整媒体 URL、Cookie、Token 或密钥。
