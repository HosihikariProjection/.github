# Hosihikari Projection

<table align="right">
  <tr>
    <td align="right">
      <img src="assets/mascot.png" width="190" alt="Hosi Hikari"><br>
      <sub>Art by 紫芈</sub>
    </td>
  </tr>
</table>

[English](README.md) | [简体中文](README.zh_hans.md)

Hosihikari Projection (星光投影) projects the Minecraft Bedrock native APIs into .NET bindings. It owns the native-to-C-ABI and native-to-managed projection only; plugin lifecycles, the framework, and game logic live elsewhere. A libTooling frontend turns C++ declarations and debug symbols into a versioned intermediate representation; emitters turn that IR into a native-style Interop assembly and a C ABI binding shim, which the shim loader resolves at run time by module offset, vtable slot, or exported symbol.

Three interop surfaces are generated from the same IR pipeline: the Bedrock interop surface from BDS 1.21.3.01 symbols on the default branch and from the BDS 1.26.60.6 header tree on `v1.26.60`, plus the LeviLamina (LLAPI) and Endstone interop surfaces from their public headers. Each repository carries its own coverage contract, and CI verifies that contract against the emitted sources on every push.

**Frontend**: LLVM 23.1.4 libTooling · IR 1.6.0 · Generator 0.3.0  
**Platforms**: Windows x64 · Linux x64 · `win-x64-server` and `linux-x64-server` configurations  
**Carriers**: BDS 1.21.3.01 (direct MCAPI) · LeviLamina 26.51 (LLAPI) · Endstone 0.11

## Coverage

Coverage counts come from the committed `coverage/manifest.jsonl` of each interop surface. "Members" and "Excluded" count distinct member ids in the manifest, including enumerator constants; "Stubbed" counts members the emitter keeps as reasoned `NotSupportedException` stubs, and "Callable" is Members minus Stubbed. Excluded covers private members.

**Rate** is callable over bound members. **Net rate** is callable over all declared members, private ones included; it is the share of the whole interop surface that a caller can actually invoke.

| Interop surface | Source | Members | Callable | Stubbed | Excluded | Rate | Net rate |
|---|---|---:|---:|---:|---:|---:|---:|
| Endstone | Endstone headers | 1,968 | 1,825 | 143 | 268 | 92.7% | **81.6%** |
| Bedrock (MCAPI) | 1.21.3.01 debug symbols and BDS headers | 76,578 | 42,289 | 34,289 | 4,051 | 55.2% | **52.4%** |
| Bedrock (MCAPI, 1.26.60.6) | 1.26.60.6 header IR and LeviLamina headers | 101,794 | 91,230 | 10,564 | 580 | 89.6% | **89.1%** |
| LeviLamina (LLAPI) | LeviLamina headers | 6,278 | 4,443 | 1,835 | 237 | 70.8% | **68.2%** |

Bedrock projects 6,433 type declarations (6,221 classes, 211 structs, 1 union) and 704 enums; every one of the 5,952 discovered types that has both a symbol and a header declaration is selected, so symbol coverage for that set is complete — 57,731 of 57,731 symbols. Remaining MCAPI stubs are dominated by members with no symbol or vtable slot (15,602), constructors with no symbol (10,828), and classes with no accessible destructor (2,270).

The 1.26.60.6 surface is emitted from 5,169 LeviLamina headers at tag `v26.60.0` (commit `072b9ffe0e`; plus `leveldb/status.h`): 12,138 declarations project into 9,210 interop types and 155,734 shim functions. Members resolve at run time through `pl_resolve_symbol`: 6,286 of 72,997 mangled names resolve in the 1.26.60.6 server (8.6%), and 5,715 of the 56,391 name-resolving shim functions resolve (10.1%). The run-time data snapshot pinned by this carrier (`bedrockdata v26.60.6-server.5`) covers a much smaller symbol table: prelink maps 6,286 of the same 72,997 names from it (6,044 at server.1), matching the in-game count exactly, versus 42,782 from the 1.26.51.1 data, so the reduction spans versions. The 26.60.6 client snapshot exposes 6,081 names against 42,953 for 26.51.1, spanning both platforms as well. Of the 126,222 non-stub members, 93,925 (74.4%) additionally have every shim call either local or resolvable; 68.7% of all emitted members. That figure does not model object reachability and is an upper bound.

### MCAPI symbol coverage (1.21.3.01)

The MCAPI numbers above describe the selected set. Symbol coverage describes how much of the discovered server interop surface that set reaches.

Discovery records 23,409 RTTI spellings, but most are standard-library instantiations, Itanium-encoded RTTI names, and anonymous-namespace artifacts: 1,599 fall in library namespaces and 12,829 do not resolve to a header declaration. The remaining 8,981 spellings collapse into the 5,952 types that have both a symbol and a header declaration.

| Measure | Covered | Discovered | Coverage |
|---|---:|---:|---:|
| Classes | 5,952 | 5,952 | 100% |
| Public symbols | 57,731 | 57,731 | **100%** |

`BindingGenerator/artifacts/symbol-classes-1.21.3.tsv` is the discovery list; `scripts/classes-1.21.3.txt` in the Bedrock repository is the selected subset. Coverage moves only when a class is added.

LLAPI stubs are concentrated in template handling and ownership: inaccessible or incomplete template arguments (584), template constructors without a visible definition (442), assignment operators on specializations (424), and classes with no accessible destructor (189).

## Interop surfaces

| Repository | Purpose |
|---|---|
| [BedrockBinding](https://github.com/HosihikariProjection/BedrockBinding) | MCAPI Interop and symbol-dispatched binding shim for BDS 1.26.60.6 (`v1.26.60`), plus the older BDS 1.21.3.01 (`v1.21.3`) surface; the 26.60 surface is name-resolved |
| [LeviLaminaBinding](https://github.com/HosihikariProjection/LeviLaminaBinding) | LLAPI Interop and binding shim over LeviLamina |
| [EndstoneBinding](https://github.com/HosihikariProjection/EndstoneBinding) | Endstone C++ API Interop and binding shim |
| [BindingGenerator](https://github.com/HosihikariProjection/BindingGenerator) | libTooling frontend, IR emitters, symbol discovery, validation tooling |
| [BindingIr](https://github.com/HosihikariProjection/BindingIr) | IR schema, records, JSONL reader/writer, canonical hash, validator |
| [BindingAbi](https://github.com/HosihikariProjection/BindingAbi) | C ABI header, value types, shim loader |

## Supported ABI

Scalars and enum underlying types; `std::string` and `std::string_view` values and const references; single-level `T*` and `T&` as `void*`; by-value records through registers, the hidden return buffer, or a copy constructor resolved by name; by-value results whose spelling omits defaulted template arguments (`Bedrock::Result<T>`) bound to the recorded specialization; `optional_ref<T>`, `std::vector<T>`, `std::optional<T>`, `std::shared_ptr<T>` and `std::weak_ptr<T>`; fields whose type is not projected surface their storage address as a readonly `nint`.

Enum declarations project as C# enums carrying their evaluated enumerators. Namespace-level functions — including carrier-exported out-of-line functions, which link through the carrier import library — become `Functions` static classes, and namespace-level comparison operators become static `operator_equal`-style methods. A type with no usable member keeps its type identity and carries `[Obsolete]`.

Maps with named key/value types, callbacks, variants, and custom owning wrappers remain stubs. A record that is not trivially copyable and has no resolvable copy constructor stays stubbed.

## Provenance

Symbol data is recovered from the proprietary 1.21.3.01 `bedrock_server.pdb`, `bedrock_server.exe`, and `bedrock_server_symbols.debug`; only derived facts and hashes are published. The Bedrock header supplement is the BDS header text that LeviLamina republishes; it only supplies declarations and layouts, and nothing is emitted on its behalf. The LLAPI surface is parsed from LeviLamina 26.51 headers. Each interop surface records the generator version, IR manifest hash, and carrier hashes in `coverage/provenance.json`, and `scripts/verify_generated.py` checks provenance, the coverage baseline, shim counts, and every generated file hash.

## License and trust boundary

LGPL-3.0-or-later for the IR, ABI, and the three binding interop surfaces; the generator is GPL-3.0. Bindings run in the server process through the binding shim, so a wrong ABI match faults the server process rather than throwing a managed exception.
