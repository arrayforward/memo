# Seed Archive v2 设计说明书

**版本**:0.2.0-draft
**读者**:实现工程师(含 AI 编程助手)
**目标**:按照本文档,无需任何额外上下文即可独立实现完整的 Seed Archive 系统(打包器、解释器、验证器)。

---

## 0. 文档约定

- 所有多字节整数均为**小端序**(little-endian),除非特别说明。
- 所有"32 位整数"均指二进制补码表示的有符号整数,运算溢出按模 2^32 回绕。
- 字节串写作十六进制序列,如 `01 13 00 04 00 00 00`。
- 本文档中所有格式定义均为**规范性**(必须严格实现),所有动机说明均为**信息性**(可不读)。
- 实现完成的判定标准:第 11 章的全部验收测试通过。任何"看起来更合理"的偏离都不允许。

---

## 1. 系统是什么

Seed Archive 是一种**自描述归档容器格式**。一个 `.bin` 档案文件包含:

1. 一段纯 ASCII 明文头,用英文教会读者执行一个极小的通用计算语言(SKI 图重写机);
2. 若干二进制节区,其中的编解码器是**程序**(SKI 图项 / SeedVM 字节码),数据是**负载**;
3. 内置测试对,使读者能机械地验证自己的重建是否正确。

设计公理(理解这些有助于把握细节取舍,但不影响实现):

- A1:自然语言只出现在 ASCII 头中;第 2 层起全部是可机械验证的程序与数据。
- A2:每一层都是上一层的负载。
- A3:ASCII 节区头是路标,二进制只出现在被 ASCII 宣告过的区域。
- A4:档案内任何机器的行为必须全定义:无"未定义行为",所有错误路径有明确错误码。

---

## 2. 总体结构

```
┌─────────────────────────────────────────────┐
│ ASCII 头(明文规格,以 "END\n" 行结束)        │  第1层:教学
├─────────────────────────────────────────────┤
│ SEC 节区头(ASCII) + 二进制负载              │  第2层:程序
│ SEC 节区头(ASCII) + 二进制负载              │  第3层:数据
│ ... (任意多个节区)                           │
├─────────────────────────────────────────────┤
│ MANIFEST(ASCII 清单)                        │
│ END-ARCHIVE                                  │
└─────────────────────────────────────────────┘
```

解析规则:

1. 从头读取,找到第一个只含 `END` 的行(其后紧跟一个 `\n`),之前是 ASCII 头。
2. 之后按节区循环:读取一行 ASCII(以 `\n` 结尾),若内容为 `MANIFEST` 则进入清单;否则必须是合法的 SEC 头(见第 7 章),随后紧跟 `len` 字节的二进制负载。
3. `MANIFEST` 之后为 ASCII 行序列,直到一行 `END-ARCHIVE`,档案结束。

**硬性规则**:
- 二进制负载不允许出现在任何 SEC 头之前。
- 节区之间无填充、无对齐,严格首尾相接。
- 所有 ASCII 行以 `\n`(0x0A)结尾,不允许 `\r\n`。

---

## 3. ASCII 头规格

ASCII 头是固定模板(允许填入实际数值,不允许改动结构)。完整模板如下,其中 `<...>` 为占位符:

```
SELF-DESCRIBING ARCHIVE v2. Payload follows line END.
This file explains itself. Everything after END is binary sections,
each announced by an ASCII line. CRC32 below is IEEE 802.3 (zlib crc32),
printed as decimal.

LAYER A - TERM LANGUAGE (SKI graph machine)
Term encoding (prefix, depth-first, binary):
  00 = integer literal, next 4 bytes little-endian
  01 = application node, two subtrees follow
  02 = duplication node, one subtree follows (shared; see rule DUP)
  10 = atom S | 11 = atom K | 12 = atom I | 13 = atom INC
  14 = atom ADD | 15 = atom MUL | 16 = atom LT
Reduction (leftmost-outermost spine, repeat to normal form):
  ((K x) y) = x ; (I x) = x ; (((S f) g) x) = ((f x) (g x))
  (INC n) = n+1 ; (ADD a b) = a+b ; (MUL a b) = a*b
  (LT a b) = 1 if a<b else 0   (32-bit two's complement, wrap on overflow)
  DUP node: evaluate the shared subtree at most once, then share result.
VERIFY (decode+reduce; these are ground truth):
  ex1 <len1> bytes -> 41 | ex2 <len2> bytes -> 7 | ex3 <len3> bytes -> 5

LAYER B - SEEDVM (bytecode machine; decoders are shipped as bytecode)
Full spec: 8-byte fixed instructions, 16 x 32-bit registers.
Authoritative semantics = the term-language interpreter program in
section <interp_sec>. A prose summary follows in this header is NOT
normative; if unsure, execute the interpreter.

SECTIONS: each begins with one ASCII line:
  SEC id=<n> type=<t> codec=<c> len=<bytes> crc32=<decimal>
payload follows immediately. Types: spec | program | data | test | font.
Codecs: none | rle | dpcm | seedvm | ski | raw.
Archive ends with line MANIFEST, index lines, then END-ARCHIVE.
CRC32(all bytes after END up to MANIFEST) = <crc>
END
```

实现要求:

- 模板文字逐字使用(占位符除外)。这套措辞经过压缩与消歧,改动会引入误读风险。
- `<len1/2/3>`、`<interp_sec>`、`<crc>` 由打包器填实际值。
- 头总长度目标 ≤ 2500 字节。

---

## 4. 第 1 层:SKI 图项语言完整规格

### 4.1 编码(字节级)

图项是二叉树的**前缀深度优先**序列化:

| 首字节 | 含义 | 后继字节 |
|---|---|---|
| `0x00` | 整数字面量 | 4 字节小端无符号 32 位(按补码解释) |
| `0x01` | 应用节点 (APP) | 左子树,然后右子树 |
| `0x02` | 复制节点 (DUP) | 1 个子树 |
| `0x10`–`0x16` | 原子 S,K,I,INC,ADD,MUL,LT | 无 |
| `0xFF` | 共享引用 | 1 字节序号 n,指代本项内第 n 个已解码的 DUP 子树(见 4.4) |

其他字节为非法,解码器必须报错。

### 4.2 求值语义

项 `(f x)` 表示 APP(f, x)。求值 = 反复应用"最左最外"归约,直到无规则可用:

```
R-K:   ((K x) y)       → x
R-I:   (I x)           → x
R-S:   (((S f) g) x)   → ((f x) (g x))     [x 以 DUP 节点共享,而非复制]
R-INC: (INC ⟨n⟩)       → ⟨n+1⟩
R-ADD: ((ADD ⟨a⟩) ⟨b⟩) → ⟨a+b⟩             [模 2^32 回绕]
R-MUL: ((MUL ⟨a⟩) ⟨b⟩) → ⟨a·b⟩             [模 2^32 回绕]
R-LT:  ((LT ⟨a⟩) ⟨b⟩)  → ⟨1⟩ 若 a<b(有符号)否则 ⟨0⟩
R-DUP: DUP 节点的子树最多求值一次,所有共享者看到同一结果(惰性+记忆化)
```

实现要点:

- 用图(带引用计数的节点池)而非树表示;R-S 的 x 必须共享,否则指数爆炸。
- "最左最外":沿 APP 链左子树下溯收集脊柱,栈式参数列表,匹配头部原子与参数个数。
- 整数是**弱头范式**:字面量即值,不再归约。
- 实现必须设置步数上限(建议 ≥ 10^7),超限报错而非死循环。

### 4.3 强制测试对(验收基准)

| 编号 | 字节序列 | 期望归约结果 |
|---|---|---|
| T-A1 | `01 01 11 00 29 00 00 00 00 63 00 00 00` | 整数 41 |
| T-A2 | `01 01 01 10 11 11 00 07 00 00 00` | 整数 7 |
| T-A3 | `01 13 00 04 00 00 00` | 整数 5 |
| T-A4 | `01 01 14 00 06 00 00 00 00 07 00 00 00` | 整数 13 |
| T-A5 | `01 01 15 00 06 00 00 00 00 07 00 00 00` | 整数 42 |
| T-A6 | `01 01 16 00 03 00 00 00 00 07 00 00 00` | 整数 1 |
| T-A7 | `01 01 16 00 09 00 00 00 00 07 00 00 00` | 整数 0 |

T-A1~A3 同时写入 ASCII 头(即 `<len1..3>` = 13, 11, 7)。

### 4.4 共享引用(0xFF)——可选实现

打包器**必须**能对重复的 DUP 子树发出 `0xFF n`(n 为本项内第 n 个 DUP 子树,从 0 计);解释器可以仅实现"见到 0xFF 即报错"的最小版本,但必须在文档中声明。v2 打包器默认关闭共享引用(生成自包含项)。

---

## 5. 第 2 层:SeedVM v0.2 指令集完整规格

### 5.1 机器模型

```
状态:
  R[0..15]   16 个 32 位补码寄存器
  PC         字节地址,始终为 8 的倍数
  MEM        64 MiB 扁平字节数组,初始全零(允许稀疏实现)
  RSTK       返回地址栈,程序不可读写,容量 ≥ 65536
无标志寄存器。比较结果写寄存器(0 或 1)。
```

### 5.2 指令格式:定长 8 字节

```
字节:  [0]      [1]  [2]  [3]  [4..7]
       opcode   rd   ra   rb   imm32(小端)
```

- 每条指令 8 字节;取指后默认 `PC += 8`。
- 指令不使用的字段**必须为 0**。实现可(建议)在打包时校验。
- 跳转目标为**绝对字节地址**,且必须是 8 的倍数,否则 err=5。

### 5.3 入口与停机约定

- 启动:`PC = 0`;`R0 = in_len`;`R1 = 0`(输入基址,固定为 0);其余寄存器为 0;输入数据已置于 `MEM[0, in_len)`。
- 正常停机(执行 `HALT`):`R1` = 输出起始地址,`R0` = 输出长度;输出 = `MEM[R1, R1+R0)`,err=0。
- 异常停机:任何错误 → err = 错误码,无输出。

### 5.4 指令表(22 条)

| opcode | 助记符 | 字段 | 语义 |
|---|---|---|---|
| `0x00` | HALT | — | 正常停机(见 5.3) |
| `0x01` | LI | rd, imm | `R[rd] = imm32`(原样,补码) |
| `0x02` | MOV | rd, ra | `R[rd] = R[ra]` |
| `0x08` | ADD | rd, ra, rb | `R[rd] = R[ra] + R[rb]`(回绕) |
| `0x09` | SUB | rd, ra, rb | `R[rd] = R[ra] - R[rb]`(回绕) |
| `0x0A` | MUL | rd, ra, rb | `R[rd] = R[ra] * R[rb]`(回绕) |
| `0x0B` | DIV | rd, ra, rb | 有符号除,向零取整;`rb==0` → err=1;`INT_MIN / -1` → `INT_MIN` |
| `0x0C` | MOD | rd, ra, rb | 余数符号同被除数;`rb==0` → err=1;`INT_MIN % -1` → `0` |
| `0x0D` | AND | rd, ra, rb | 按位与 |
| `0x0E` | OR | rd, ra, rb | 按位或 |
| `0x0F` | XOR | rd, ra, rb | 按位异或 |
| `0x10` | NOT | rd, ra | `R[rd] = ~R[ra]` |
| `0x11` | SHL | rd, ra, rb | 左移,计数 = `rb & 31` |
| `0x12` | SHR | rd, ra, rb | 逻辑右移,计数 = `rb & 31` |
| `0x13` | SAR | rd, ra, rb | 算术右移,计数 = `rb & 31` |
| `0x14` | EQ | rd, ra, rb | `R[rd] = (R[ra]==R[rb]) ? 1 : 0` |
| `0x15` | LT | rd, ra, rb | 有符号小于 → 1/0 |
| `0x16` | LTU | rd, ra, rb | 无符号小于 → 1/0 |
| `0x18` | LD8 | rd, ra, imm | `R[rd] = MEM[R[ra]+imm]`(零扩展);越界 → err=2 |
| `0x19` | LD32 | rd, ra, imm | 4 字节小端加载;地址计算按无符号模 2^32;越界 → err=2 |
| `0x1A` | ST8 | rd, ra, imm | `MEM[R[ra]+imm] = R[rd] & 0xFF`;越界 → err=2 |
| `0x1B` | ST32 | rd, ra, imm | 4 字节小端存储;越界 → err=2 |
| `0x1C` | JMP | imm | `PC = imm`;imm 非 8 倍数 → err=5 |
| `0x1D` | JZ | ra, imm | `R[ra]==0` 则 `PC = imm`(对齐检查同上) |
| `0x1E` | CALL | imm | RSTK 压入 `PC+8`,`PC = imm`;栈满 → err=6 |
| `0x1F` | RET | — | RSTK 空 → err=4;否则 `PC = 弹出` |
| `0x3F` | HALT_ERR | imm | 异常停机,`err = imm & 0xFF`(不得为 0) |

未列出的 opcode → err=3。

### 5.5 错误码总表

| 码 | 含义 |
|---|---|
| 0 | 正常停机 |
| 1 | 除零 |
| 2 | 内存越界 |
| 3 | 非法 opcode |
| 4 | 返回栈空时执行 RET |
| 5 | 跳转目标非 8 对齐 |
| 6 | 返回栈溢出 |

### 5.6 边界行为的进一步钉死(防歧义)

- 地址表达式 `R[ra] + imm` 按**无符号 32 位回绕**计算后再判越界。
- 输入区与输出区允许重叠,无特殊保护;行为即内存语义。
- `LD8` 永远零扩展;需要符号扩展时用 `SHL 24` + `SAR 24` 合成。
- `HALT` 时 `R1 + R0` 越界 → err=2(输出声明也受内存检查)。
- 单条指令内 `rd == ra == rb` 合法,语义为先读全部源再写目的。

---

## 6. SeedVM 的 SKI 解释器(第 2 层关键程序)

档案的"权威语义"是一个用 SKI 图项编写的 SeedVM 解释器,作为 `type=program codec=ski` 的节区存放。

### 6.1 功能要求(行为等价即可,内部实现自由)

输入(在图项世界中的表示):

- 一段 SeedVM 字节码(编码为 Church 编码的字节列表,或等价的打包整数序列——实现者自选,但必须在节区头注释所用表示);
- 一段输入字节(同上表示)。

输出:二元组 `(err, 输出字节列表)`。

必须逐条实现 5.4 的 22 条指令与 5.5 的错误码。

### 6.2 指令级测试向量(26 条,全部强制)

测试向量以 `type=test` 节区存放,格式为 ASCII 行序列:

```
TEST <id> ins=<8字节指令hex> code=<程序hex> in=<输入hex> expect_err=<d> expect_out=<hex>
```

必须覆盖:

- 22 条指令各至少 1 条正常路径;
- err=1, 2, 3, 4, 5, 6 各至少 1 条(用 HALT_ERR 模拟 err 透传不算,必须真实触发);
- `DIV INT_MIN/-1`、`SHL 计数 33`(验证 `&31`)、`LD8` 零扩展、`rd==ra==rb` 各 1 条。

验收:任何独立实现的 SeedVM 解释器跑完全部向量,结果逐字节一致。

---

## 7. 节区与清单格式

### 7.1 SEC 头(单行 ASCII)

```
SEC id=<n> type=<t> codec=<c> len=<bytes> crc32=<decimal>\n
```

- `id`:从 0 开始的整数,递增,不重复;
- `type`:`spec | program | data | test | font`;
- `codec`:`none | ski | seedvm | rle | dpcm | raw`;
- `len`:紧随其后的二进制负载字节数;
- `crc32`:负载的 IEEE CRC32(zlib.crc32),十进制打印。

解析器必须拒绝未知 type/codec(为将来扩展留的是 HALT_ERR 风格的显式失败,而非静默跳过)。

### 7.2 编解码器语义

| codec | 负载含义 | 解码方式 |
|---|---|---|
| `none` | 已是最终形式(ASCII 规格文本、测试向量) | 直接读 |
| `raw` | 原始文件封装(B/C 级数据) | 不解码,直接还原为原文件 |
| `ski` | SKI 图项(第 4 章) | SKI 求值器 |
| `seedvm` | SeedVM 字节码 | SeedVM 解释器 |
| `rle` | RLE 压缩数据 | 第 8 章的 RLE 解码程序 |
| `dpcm` | DPCM 音频 | 第 8 章的 DPCM 解码程序 |

`rle` / `dpcm` 的解码器**必须以 `type=program codec=seedvm` 的节区形式存在于同一档案中**;SEC 头的 codec 字段只是索引,清单(MANIFEST)中给出解码器节区 id 的映射:

```
CODEC rle = program section <id>
CODEC dpcm = program section <id>
```

### 7.3 MANIFEST 格式

```
MANIFEST
CODEC rle = program section 3
CODEC dpcm = program section 4
FILE id=5 name="docs/spec-jpeg.txt" type=spec codec=none len=81234 crc32=...
FILE id=6 name="img/disc.rle" type=data codec=rle len=404 crc32=...
TESTSUITE section=2 vectors=26
END-ARCHIVE
```

- 每节区一行 `FILE`;name 用双引号,内部仅允许可打印 ASCII(0x20–0x7E,不含 `"`)。
- MANIFEST 是冗余索引:即使丢失,扫描全部 SEC 头也可重建。

---

## 8. 标准编解码器规格(随档程序)

### 8.1 RLE(图像/通用)

内存布局(输入,32 位字序列,小端):

```
word[0] = 宽度 w
word[1] = 高度 h
word[2] = 游程对数 p
word[3 + 2i]     = count_i
word[3 + 2i + 1] = value_i   (0 ≤ i < p)
```

输出 = 逐对展开的字节流,长度必须 = `w*h`,否则程序必须 `HALT_ERR 7`。
SeedVM 参考实现 ≤ 30 条指令。测试向量:49 对、32×32 圆盘,输出首字节 0,中央字节 255。

### 8.2 DPCM(音频)

输入布局:

```
word[0] = 样本数 n
word[1] = 采样率
word[2] = 阶数 order(v2 固定为 1)
word[3] = 初值 s0
word[4+i] = 第 i 个残差(有符号)
```

输出 = n 个 16 位小端样本:`s_i = clip16(s_{i-1} + r_i)`,clip 到 [-32768, 32767]。
测试向量:64 样本正弦四分之一周期,给出全部输出哈希与前 8 个样本真值。

### 8.3 未来编解码器的加入规则

任何新 codec 必须同时提供:(a) SeedVM 解码程序节区;(b) ≥ 3 条测试向量;(c) MANIFEST 的 CODEC 映射行。缺一项,打包器必须拒绝。

---

## 9. 数据分级与打包策略

| 级 | 内容 | 处理 | codec |
|---|---|---|---|
| A | 文明重启核心(教科书、图纸、规格文档、本说明书的自引用副本) | 转码为档案原生格式 | rle/dpcm/seedvm |
| B | 重要存量(影像、数据集、文献) | 原样封装 + 格式规格随葬(type=spec)+ 测试对 | raw |
| C | 海量冷数据 | 原样封装,仅登记 | raw |

B 级每出现一种新文件格式,规格区必须新增对应 `type=spec` 节区与该格式的"最小可解码子集"标注(规格文本中的一行 `MINIMAL: ...`)。

---

## 10. 打包器(pack)规格

```
pack <目录> -o <档案.bin> [--level-map <文件>] 
```

行为:

1. 扫描目录,按 level-map(或默认规则:`.txt/.md` → A级转码尝试,`.jpg/.png/.mp4/...` → B级)分级;
2. 生成 ASCII 头(第 3 章模板,填入实际值);
3. 依次写入:SKI 解释器节区、SeedVM 测试向量节区、SeedVM 编解码器节区、spec 节区、数据节区;
4. 生成 MANIFEST,计算头的全局 CRC 填回头部;
5. 自检:对档案立即执行一次完整解包验证(第 11 章),失败则删除输出并退出非零。

性能要求:吞吐 ≥ 磁盘顺序读速的 80%;除 A 级外不得执行任何解码/编码。

---

## 11. 验证器(verify)与验收测试

```
verify <档案.bin>
```

按序执行,全部 PASS 方可输出 `ARCHIVE OK`:

1. **结构**:END 定位、SEC 头合法性、负载长度吻合、MANIFEST 与 SEC 序列一致;
2. **CRC**:每个节区负载 CRC32;头的全局 CRC;
3. **SKI 层**:第 4.3 节 T-A1~A7 全部归约到期望值;
4. **SeedVM 层**:test 节区全部向量通过(26+ 条);
5. **端到端**:用档案内 SeedVM 编解码器程序解出每个 rle/dpcm 数据节区,输出与 test 节区中的已知输出哈希一致;
6. **独立性**:步骤 3-5 使用的解释器必须是从零按第 4、5 章规格实现的(即:verify 自带参考实现,不得复用打包器内部代码)。

里程碑建议:

- **M1**:SKI 求值器 + T-A1~A7;
- **M2**:SeedVM 解释器(宿主语言版)+ 26 条指令向量;
- **M3**:容器读写 + CRC + MANIFEST;
- **M4**:SeedVM 的 SKI 解释器 + 双层端到端;
- **M5**:pack/verify 工具链 + 演示档案(圆盘图 + 1 秒正弦音 + 本说明书)。

---

## 12. 常见实现陷阱(给实现者的检查单)

1. 小端:所有 imm32、LD32/ST32、整数原子,全部小端。写一条"存储 0x01020304 后逐字节读出"的测试。
2. `DIV/MOD` 是**向零取整**且符号规则特殊,先写 `INT_MIN/-1`、`-7/2`、`7/-2` 三个用例再写指令。
3. 移位计数必须 `& 31`,且 SHR 是逻辑移(无符号解释)。
4. 指令未用字段必须为 0;跳转地址必须 8 对齐。
5. SKI 的 R-S 必须共享 x(DUP),不允许朴素复制——朴素复制在 Church 数值上会指数爆炸,这是功能错误而非性能问题。
6. `HALT` 的输出声明 `(R1, R0)` 同样要越界检查。
7. ASCII 行只允许 `\n`,解析时遇到 `\r` 直接报错。
8. CRC32 一律 IEEE(zlib 默认),不是 CRC32C。
9. 图项解码遇到未知首字节必须报错,不允许"跳过"。
10. 测试对不是示例,是**规范的一部分**:实现与文档冲突时,以测试对的期望输出为准,并回报文档缺陷。

---

## 附录 A:最小完整档案示例(十六进制骨架)

```
[ASCII 头, 第3章模板, ~2400 字节]
END
SEC id=0 type=program codec=ski len=<n0> crc32=<c0>        ← SeedVM 的 SKI 解释器
<n0 字节图项>
SEC id=1 type=test codec=none len=<n1> crc32=<c1>          ← 26 条指令向量(ASCII)
<n1 字节>
SEC id=2 type=program codec=seedvm len=<n2> crc32=<c2>     ← RLE 解码器(≤240 字节)
<n2 字节>
SEC id=3 type=data codec=rle len=404 crc32=<c3>            ← 32×32 圆盘 RLE
<404 字节: 20 00 00 00 20 00 00 00 31 00 00 00 ... >
MANIFEST
CODEC rle = program section 2
FILE id=0 name="seedvm-interp.ski" type=program codec=ski ...
FILE id=1 name="seedvm-tests.txt" type=test codec=none ...
FILE id=2 name="rle-decoder.vm" type=program codec=seedvm ...
FILE id=3 name="img/disc.rle" type=data codec=rle ...
TESTSUITE section=1 vectors=26
END-ARCHIVE
```

(完)
