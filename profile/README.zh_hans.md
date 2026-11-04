# 星光 · 星光投影

<table align="right">
  <tr>
    <td align="right">
      <img src="assets/mascot.png" width="190" alt="星光"><br>
      <sub>立绘：紫芈</sub>
    </td>
  </tr>
</table>

[English](README.md) | [简体中文](README.zh_hans.md)

星光投影把 Minecraft 基岩版原生 API 投影成 .NET 绑定。它只负责“原生事实 → C ABI / 托管投影”，不负责插件生命周期、框架本身或游戏逻辑。libTooling 前端把 C++ 声明与调试符号转成带版本的中间表示；发射器再把 IR 转成贴近原生风格的 Interop 程序集与 C ABI binding shim，shim 加载器在运行时按模块偏移、vtable 槽或导出符号解析。

三条互操作面共用同一条 IR 管线：Bedrock 互操作面在默认分支来自 BDS 1.21.3.01 的符号，在 `v1.26.60` 分支来自 BDS 1.26.60.6 的头文件树；LeviLamina（LLAPI）与 Endstone 互操作面来自各自公开的头文件。每个仓库都带自己的覆盖率契约，CI 在每次推送时对照发射产物校验该契约。

**前端**：LLVM 23.1.4 libTooling · IR 1.6.0 · 生成器 0.3.0  
**平台**：Windows x64 · Linux x64 · `win-x64-server` 与 `linux-x64-server` 配置  
**载体**：BDS 1.21.3.01（直连 MCAPI）· LeviLamina 26.60（LLAPI）· Endstone 0.11

## 覆盖

覆盖数字取自各互操作面提交的 `coverage/manifest.jsonl`。「成员」与「excluded」按清单中不同的成员 id 计数（含枚举常量）；「stub」为发射器保留为带原因 `NotSupportedException` stub 的成员；「可调用」为成员减 stub。excluded 为私有成员。

「比例」是「可调用」占已绑定成员的比例。「净比例」是「可调用」占全部声明成员（含私有）的比例，即调用方真正能调到的互操作面份额。

| 互操作面 | 来源 | 成员 | 可调用 | stub | excluded | 比例 | 净比例 |
|---|---|---:|---:|---:|---:|---:|---:|
| Endstone | Endstone 头文件 | 1,968 | 1,825 | 143 | 268 | 92.7% | **81.6%** |
| Bedrock（MCAPI） | 1.21.3.01 调试符号与 BDS 头文件 | 76,578 | 42,289 | 34,289 | 4,051 | 55.2% | **52.4%** |
| Bedrock（MCAPI，1.26.60.6） | 1.26.60.6 头文件 IR 与 LeviLamina 头文件 | 101,794 | 91,230 | 10,564 | 580 | 89.6% | **89.1%** |
| LeviLamina（LLAPI） | LeviLamina 头文件 | 7,718 | 5,816 | 1,902 | 259 | 75.4% | **71.4%** |

Bedrock 投影 6,433 个类型声明（6,221 个类、211 个 struct、1 个 union）与 704 个枚举；5,952 个「既有符号又有头文件声明」的类型全部入选，符号覆盖对这批类型是完整的——57,731 / 57,731。其余 MCAPI stub 主要是无符号或无 vtable 槽的成员（15,602）、无符号的构造函数（10,828）与没有可访问析构的类（2,270）。

1.26.60.6 面由 LeviLamina tag `v26.60.0`（commit `072b9ffe0e`）的 5,169 个头文件发射（另加固定 leveldb 1.23 包中的 `leveldb/status.h`）：12,138 个声明投影为 9,184 个互操作类型与 155,734 个垫片函数。成员在运行期经 `pl_resolve_symbol` 按名解析：72,997 个修饰名中 6,286 个在 1.26.60.6 服务器中可解析（8.6%）；56,391 个按名垫片函数中 5,715 个可解析（10.1%）。该载体锁定的运行期数据快照（`bedrockdata v26.60.6-server.5`）覆盖的符号表远小于 1.26.51.1 的：prelink 从该数据中映射出 72,997 个同名符号中的 6,286 个（server.1 时为 6,044 个，与游戏内命中完全一致），而从 1.26.51.1 数据中映射出 42,782 个，说明缩减跨版本存在；26.60.6 客户端快照暴露 6,081 个、26.51.1 为 42,953 个，也同样跨客户端与服务端。126,222 个非 stub 成员中另有 93,925 个（74.4%）的每个 shim 调用要么是本地实现、要么可解析，占全部发射成员的 68.7%；该数字未建模对象可获得性，是上限。

### MCAPI 符号覆盖（1.21.3.01）

上面的 MCAPI 数字描述的是已选类集。符号覆盖描述的是该类集对已发现服务端互操作面的到达程度。

发现结果记录 23,409 个 RTTI 拼写，但大部分是标准库实例化、Itanium 编码的 RTTI 名与匿名命名空间产物：1,599 个落在库命名空间，12,829 个解析不到头文件声明。剩下的 8,981 个拼写归并为 5,952 个「同时有符号与头文件声明」的类型。

| 度量 | 已覆盖 | 已发现 | 覆盖率 |
|---|---:|---:|---:|
| 类 | 5,952 | 5,952 | 100% |
| 公开符号 | 57,731 | 57,731 | **100%** |

`BindingGenerator/artifacts/symbol-classes-1.21.3.tsv` 是发现清单，Bedrock 仓库的 `scripts/classes-1.21.3.txt` 是其中被选中的子集。只有新增类时覆盖率才会变动。

LLAPI 的 stub 集中在模板与所有权处理：不可访问或不完整的模板参数（584）、无可见定义的模板构造（442）、模板特化的赋值运算符（424）与没有可访问析构的类（189）。

## 互操作面

| 仓 | 作用 |
|---|---|
| [BedrockBinding](https://github.com/HosihikariProjection/BedrockBinding) | BDS 1.26.60.6（`v1.26.60`）的 MCAPI Interop 与 binding shim（另保留较早的 BDS 1.21.3.01 / `v1.21.3` 面）；26.60 面按名解析 |
| [LeviLaminaBinding](https://github.com/HosihikariProjection/LeviLaminaBinding) | 基于 LeviLamina 的 LLAPI Interop 与 binding shim |
| [EndstoneBinding](https://github.com/HosihikariProjection/EndstoneBinding) | Endstone C++ API 的 Interop 与 binding shim |
| [BindingGenerator](https://github.com/HosihikariProjection/BindingGenerator) | libTooling 前端、IR 发射器、符号发现与校验工具 |
| [BindingIr](https://github.com/HosihikariProjection/BindingIr) | IR schema、记录模型、JSONL 读写、规范哈希与校验器 |
| [BindingAbi](https://github.com/HosihikariProjection/BindingAbi) | C ABI 头、值类型与 shim 加载器 |

## 支持的 ABI

标量与枚举底层类型；`std::string` 与 `std::string_view` 的值与 const 引用；单层 `T*` 与 `T&` 以 `void*` 传递；按值 record 走寄存器、隐藏返回缓冲区或按名解析的拷贝构造函数；按值返回的签名省略默认模板实参时（`Bedrock::Result<T>`）绑定到已记录的特化；`optional_ref<T>`、`std::vector<T>`、`std::optional<T>`、`std::shared_ptr<T>` 与 `std::weak_ptr<T>`；类型未投影的字段以只读 `nint` 暴露存储地址。

枚举声明投影为携带求值结果的 C# 枚举。命名空间级函数——包括经载体导入库调用的 out-of-line 函数——投影为 `Functions` 静态类，命名空间级比较运算符投影为 `operator_equal` 形式的静态方法。没有可用成员的类型保留类型身份并带 `[Obsolete]`。

map（需具名 key/value 类型）、回调、variant 与自定义拥有型包装仍为 stub。非平凡可复制且缺少可解析拷贝构造的 record 保持 stub。

## 数据来源

符号数据取自专有的 1.21.3.01 `bedrock_server.pdb`、`bedrock_server.exe` 与 `bedrock_server_symbols.debug`，只发布派生事实与哈希。Bedrock 的头文件补充是 LeviLamina 转载的 BDS 头，只用来提供声明与布局，不为它发射任何东西。LLAPI 互操作面解析的是 LeviLamina 26.60 的头。各互操作面把生成器版本、IR manifest 哈希与载体哈希记录在 `coverage/provenance.json`，`scripts/verify_generated.py` 校验来源、覆盖基线、shim 数量与每个生成文件的哈希。

## 许可与信任边界

IR、ABI 与三条绑定互操作面为 LGPL-3.0-or-later；生成器为 GPL-3.0。绑定经 binding shim 在服务器进程内运行，ABI 匹配错误会让服务器进程崩溃，而不是抛出托管异常。
