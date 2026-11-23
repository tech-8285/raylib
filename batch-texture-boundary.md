# rlgl 桌面批渲染：提交边界、顶点/纹理归属与自定义采样器有效期

分析对象：`src/rlgl.h`（OpenGL 3.3 Core 路径，`GRAPHICS_API_OPENGL_33`；ES2 路径代码结构一致，差异在索引类型与默认容量）。
调用方：`src/rcore.c`、`src/rshapes.c`、`src/rtext.c`、`src/rtextures.c`。

本文所有行为都可以直接对着源码复算；第 1 节给出了一段可复算序列和一组"缩容"参数（`RL_DEFAULT_BATCH_BUFFER_ELEMENTS = 4`），把它们代回去就能手算出同样的 draw call 边界。

---

## 0. 先看三条硬事实（后面所有结论都由它们推出）

### 0.1 缓冲区按"顶点槽"分配，不按图元分配

`rlLoadRenderBatch()` 为每个 buffer 分配 `elementCount * 4` 个顶点槽（一个 element = 一个 QUAD = 4 个顶点）：

- `src/rlgl.h:2787-2799`：`vertices/texcoords/normals/colors/indices` 按 `bufferElements*4` 分配（`elementCount = bufferElements`，`2789`）。
- `src/rlgl.h:203-214`：默认 `RL_DEFAULT_BATCH_BUFFER_ELEMENTS = 8192`（GL3.3/GL1.1），ES2 为 `2048`。
- 默认容量因此是 `8192*4 = 32768` 个顶点槽。

`rlCheckRenderBatchLimit(vCount)` 的溢出判断是 `vertexCounter + vCount >= elementCount*4`（`src/rlgl.h:3210-3211`）。注意是 `>=` 而不是 `>`：容量被用满时**提前一格**就认为溢出。

### 0.2 一个批次 = 一条共享顶点缓冲 + 一组 `rlDrawCall`

```c
typedef struct rlDrawCall {          // src/rlgl.h:407-418
    int mode;                        // LINES / TRIANGLES / QUADS
    int vertexCount;
    int vertexAlignment;             // 补齐到 4 的倍数，供下一段 QUADS 索引对齐
    unsigned int textureId;          // 只有它变化才切 draw call
} rlDrawCall;

typedef struct rlRenderBatch {        // src/rlgl.h:420-429
    rlVertexBuffer *vertexBuffer;
    rlDrawCall *draws;  int drawCounter;
    float currentDepth;
} rlRenderBatch;
```

关键点：`rlDrawCall` 里**没有** shaderId / vaoId / 矩阵字段（`src/rlgl.h:413-418` 是注释掉的）。所以"批次内共享的状态"只有：当前 shader 程序、当前 VAO/VBO、当前投影与 modelview 矩阵、当前混合/深度/裁剪状态。这些一旦变化，只能靠调用方（core 模块）整批提交，见 `src/rlgl.h:405-406` 的注释：

> `NOTE: Only texture changes register a new draw, other state-change-related elements are not used at this moment (vaoId, shaderId, matrices), raylib forces a batch draw call if any of those state-change happens (this is done in core module)`

### 0.3 批次内每段几何的顶点区间是按 `vertexCount + vertexAlignment` 累加出来的

`rlDrawRenderBatch()` 里唯一的偏移计算（`src/rlgl.h:3111-3131`）：

```c
for (int i = 0, vertexOffset = 0; i < batch->drawCounter; i++)
{
    glBindTexture(GL_TEXTURE_2D, batch->draws[i].textureId);
    if (mode == LINES || mode == TRIANGLES)
        glDrawArrays(mode, vertexOffset, draws[i].vertexCount);
    else
        glDrawElements(GL_TRIANGLES, draws[i].vertexCount/4*6,
                       GL_UNSIGNED_INT, (void *)(vertexOffset/4*6*sizeof(GLuint)));
    vertexOffset += (draws[i].vertexCount + draws[i].vertexAlignment);   // ← 3111-3131
}
```

`vertexOffset` 是**绝对顶点下标**（不是字节），QUADS 再用 `/4*6` 换算成索引偏移。因此：

- 段与段之间只要有 `vertexAlignment > 0`，就产生一段**写了不画**的空槽（`rlVertex3f` 从不写这些槽，见 1.3）。
- 这些空槽照样占用容量，也照样把后面所有段的下标推后。
- QUADS 段的偏移必须是 4 的倍数（索引表 `4k..4k+3` 是全局静态写好的，`src/rlgl.h:2809-2820`），这就是对齐存在的原因。

---

## 1. 可复算序列：自动提交前后到底画了什么

### 1.1 场景与参数

为了能手算，把容量缩小到 `RL_DEFAULT_BATCH_BUFFER_ELEMENTS = 4`，即 **16 个顶点槽**（真实默认是 32768，机制完全相同，只是数字放大 2048 倍）。

三个纹理标识：

| 记号 | 含义 | 来源 |
|---|---|---|
| `defaultTex` | rlgl 内部的 1×1 白色纹理 | `RLGL.State.defaultTextureId`，`src/rlgl.h:2280` |
| `shapeTex` | 形状/字体共享图集 | `InitWindow()` 里 `SetShapesTexture(GetFontDefault().texture, ...)`，`src/rcore.c:692-706` |
| `noiseTex` | 自定义 shader 的附加采样器 | 本序列通过 `SetShaderValueTexture()` 注册 |

一帧内依次调用（全部是公开 API，都在同一帧、同一 `BeginDrawing()/EndDrawing()` 之间）：

| 组 | 调用 | rlgl 侧动作 | 顶点数 |
|---|---|---|---|
| G0 | `SetShaderValueTexture(shader, noiseLoc, noiseTex)` | `rlSetUniformSampler` → `activeTextureId[0]=noiseTex`，`glUniform1i(noiseLoc, 1)` | 0 |
| G1 | `DrawLineV((10,10),(50,10),BLACK)` | `rlBegin(RL_LINES)` + 2 顶点 | 2 |
| G2 | `DrawRectangle(10,20,40,20,RED)` | `rlSetTexture(shapeTex)` → `rlBegin(RL_QUADS)` → 4 顶点 → `rlSetTexture(0)` | 4 |
| G3 | 1 个字形（`DrawTextCodepoint` → `DrawTexturePro(font.texture,…)`） | `rlSetTexture(shapeTex)` 相同 → `rlBegin(RL_QUADS)` → 4 顶点 → `rlSetTexture(0)` | 4 |
| G4 | `DrawLineV((10,50),(50,50),BLACK)` | `rlBegin(RL_LINES)` + 2 顶点 | 2 |
| G5 | `DrawRectangle(10,60,40,20,RED)` | 同 G2 | 4 |
| G6 | `DrawLineV((10,90),(50,90),BLACK)` | `rlBegin(RL_LINES)` + 2 顶点 | 2 |
| G7 | `EndDrawing()` | `rlDrawRenderBatchActive()` 收尾提交 | — |

逐条对应的源码位置：

- `DrawLineV`：`src/rshapes.c:200-208`（`rlBegin(RL_LINES)` … `rlEnd()`）。
- `DrawRectangle` → `DrawRectangleV` → `DrawRectanglePro`：`src/rshapes.c:601-682`，QUADS 分支在 `659-682`（`GetShapesTexture().id` 取 `shapeTex`）。
- 字形：`src/rtext.c:1282-1303` 的 `DrawTextCodepoint` 调 `DrawTexturePro(font.texture, …)`，后者在 `src/rtextures.c:4876`，纹理与绘制代码在 `4929-4956`。
- `rlBegin`：`src/rlgl.h:1452-1482`；`rlSetTexture`：`1638-1691`；`rlVertex3f`：`1495-1555`；`rlEnd`：`1485-1491`。

### 1.2 逐次操作后的状态（可逐步复算）

下表每一行是"这次 API 结束后 `vertexCounter` / `drawCounter` / 当前 draw 的状态"。**加粗**行是发生了自动提交的行。

| # | 操作 | 进入时 counter | 关键判断 | 退出时 counter | drawCounter | 当前 draw#k |
|---|---|---|---|---|---|---|
| G1 | `rlBegin(LINES)` | 0 | `draws[0].mode` 初始化即 `RL_QUADS`（`src/rlgl.h:2891`）≠ LINES 且 `vertexCount==0`，**不加对齐、不切 draw**（`1457` 的 `if (vertexCount > 0)` 拦住了） | 0 | 1 | #0: LINES, vc=0, tex=currentTextureId=`defaultTex`（`1478-1480`），随后 `currentTextureId` 复位为 `defaultTex` |
| G1 | 2×`rlVertex3f` | 0 | 容量守卫 `1512-1532` 不触发（`0 > 16-4` 为假）；写入 `1534-1554` | 2 | 1 | #0: LINES, vc=2 |
| G2 | `rlSetTexture(shapeTex)` | 2 | `draws[0].textureId(defaultTex) != shapeTex` → 需要切 draw；`vertexCount=2>0` → 算对齐：LINES 且 vc<4 ⇒ `vertexAlignment = vertexCount = 2`（`1669`）；`rlCheckRenderBatchLimit(2)`：`2+2 < 16` 不溢出 | 4 | 2 | 新 #1：`mode` 继承 `draws[0].mode = LINES`（`1679`），`textureId=shapeTex`，vc=0 |
| G2 | `rlBegin(QUADS)` | 4 | `draws[1].mode(LINES) != QUADS`，但 `vertexCount==0` → 不切 draw；此时 `currentTextureId` 仍是 `shapeTex`（`1659` 赋值，后续无操作覆盖它） | 4 | 2 | #1: mode→QUADS（`1478`），`textureId = currentTextureId = shapeTex`（`1479`）；随后 `currentTextureId ← defaultTex`（`1480`） |
| G2 | 4×`rlVertex3f` | 4 | — | 8 | 2 | #1: QUADS, vc=4 |
| G2 | `rlSetTexture(0)` | 8 | `id==0` 分支（`1640-1652`）：`8 < 16` 不提交；`currentTextureId ← defaultTex` | 8 | 2 | #1 不变 |
| G3 | `rlSetTexture(shapeTex)` | 8 | 当前 `draws[1].textureId == shapeTex`，**相同纹理不切 draw**（`1660`），只改 `currentTextureId` | 8 | 2 | #1: QUADS, vc=4, tex=shapeTex |
| G3 | `rlBegin(QUADS)` | 8 | mode 相同，无事 | 8 | 2 | #1 |
| G3 | 4×`rlVertex3f` | 8 | — | 12 | 2 | #1: QUADS, vc=8 |
| G3 | `rlSetTexture(0)` | 12 | `12 < 16` | 12 | 2 | #1 |
| G4 | `rlBegin(LINES)` | 12 | `draws[1].mode(QUADS) != LINES` 且 `vc=8>0` → 对齐：QUADS ⇒ 0（`1671`）；`check(0)`：`12+0 < 16` 不溢出 → `drawCounter=3` | 12 | 3 | 新 #2：随后 `1478-1480` 置 `mode=LINES`、`textureId=currentTextureId=defaultTex`（G3 的 `rlSetTexture(0)` 已在 `1651` 把它复位为默认纹理） |
| G4 | 2×`rlVertex3f` | 12 | 容量守卫在 `vertexCounter > 16-4` 时才检查（`1512`）：第 1 个顶点前 `12 > 12` 为假不检查，写入后 counter=13；第 2 个顶点前 `13 > 12` 为真，LINES 且 `vc=1` 是奇数（图元未闭合）⇒ 守卫**跳过**（`1514-1521`），写入后 counter=14 | 14 | 3 | #2: LINES, vc=2, tex=defaultTex |
| G5 | `rlSetTexture(shapeTex)` | 14 | `draws[2].textureId(defaultTex) != shapeTex`，`vc=2>0` → 对齐：LINES 且 vc<4 ⇒ `vertexAlignment=2`（`1669`）；`rlCheckRenderBatchLimit(2)`：`14+2 >= 16` **溢出 ⇒ 自动提交** | **0** | **1** | 提交后 `rlCheckRenderBatchLimit` 把 `mode=LINES, textureId=defaultTex` 写回新的 `draws[0]`（`3221-3223`）；然后 `rlSetTexture` 继续把 `draws[0].textureId = shapeTex`、`vc=0`（`1686-1687`） |
| G5 | `rlBegin(QUADS)` | 0 | `draws[0].mode(LINES) != QUADS` 且 vc==0 → 不切 draw | 0 | 1 | #0: mode→QUADS（`1478`），tex=`currentTextureId`=`shapeTex`（`1479`：`rlSetTexture` 在 `1659` 已设 `currentTextureId=shapeTex`），随后复位 `defaultTex` |
| G5 | 4×`rlVertex3f` | 0 | — | 4 | 1 | #0: QUADS, vc=4 |
| G5 | `rlSetTexture(0)` | 4 | `4 < 16` | 4 | 1 | #0 |
| G6 | `rlBegin(LINES)` | 4 | mode 不同、vc=4>0 → 对齐 QUADS ⇒ 0；`check(0)`：`4 < 16` 不溢出 → `drawCounter=2`；随后 `1478-1480` 置 `mode=LINES`、`textureId=currentTextureId=defaultTex`（G5 的 `rlSetTexture(0)` 已在 `1651` 复位） | 4 | 2 | 新 #1: LINES, vc=0, tex=defaultTex |
| G6 | 2×`rlVertex3f` | 4 | — | 6 | 2 | #1: LINES, vc=2 |
| G7 | `rlDrawRenderBatchActive()` | 6 | `vertexCounter=6 > 0` → 上传并绘制；随后复位（`3154/3157/3172/3175`） | 0 | 1 | — |

### 1.3 两次提交后屏幕上各组图元的真实归属

以下数字由上面的状态表直接累加得到，与模拟器输出一致。

**批次 #1 —— 触发点 G5 的 `rlSetTexture`，`rlCheckRenderBatchLimit(2)`，提交时 `vertexCounter = 14`**

| draw# | 归属图元 | mode | vertexCount | vertexAlignment | 占用的顶点槽 | texture0 | 实际发出的 GL 调用 |
|---|---|---|---|---|---|---|---|
| 0 | G1 `DrawLineV` | LINES | 2 | **2** | [0,4) | `defaultTex` | `glDrawArrays(GL_LINES, 0, 2)` |
| 1 | G2 `DrawRectangle` + G3 字形 | QUADS | 8 | 0 | [4,12) | `shapeTex` | `glDrawElements(GL_TRIANGLES, 12, idx + 24B)` |
| 2 | G4 `DrawLineV` | LINES | 2 | **2** | [12,16) | `defaultTex` | `glDrawArrays(GL_LINES, 12, 2)` |

要点：

- G2 与 G3 **合并成了同一个 draw call**（draw#1），因为 `rlSetTexture(shapeTex)` 在 G3 时看到 `draws[1].textureId` 已经是 `shapeTex`，`src/rlgl.h:1660` 的 `!=` 判断不成立。这正是 `SetShapesTexture()` 想达到的效果（`src/rshapes.c:5-12` 注释、`src/rcore.c:689-706`）。
- 槽 [2,4) 和 [14,16) 是**对齐空洞**：`rlVertex3f` 只写 `3*vertexCounter` 起的槽（`1535-1553`），`vertexCounter += vertexAlignment`（`1675`）只是把游标推过去，空洞里的数据永远停在 `RL_CALLOC` 的 0（`src/rlgl.h:2802-2805`）。这些槽不会被任何 `glDraw*` 触及，纯粹是"占位"。
- 提交时 `vertexCounter=14`，但实际槽跨度是 16 —— 因为 draw#2 的 `vertexAlignment=2` 在提交**之前**已经被算进 `vertexCounter`（`1675`）。这 2 个空槽属于批次 #1 的地址空间，不属于批次 #2。

**批次 #2 —— 触发点 G7 `EndDrawing()`**

| draw# | 归属图元 | mode | vertexCount | vertexAlignment | 槽 | texture0 | GL 调用 |
|---|---|---|---|---|---|---|---|
| 0 | G5 `DrawRectangle` | QUADS | 4 | 0 | [0,4) | `shapeTex` | `glDrawElements(GL_TRIANGLES, 6, idx + 0B)` |
| 1 | G6 `DrawLineV` | LINES | 2 | 0 | [4,6) | `defaultTex` | `glDrawArrays(GL_LINES, 4, 2)` |

批次 #2 只用了 6/16 槽，`[6,16)` 的 10 个槽被丢弃——这就是"留下空间"的代价：跨批次容量不回收。

### 1.4 "空间"与"几何归属"的判据

把上面两张表抽象成规则：

1. **几何属于哪个批次，由写入时刻的 `RLGL.currentBatch`（默认就是 `RLGL.defaultBatch`）和当时的 `vertexCounter` 决定**，与调用顺序上的"意图"无关。G5 的 4 个顶点是在批次 #1 提交**之后**才写进去的，所以它们落在批次 #2 的槽 [0,4)；G4 的 2 个顶点在提交前写入，落在批次 #1 的槽 [12,14)。
2. **对齐空洞是"前一段的尾巴"，不是"后一段的开头"**。`vertexAlignment` 写在 `draws[i]`（被打断的那一段）上（`1669-1671`、`1471-1473`），`vertexOffset` 在下一段开始前把它加进去（`3130`）。所以 G1 的 LINES（2 顶点）实际占用 4 个槽，G2 的 QUADS 从槽 4 开始而不是从槽 2。
3. **`rlBegin()` 的 mode 切换通常不产生新 draw call，只改当前 draw 的 `mode` 字段**（`1478`），前提是 `vertexCount == 0`。G2 的 `rlBegin(QUADS)` 发生在刚被 `rlSetTexture` 开出来的空 draw 上，所以它是在"改"draw#1 而不是"开"draw#2。这也是 1.3 里 G2 和 G3 能共用 draw#1 的前提。
4. **`rlSetTexture(0)` 的两条路径**：
   - `id==0`：只在 `vertexCounter >= elementCount*4` 时提交（`1646-1650`），这是个"满缓冲"的安全阀，不做对齐。
   - `id>0` 且与当前 draw 纹理不同：先对齐再切 draw，再 `check(alignment)`（`1660-1687`）。这里的 `check` 是 G5 触发提交的地方。
5. **只在正确的位置提交**：`rlVertex3f` 的守卫（`1512-1532`）只在 `vertexCount % 2/3/4 == 0`（即图元闭合）时才调 `rlCheckRenderBatchLimit`，避免把一个 LINES/TRIANGLES/QUADS 图元劈成两半。代价是：守卫一旦进入 `vertexCounter > elementCount*4 - 4` 区间而图元尚未闭合，就会一直跳过，最多"冲过"容量边界 `图元大小-1` 个顶点才提交（LINES ≤1、TRIANGLES ≤2、QUADS ≤3 个顶点）。真正的越界写不会发生，因为 `rlCheckRenderBatchLimit(n)` 用的是 `vertexCounter + n`，预留了整段对齐/整段图元的空间（`3211`）。这也是为什么 `rlVertex3f` 不直接在 `vertexCounter >= capacity` 时提交。
6. **容量不跨批次复用**：`rlDrawRenderBatch()` 把 `vertexCounter` 清零（`3154`），批次 #2 从槽 0 重新开始，批次 #1 的 `[14,16)` 空洞彻底作废。
7. **`vertexAlignment` 在批次复位时不会被清零**：`src/rlgl.h:3164-3169` 的复位循环只写 `mode`、`vertexCount`、`textureId`，`vertexAlignment` 保持残值（只有 `rlLoadRenderBatch` 的初始化才置 0，`2893`）。这一残留**不会**让任何 draw 的发出区间发生偏移：一个 draw 的 `vertexAlignment` 只在它被"打断"（`rlBegin`/`rlSetTexture` 的 mode/纹理切换）时重新计算（`1669-1671`、`1471-1473`），而"中间某个 draw 带着非 0 对齐"这一状态本身就要求它后面还有 draw，后面那个 draw 的存在又必然来自一次打断——打断会重算前一个 draw 的对齐。于是残值只可能停在两类地方：`vertexCount==0` 的空 draw，或本批次的最后一个 draw。前者不发 `glDraw*`，后者的 `alignment` 只影响"槽跨度"而不影响它自己的 `first/count`。实测方式：抓帧时若看到 `vertexCount=0` 却带非 0 `vertexAlignment` 的条目，那只是复位不彻底的痕迹，**判断是否真正绘制一律以 `vertexCount` 为准**。

---

## 2. 自定义 shader：主贴图（texture0）与附加采样器的有效期

### 2.1 两条完全不同的生命周期通路

| | 主贴图 `texture0` | 附加采样器 `texture1..4` |
|---|---|---|
| 登记方式 | `rlSetTexture(id)`（`1638`）或 `rlBegin` 时取 `currentTextureId`（`1479`） | `rlSetUniformSampler(loc, id)`（`4564`），或 `SetShaderValueTexture()`（`src/rcore.c:1392-1400`） |
| 存放位置 | `batch->draws[k].textureId`，**每段几何一份** | `RLGL.State.activeTextureId[0..3]`，**每批次一份** |
| 是否切割 draw call | **是**（`1660` 纹理不同就切） | **否**，只是把纹理 id 记进槽 |
| 提交时绑定 | `glBindTexture(GL_TEXTURE_2D, draws[i].textureId)` 逐段绑定（`3114`） | 提交开始时一次性遍历 `activeTextureId[]` 绑到 `GL_TEXTURE0+1+i`（`3098-3105`） |
| 绑定次数 | 每段 1 次 | 每批次 1 次 |
| uniform 值 | `glUniform1i(MAP_DIFFUSE, 0)` 硬编码为 0（`3094`） | `glUniform1i(loc, 1+i)` 在 `rlSetUniformSampler` 里写（`4572/4583`） |
| 复位时机 | `rlDrawRenderBatch` 把每段 `textureId` 重置为 `defaultTextureId`（`3168`） | `rlDrawRenderBatch` 把 `activeTextureId[]` 全清 0（`3172`） |

源码依据：

- 主贴图按段记录：`rlDrawCall.textureId`（`src/rlgl.h:415`）。
- 附加采样器槽位：`unsigned int activeTextureId[RL_DEFAULT_BATCH_MAX_TEXTURE_UNITS]`（`src/rlgl.h:1055`），`RL_DEFAULT_BATCH_MAX_TEXTURE_UNITS = 4`（`223-225`）。
- 提交时的绑定顺序：先 `activeTextureId[]` 绑到 unit 1..4（`3098-3105`），再 `glActiveTexture(GL_TEXTURE0)`（`3109`），再逐段 `glBindTexture(draws[i].textureId)`（`3114`）。
- 复位：`3168` / `3172`。

### 2.2 回到 1.1 的序列：两者各自的有效期

序列里 G0 注册了 `noiseTex` 到 `activeTextureId[0]`，之后 G1..G4 被提交为批次 #1，G5..G6 被提交为批次 #2。对照 1.3 的两张表：

**主贴图（`texture0`）的有效期 = 单段几何**

- 批次 #1 里，draw#0 绑 `defaultTex`，draw#1 绑 `shapeTex`，draw#2 又绑 `defaultTex`。三者是三个独立 `glBindTexture` 调用（`3114`），互不影响。
- G2 和 G3 之所以能共享 draw#1，是因为两者**碰巧**用了同一张 `shapeTex`；如果 G3 换成一个非图集字体，`rlSetTexture(newFontTex)` 会立即切出新 draw（并在 G2 的 QUADS 上留下 `vertexAlignment=0` 的尾巴）。
- 特别注意 `rlBegin` 对主贴图的"继承"语义：`rlSetTexture(shapeTex)` 后再 `rlBegin(RL_QUADS)`，`draws[k].textureId` 取自 `currentTextureId`（`1479`）。如果调用方写成 `rlBegin()` 再 `rlSetTexture()`，顺序不同会得到不同的 draw 切分，但**最终渲染结果相同**，因为 `rlSetTexture` 无论如何都会把 `textureId` 写到当前 draw。真正影响结果的只有"同一纹理是否跨图元/跨 `rlBegin`"。

**附加采样器（`texture1..4`）的有效期 = 批次**

- 批次 #1 提交时，`activeTextureId[0] = noiseTex`，所以 `3098-3105` 把 `noiseTex` 绑到 `GL_TEXTURE1`，**draw#0/#1/#2 三段全部看到 `noiseTex`**，尽管 G1 的 `DrawLineV` 与自定义 shader 毫无关系。
- 提交过程末尾 `3172` 把 `activeTextureId[]` 清零。到批次 #2（G5、G6）提交时，`activeTextureId[]` 已经全 0，`3098-3105` 的循环**什么都不绑**。于是 `GL_TEXTURE1` 上残留的是批次 #1 绑过的 `noiseTex` —— 这不是"未定义"，而是"上一批次的遗留"。
- 后果：如果同一帧里 `G0` 的 `SetShaderValueTexture` 只调用一次，那么**只有包含 G0 之后、第一次提交之前那段几何的批次**会拿到 `noiseTex`；后面的批次拿到的是同一条残留绑定（碰巧还是 `noiseTex`，所以看起来"没问题"），直到别的代码把 `GL_TEXTURE1` 绑成别的东西。

### 2.3 哪些结果由库内状态决定

下列行为完全由 rlgl 的内部状态机决定，可离线推算，与 GPU 无关：

1. **draw call 如何切分**：只取决于 `draws[k].mode`、`draws[k].textureId`、`vertexCount`、对齐规则（`1457-1481`、`1660-1687`）。
2. **每个顶点的槽位与 `glDrawArrays`/`glDrawElements` 的 offset/count**：由 `vertexOffset += vertexCount + vertexAlignment`（`3130`）累加。
3. **批次如何、何时提交**：`rlSetTexture(0)` 满缓冲分支（`1646`）、`rlSetTexture(id)` 的 `check(alignment)`（`1673`）、`rlVertex3f` 守卫（`1512-1532`）、`drawCounter >= 256`（`1476`/`1684`）、以及 core 模块的前置提交（`rlSetShader` `4597`、`rlSetBlendMode` `2123`、`BeginMode2D` `rcore.c:932`、`BeginMode3D` `953`、`EndMode3D` `992`、`BeginTextureMode` `1008`、`EndTextureMode` `1039`、`BeginScissorMode` `1086`、`EndScissorMode` `1113`、`EndDrawing` `887`）。
4. **附加采样器落在哪个 unit**：`rlSetUniformSampler` 按 `activeTextureId[]` 的"首个空槽"分配（`4579-4586`），并写 `glUniform1i(loc, 1+i)`。同一批次内多个 `loc` 指向同一 `textureId` 会复用同一 unit（`4567-4575`）。
5. **每批次结束时 `activeTextureId[]` 被清零**（`3172`），这是"附加采样器只在当前批次有效"的根源。

### 2.4 哪些结果取决于驱动或着色器输入

以下不在 rlgl 控制范围内：

1. **`glUniform1i` 是否真的影响采样**：取决于 shader 里该 `loc` 是否真的对应一个 `sampler2D`/`samplerCube` 且被实际引用。`rlSetUniformSampler` 不做 `locIndex == -1` 检查，也不校验类型（`4564-4589`）；`SetShaderValueTexture` 只检查 `locIndex > -1`（`src/rcore.c:1394`）。如果 uniform 被优化掉，`glUniform1i` 静默无效。
2. **`glGetUniformLocation` 得到的位置**：`LoadShaderFromMemory` 对自定义 shader 只按约定名抓 `texture0/texture1/texture2` 三个位置（`src/rcore.c:1298-1301`），自定义名字的 sampler 必须调用方自己 `GetShaderLocation`。
3. **采样器编号超过上限时**：`rlSetUniformSampler` 的第二个循环找不到空槽就**直接返回、不写 uniform**（`4579-4587` 没有 else 分支）。这不会报错，也不会有 trace log，行为是静默的"uniform 保持旧值"。
4. **实际绑定到 unit 上的纹理对象**：rlgl 只在"批次有 `vertexCounter > 0` 且 `activeTextureId[i] > 0`"时才绑（`3098-3105`），且批次结束时仅 `glBindTexture(GL_TEXTURE_2D, 0)` 解绑 **unit 0**（`3139`）。unit 1..4 的绑定会一直残留，后续依赖它们的绘制会读到这张残留纹理——除非别的代码覆盖。这属于 GL 全局状态，跨批次甚至跨帧都可能继承。
5. **纹理内容本身何时可见**：`rlUpdateTexture` 走 `glTexSubImage2D`，CPU 侧立刻提交，但 GPU 看到的是"提交那一刻的纹理对象内容"。同一帧内先 `UpdateTexture` 再绘制、但绘制被推迟到批次提交时，读到的仍是提交时的内容——因为纹理对象是同一个，`glTexSubImage2D` 已经写进去了。若改用 PBO 或跨上下文共享，时序就由驱动决定。
6. **多缓冲区（`RL_DEFAULT_BATCH_BUFFERS > 1`）**：`rlDrawRenderBatch` 末尾 `currentBuffer++`（`3178-3180`），每个 buffer 有自己的 VAO/VBO。`activeTextureId[]` 是全局的 `RLGL.State` 字段而不是 per-buffer，所以多缓冲下附加采样器的"有效期"比单缓冲更短——每次提交都会清空。

### 2.5 调用方维持图像一致的条件

要让"自定义 shader + 主贴图 + 附加采样器"在一帧内稳定正确，调用方需要满足：

**A. 附加采样器要在"它要生效的那个批次"之前、且之后不再触发提交之前登记。**

最稳的写法是每画一段就重新登记一次，示例（对应本仓库 `src/rcore.c:1392` 的 API）：

```c
BeginShaderMode(shader);                       // rlSetShader → 会先把旧批次提交掉（4597）
for (int i = 0; i < n; i++)
{
    SetShaderValueTexture(shader, noiseLoc, noiseTex);   // 每个批次前重新登记，避免 3172 清零
    SetShaderValueTexture(shader, maskLoc,  maskTex);
    DrawRectangle(...);                        // 这里可能触发容量提交，也可能不触发
}
EndShaderMode();
```

关键：`rlSetUniformSampler` 只写 `activeTextureId[]`，**不会**冲刷批次。所以登记动作必须发生在"批次开始填充"之前；如果登记发生在批次已经积累几何之后，那已经积累的那段会在下一次提交时看到新登记的纹理（因为它们共用同一个批次提交点），而更早提交掉的批次则看不到——这正是 2.2 的描述。

**B. 不要在批次中途改变不可入批的状态。**

`rlDrawCall` 不记录 shader / VAO / 矩阵（`src/rlgl.h:413-418`）。因此：

- 切 shader 用 `BeginShaderMode/EndShaderMode`（内部 `rlSetShader` 会整批提交，`4597`），不要在 `rlBegin/rlEnd` 之间直接 `rlEnableShader`。
- 切混合模式用 `BeginBlendMode/EndBlendMode`（`rlSetBlendMode` 提交，`2123`）。
- 切裁剪用 `BeginScissorMode/EndScissorMode`（提交语句在 `src/rcore.c:1086/1113`）。
- 切矩阵/相机用 `BeginMode2D/BeginMode3D/EndMode3D`（提交语句在 `src/rcore.c:932/953/992`）。
- 切渲染目标用 `BeginTextureMode/EndTextureMode`（提交语句在 `src/rcore.c:1008/1039`）。

这些都在 core 模块里做了前置提交，**直接用 rlgl 的话调用方得自己 `rlDrawRenderBatchActive()`**。

**C. 主贴图与附加采样器指向同一张纹理时，要意识到它们在两个 unit 上的绑定时机不同。**

同一张纹理会被绑到 unit 0（逐段，`3114`）和 unit 1..4（逐批次，`3098-3105`）各一次。功能上正确，但两者的可见期不同：unit 0 的绑定在每两个 `rlSetTexture` 之间可能被多次改写，unit 1..4 的绑定在每个批次开始时只写一次。`rlSetTexture(X)` 是否切出新 draw call **只**取决于当前 draw 的 `textureId` 是否等于 `X`（`1660`），与它是否同时被注册为附加采样器无关。如果调用方在同一帧内 `UnloadTexture(X)` 或替换了该纹理，两个 unit 看到的是同一个 GL 纹理对象，因此内容一致；但 unit 1..4 的**绑定**会残留到后续批次（见 E），而 unit 0 每次都会重新绑。

**D. 附加采样器的数量不超过 `RL_DEFAULT_BATCH_MAX_TEXTURE_UNITS`（默认 4）。**

超过时 `rlSetUniformSampler` 静默不写 uniform（`4579-4587`），且没有日志。这是最容易"看起来正常但偶发黑屏"的一类问题。可用 `RL_DEFAULT_BATCH_MAX_TEXTURE_UNITS` 宏重定义（`src/rlgl.h:222-225`、`src/config.h` 注释形式 `166-168`）。

**E. 避免依赖 unit 1..4 的"残留绑定"。**

批次结束时只有 unit 0 被 `glBindTexture(...,0)` 解绑（`3139`）。如果自定义 shader 的某个 sampler 在某个批次里没有对应的 `activeTextureId[]` 条目，它会读到上一批次留下的纹理。稳妥做法是这个 shader 的每个附加 sampler 在每个它出现的批次里都显式登记。

**F. 3D 与 2D 混画时注意 `EndMode3D` 会关掉深度测试。**

`EndMode3D` 末尾 `rlDisableDepthTest()`（`src/rcore.c:1002`）。批次提交发生在 `rlDrawRenderBatchActive()` 里，而深度测试状态是批次间共享的 GL 状态——所以在 `EndMode3D` 之前提交的 3D 几何用深度测试，之后提交的 2D 几何不用。这与顶点/纹理归属无关，但同样影响"屏幕上实际看到什么"。

**G. `SetShapesTexture` 的收益依赖于"形状与文本同图集"。**

`src/rcore.c:689-706` 默认把形状纹理设为默认字体的一个白色矩形，所以 `DrawText` + `DrawRectangle` 能合批（1.3 的 draw#1）。调用方若替换字体或形状纹理，两者纹理不同就会各自成段。这是"图像一致"没有被破坏、但 draw call 数量变化的典型例子——对结果无影响，对性能有影响。

---

## 3. 一页速查：判据、陷阱与验证方法

### 3.1 判定几何归属的算法（可直接实现）

```
# roundUp4: LINES -> vc<4 ? vc : vc%4 ; TRIANGLES -> vc<4 ? 1 : 4-(vc%4) ; QUADS -> 0
# checkLimit(n): if vertexCounter + n >= elementCount*4 -> flush batch, restore mode+textureId, return true

on rlBegin(mode):
    d = draws[drawCounter-1]
    if d.mode != mode:
        if d.vertexCount > 0:
            d.vertexAlignment = roundUp4(d.mode, d.vertexCount)
            if not checkLimit(d.vertexAlignment):        # 已提交则 counter/drawCounter 已被复位
                vertexCounter += d.vertexAlignment
                drawCounter  += 1
                draws[drawCounter-1].mode = d.mode       # 只继承 mode（1679）
        draws[drawCounter-1].mode      = mode
        draws[drawCounter-1].textureId = currentTextureId
        currentTextureId = defaultTextureId

on rlSetTexture(id > 0):
    currentTextureId = id
    d = draws[drawCounter-1]
    if d.textureId != id:
        if d.vertexCount > 0:
            d.vertexAlignment = roundUp4(d.mode, d.vertexCount)
            if not checkLimit(d.vertexAlignment):
                vertexCounter += d.vertexAlignment
                drawCounter  += 1
                draws[drawCounter-1].mode = d.mode       # 继承 mode（1679）
        draws[drawCounter-1].textureId   = id
        draws[drawCounter-1].vertexCount = 0

on rlSetTexture(0):
    if vertexCounter >= elementCount*4: rlDrawRenderBatch()   # 只此一条提交路径
    currentTextureId = defaultTextureId

on rlVertex3f:
    d = draws[drawCounter-1]
    if vertexCounter > elementCount*4 - 4 and d.vertexCount % primSize(d.mode) == 0:
        checkLimit(primSize(d.mode) + 1)
    write slot vertexCounter; vertexCounter += 1; d.vertexCount += 1

on rlDrawRenderBatch:
    offset = 0
    for k in 0..drawCounter-1:
        emit(draws[k], offset)                    # 只对 vertexCount>0 的 draw 真正 glDraw*
        offset += draws[k].vertexCount + draws[k].vertexAlignment
    vertexCounter = 0; drawCounter = 1
    reset draws[i].{mode, vertexCount, textureId}; activeTextureId[] = 0   # vertexAlignment 不清（3164-3169）
```

（`src/rlgl.h:1669-1671`、`1471-1473`；`checkLimit` 见 `3205-3228`。）

### 3.2 最小可复算示例（默认容量，可直接手算）

若不想改宏，用默认 `8192` elements（32768 槽）也能复算，只是数字大。把 1.1 的序列里 G4 的 `DrawLineV` 换成 `DrawTriangleLines`（6 顶点 LINES），并把每处 `rlSetTexture(texture.id)` 放在 `rlBegin` 之前（即 `rlSetTexture` 先于 `rlBegin`，和 1.1 的顺序一致）。每组结束时 `vertexCounter` 依次为 **2 → 8 → 12 → 18 → 24 → 26**，`drawCounter` 累加到 5，最大 26 远小于 32768，整个帧只在 `EndDrawing` 提交一次（`rlCheckRenderBatchLimit` 从未触发）。这几次增长里的 `+2` 都来自对齐：G2 打开时 LINES 的 2 个顶点补成 4 槽，G5 打开时 LINES 的 6 个顶点补成 8 槽。注意这个序列只是一次小规模示例；换成大量文本或高分段圆（`DrawCircleGradient` 一次 108 顶点，`src/rshapes.c:1632-1645`）时 `vertexCounter` 会快速逼近容量。这说明：**默认容量下，普通 2D 场景的提交大多来自"耗光 32768 槽"，而小场景里能看到的主要是 shader/混合/相机/渲染目标/裁剪的模式切换**（2.3 第 3 条）。

要观察容量提交，有两个办法：

1. 把 `RL_DEFAULT_BATCH_BUFFER_ELEMENTS` 改小（如 4 或 8）重新编译——本报告 1.3 的数字就是这样得到的。
2. 在同一帧内画足够多的图元（`8192` 个 QUAD 或 32768 个顶点）——`DrawTextEx` 长字符串、`DrawCircleSector` 高分段最容易触发。

### 3.3 常见误解与纠正

| 误解 | 实际 |
|---|---|
| "改 `textureId` 才切 draw，改 mode 也切 draw" | `rlBegin` 改 mode **不**切 draw（除非 `vertexCount>0` 且需要对齐），只改当前 draw 的 `mode` 字段（`1478`）。真正切 draw 的是 `rlSetTexture` 的纹理不等（`1660`）。 |
| "`rlSetTexture(0)` 与 `rlSetTexture(id>0)` 走同一套逻辑" | 两条路径不同：`id==0` 只在满缓冲时提交（`1646-1650`，不做对齐）；`id>0` 走"对齐 + 切 draw + `check(alignment)`"（`1660-1687`）。 |
| "`rlSetTexture(0)` 只是解绑" | 除了解绑 unit 0，它在 `vertexCounter >= elementCount*4` 时会提交整批（`1646-1650`），这是库内**唯一**一条"满缓冲"提交路径。 |
| "附加采样器在整帧有效" | 只在**单批次**有效：`rlDrawRenderBatch` 末尾清空 `activeTextureId[]`（`3172`），且只有 `vertexCounter>0` 时才绑（`3034`）。 |
| "对齐空洞是零成本" | 空洞占容量（`1675` 推 `vertexCounter`）且把后续 draw 的偏移推后（`3130`）。LINES 段只要顶点数不是 4 的倍数就必然留 2 个空槽。 |
| "顶点数 = 容量" | 容量单位是 **element=QUAD=4 顶点**；`rlCheckRenderBatchLimit` 用 `elementCount*4`（`3211`）。 |
| "draw call 上限也是 256 个顶点" | `RL_DEFAULT_BATCH_DRAWCALLS = 256` 是 draw call 数（`220`），与顶点容量独立。超了在 `1476/1684` 整批提交。 |

### 3.4 如何实测验证

- **看批次提交次数**：在 `rlDrawRenderBatch`（`2958`）里插桩，打印 `batch->drawCounter`、`RLGL.State.vertexCounter`、每个 `draws[i].{mode,vertexCount,vertexAlignment,textureId}`。
- **看 GL 调用**：用 RenderDoc / apitrace 抓帧，对照 1.3 两张表的 `glDrawArrays`/`glDrawElements` 的 `first`/`count`/`offset`。`first` 应等于表里"槽"列的下界。
- **看附加采样器**：抓帧时看 `GL_TEXTURE1..4` 的 `glBindTexture` 次数与绑定的纹理 id，应只在每个"有几何的批次"开始时出现一次；批次之间不应有新绑定（说明 `activeTextureId[]` 被清空且未重新登记）。
- **看对齐空洞**：在 `rlVertex3f` 写槽时断言 `vertexCounter` 只增 1、且 `vertexAlignment` 涉及的槽从未被写。可对照 `src/rlgl.h:2802-2805` 的初始化值（全 0）验证空洞内容。
- **看 `currentDepth`**：`rlEnd` 每调用一次把 `currentDepth` 加 `1/20000`（`1487-1490`），`rlVertex2f` 用它作为 z（`1559-1562`）。这是 2D 图元在批次内保持调用顺序的隐性机制，与顶点/纹理归属无关，但如果调用方手工传 `rlVertex3f` 的 z，就会覆盖它。

---

## 4. 结论摘要

1. **提交由两个独立维度驱动**：
   - 空间：`rlCheckRenderBatchLimit` 的 `vertexCounter + n >= elementCount*4`（`3210`），从 `rlSetTexture` 对齐（`1673`）、`rlVertex3f` 守卫（`1512`）、`rlSetTexture(0)` 满缓冲（`1646`）、draw call 上限（`1476/1684`）四处进入。
   - 材质/状态：纹理变化在库内切 draw（`1660`）；shader/混合/相机/渲染目标/裁剪的切换由 core 模块**整批**提交（`src/rlgl.h:4597/2123`，`src/rcore.c:932/953/992/1008/1039/1086/1113/887`）。
2. **组间空间**：`vertexAlignment` 是"被打断那一段的尾巴"，写给被打断的 draw（`1669-1671`），由 `vertexOffset` 在下一段前加进去（`3130`）。空洞占容量、不被绘制、不跨批次回收。
3. **几何归属的新旧分界**：以 `rlDrawRenderBatch` 执行的那一刻为界；`vertexCounter` 归零（`3154`）之后写入的顶点属于新批次。G5 的矩形属于批次 #2，G4 的线属于批次 #1，尽管它们在调用序列上相邻。
4. **主贴图有效期 = 单段**（`draws[i].textureId`，逐段 `glBindTexture`，`3114`）；**附加采样器有效期 = 单批次**（`activeTextureId[]`，逐批次绑定一次 `3098-3105`，批次末清零 `3172`）。
5. **库内状态决定**：draw 切分、偏移、提交时机、采样器 unit 分配、批次末清零。
   **驱动/着色器输入决定**：`glUniform1i` 是否生效、location 是否有效、超出 4 个附加采样器时的静默失败、unit 1..4 的残留绑定、纹理内容可见性。
6. **调用方维持一致性**：每个批次前重新登记附加采样器；不可入批的状态变化走 core 的 `Begin*/End*`；附加采样器数量 ≤ 4；不依赖 unit 1..4 的残留绑定；注意 `EndMode3D` 会关闭深度测试。

---

## 附录 A：关键源码位置索引

| 主题 | 位置 |
|---|---|
| 批次容量与默认值 | `src/rlgl.h:203-225` |
| `rlVertexBuffer` / `rlDrawCall` / `rlRenderBatch` | `src/rlgl.h:386-429` |
| `activeTextureId[]` | `src/rlgl.h:1055` |
| `rlBegin` | `src/rlgl.h:1452-1482` |
| `rlEnd`（仅推进 `currentDepth`，不做提交） | `src/rlgl.h:1485-1491` |
| `rlVertex3f`（写入 + 容量守卫） | `src/rlgl.h:1495-1555` |
| `rlSetTexture`（两条提交路径） | `src/rlgl.h:1638-1691` |
| `rlSetBlendMode`（前置提交） | `src/rlgl.h:2118-2123` |
| 默认纹理 = 1×1 白 | `src/rlgl.h:2280-2283` |
| 索引表初始化（`4k..4k+3`） | `src/rlgl.h:2809-2820` |
| `rlDrawRenderBatch` 上传 / 绘制 / 复位 | `src/rlgl.h:2958-3180`（绘制循环 `3111-3131`；附加采样器绑定 `3098-3105`；`texture0` 绑定 `3114`；复位 `3154/3157/3168/3172/3175`） |
| `rlCheckRenderBatchLimit` | `src/rlgl.h:3205-3228` |
| `rlSetUniformSampler` | `src/rlgl.h:4564-4589` |
| `rlSetShader`（前置提交） | `src/rlgl.h:4592-4602` |
| `SetShaderValueTexture` | `src/rcore.c:1392-1400` |
| `SetShapesTexture` / shapes 默认纹理 | `src/rshapes.c:87-114`、`src/rcore.c:689-706` |
| `DrawRectanglePro`（QUADS 分支） | `src/rshapes.c:659-682` |
| `DrawLineV` | `src/rshapes.c:200-208` |
| `DrawTriangleLines` | `src/rshapes.c:365-378` |
| `DrawTextCodepoint` → `DrawTexturePro` | `src/rtext.c:1282-1303`、`src/rtextures.c:4876-4956` |
| core 中的整批提交点 | `src/rcore.c:887, 932, 953, 992, 1008, 1039, 1086, 1113`；`src/rlgl.h:2123, 4597` |

## 附录 B：复算脚本说明

第 1 节的数字由一份逐行对照 `rlBegin`/`rlSetTexture`/`rlVertex3f`/`rlCheckRenderBatchLimit`/`rlDrawRenderBatch` 的 Python 状态机模拟得到（`RL_DEFAULT_BATCH_BUFFER_ELEMENTS = 4`，即 16 槽）。模拟器只实现 GL3.3 路径的记账部分，不含 GL 调用。用它复算 1.2 的表格可逐步打印每次 API 后的 `vertexCounter` / `drawCounter` / 当前 draw 的 `{mode, vertexCount, vertexAlignment, textureId}`，以及每次提交的触发点与提交后的区间划分。
