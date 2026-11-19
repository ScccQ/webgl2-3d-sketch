# 3D Sketch Canvas (Native WebGL2)

一个极简、高性能的原生 WebGL2 2D-to-3D 空间笔触绘制引擎。砍掉传统重型场景图（Scene Graph），专为低延迟实时绘图与 3D 轨迹导出设计。

## 技术选型

- **渲染底层**：原生 WebGL2（无 Three.js / Babylon.js 等重型引擎依赖）
- **开发语言**：TypeScript
- **数学库**：轻量级 4×4 矩阵与向量运算（View / Projection 矩阵、射线反算）
- **着色方案**：MatCap / 极简 Blinn-Phong 光照

## 核心管线规划

### 1. 深度求解（Depth Resolution）
- 采用**虚拟画板（Virtual Drawing Plane）**约束欠定深度问题。
- 屏幕坐标经相机矩阵反算为世界射线（Raycasting），与虚拟平面或已有 3D 网格求交，实时生成带深度的 3D 轨迹点。
- 支持旋转视角联动画板，以及快捷键推拉画板深度。

### 2. 几何生成（Stroke-to-Mesh）
- **路径平滑**：对原始输入点阵执行移动平均或 Catmull-Rom 样条插值，消除高频采样锯齿。
- **管体挤出**：采用**平行移动标架（Parallel Transport Frame, PTF）**沿切线方向传递法线，规避 Frenet 标架在拐点或直线段导致的管体扭曲（拧麻花）问题。
- **UV 映射**：圆环周长映射为 `U`，笔触累计弧长映射为 `V`，保证笔刷贴图在任意长度下不拉伸。

### 3. 贴图管理与动态合批（Batching & Performance）
- **笔刷贴图**：基于 WebGL2 `Texture2DArray` 统一管理所有笔刷纹理，顶点属性携带 `brushId`，Shader 按层采样，消除纹理切换开销。
- **单 Draw Call 合批**：全画布共用预分配的大块静态 VBO（使用 `TRIANGLES` 或带退化三角形的 `TRIANGLE_STRIP` 拼接）。
- **防主线程阻塞**：
  - **按帧攒批**：`pointermove` 仅写入 JS 内存队列，`requestAnimationFrame` 渲染前每帧仅执行一次 `bufferSubData` 批量上传。
  - **动静分离**：当前正在绘制的笔触独占独立小块动态 VBO，抬笔（`pointerup`）后一次性合并追加至静态大 VBO 末尾，避免 GPU 管线同步等待（Pipeline Stall）。

### 4. 资产导出（glTF 2.0 Export）
- 内置标准 glTF 2.0（`.glb` / `.gltf`）导出模块，将画布中的 3D 笔触网格导出为通用 3D 资产。
- 负责顶点/法线/UV 数据的二进制打包与 4 字节对齐（Byte Alignment）、`bufferView` 与 `accessor` 元数据生成，以及目标 DCC 软件的坐标系转换。
- 验收标准：通过 Khronos 官方 glTF Validator 校验，导入 Blender 后几何体与法线方向无误。

## 路线图

- [ ] 基础相机控制与虚拟画板射线求交
- [ ] PTF 管体生成与 MatCap 着色
- [ ] `Texture2DArray` 笔刷系统与动静分离 VBO 合批
- [ ] Undo/Redo 状态栈
- [ ] glTF 2.0 导出模块
