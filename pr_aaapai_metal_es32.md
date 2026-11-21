## 背景

在 iPhone X（iOS 16.7.15，A11）上通过 Amethyst 启动器 + MobileGlues 2.0.16 运行 Minecraft 26.1 时，Metal 后端存在两处会导致崩溃/几何错乱的问题；同时光影常用的一批 ES 3.1 能力还没有接通。本 PR 修复这些问题。

## 主要修改

### 1. 纹理缓冲（TBO）

- `newTextureWithDescriptor:offset:bytesPerRow:` 即使对 `MTLTextureTypeTextureBuffer` 也会校验行跨度，传 `0` 会在 iOS 上触发断言：
  `Linear texture: bytesPerRow (0) must be greater or equal to (N) bytes`。
  Minecraft 26.1 进入世界、第一次创建较大的 texel buffer 时必崩。现改为传 `width * pixelSize`。
- `MTLTextureTypeTextureBuffer` 与 `minimumTextureBufferAlignmentForPixelFormat:` 自 iOS/tvOS 12 起就可用，`MAX_TEXTURE_BUFFER_SIZE` 不再错误地按 iOS 14 门控，避免 iOS 12/13 上上限为 0。
- 修复 buffer texture 绑定路径的编译错误（`mViewFromBaseToMaxLevel` 需要 `getNativeTexture()`）。

### 2. ES 3.1 顶点属性绑定（AVB）

`VertexArrayMtl` 之前只用 `gl::VertexBinding::getOffset()` 作为 Metal 顶点缓冲偏移，从未应用 `gl::VertexAttribute::relativeOffset`。当多个属性共享同一个 `glBindVertexBuffer` 绑定时（Minecraft 26.1 的 `VertexArrayCache.Separate` 就是这种用法），所有属性都从 binding 基址取数据，顶点全部错位。

本次修改：

- `syncDirtyAttrib` 统一计算 `attribOffset = binding.offset + attrib.relativeOffset`，用于对齐检查、直接绑定偏移和转换路径。
- 格式转换路径（转换缓存 key、源数据起始偏移、可读顶点数、输出流偏移）全部改用 `attribOffset`。
- 处理 `glBindVertexBuffer` 允许的 `stride == 0`（所有顶点读同一份数据）：只转换一个元素，输出流 stride 保持 0，避免除零。

### 3. 着色器 image / 扩展

- compute dispatch 也绑定 shader image（之前只绑定 sampler，image 单元在 compute 里始终未绑定）。
- 对 3D / MSAA / 深度等无法创建 2D 视图的 image，返回 `GL_INVALID_OPERATION`，而不是在资源层断言崩溃。
- 开启 `GL_EXT_gpu_shader5`、`GL_EXT_copy_image` / `GL_OES_copy_image`（后端路径均已可用）。
- 修正 `MAX_COMBINED_IMAGE_UNIFORMS` 与 `MAX_COMBINED_SHADER_OUTPUT_RESOURCES`。
- 恢复上报 ES 3.2：AVB 修复后 ES 3.1 顶点路径可用，此前的 3.0 上报只是规避该缺陷。

### 4. SSBO

仍未对外声明（`maxShaderStorageBlockSize = 0`）。MSL 翻译器目前无法把 buffer block lowering 成 `device T *`，广告出去只会让 shader 在翻译阶段失败；翻译器对 SSBO 给出明确错误，而不是生成非法 Metal。

## 验证

iPhone X / iOS 16.7.15，Amethyst 6.0.0 + MobileGlues 2.0.16 + Minecraft 26.1：

- 修复前：进世界几何错乱，随后在创建 texel buffer 时 Metal 断言闪退。
- 修复后：区块正常生成，可以正常进入世界。

## 说明

本 PR 基于 `aaaapai/angle` 的 `develop`（单 commit，无合并提交、无二进制文件）。如果维护者希望调整版本上报策略（3.2 vs 3.0）或拆分 SSBO 相关改动，我可以按需调整。
