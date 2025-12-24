# SoFixer 技术文档

> 从内存 dump 的 Android SO 文件修复工具深度解析

[English Version](#english-version)

---

## 目录

- [概述](#概述)
- [背景知识](#背景知识)
- [核心原理](#核心原理)
- [架构设计](#架构设计)
- [源码解析](#源码解析)
- [使用指南](#使用指南)
- [局限性与已知问题](#局限性与已知问题)

---

## 概述

SoFixer 是一个用于修复从 Android 进程内存中 dump 出来的 `.so` 文件的工具。

**核心问题**：从内存 dump 的 SO 文件缺少 Section Header Table (SHT)，导致 IDA Pro 等逆向工具无法正确解析符号表、重定位表等关键信息。

**解决方案**：利用 Program Header Table (PHT) 中的 `PT_DYNAMIC` 段信息，逆向重建 SHT。

---

## 背景知识

### ELF 文件结构

```
┌─────────────────────────────┐
│       ELF Header            │  ← 文件入口，描述整体结构
├─────────────────────────────┤
│   Program Header Table      │  ← 运行时视图（linker 使用）
│   (PHT / Segment Headers)   │
├─────────────────────────────┤
│                             │
│        Segments             │  ← 实际数据（代码、数据等）
│   (.text, .data, .rodata)   │
│                             │
├─────────────────────────────┤
│   Section Header Table      │  ← 链接时视图（调试器使用）
│   (SHT / Section Headers)   │
└─────────────────────────────┘
```

### 两种视图的区别

| 特性 | Program Headers (PHT) | Section Headers (SHT) |
|------|----------------------|----------------------|
| 用途 | 运行时加载 | 链接/调试 |
| 使用者 | Linker | 调试器、反编译器 |
| 必要性 | **必须** | 可选 |
| Dump 后 | 保留 | **丢失** |

### 为什么 SHT 会丢失？

Android linker 加载 SO 时：
1. 只读取 PHT 来确定如何映射内存
2. SHT 不参与运行时加载
3. 加载完成后，SHT 所在的文件区域可能被覆盖或未映射

因此，从内存 dump 的 SO 文件通常只有 PHT，没有 SHT。

---

## 核心原理

### 关键洞察

虽然 SHT 丢失了，但 `PT_DYNAMIC` 段包含了 linker 需要的所有动态链接信息：

```
PT_DYNAMIC 段包含的 DT_* 标签：
├── DT_SYMTAB   → 符号表地址
├── DT_STRTAB   → 字符串表地址
├── DT_STRSZ    → 字符串表大小
├── DT_HASH     → 哈希表地址
├── DT_REL      → 重定位表地址
├── DT_RELSZ    → 重定位表大小
├── DT_JMPREL   → PLT 重定位表地址
├── DT_PLTRELSZ → PLT 重定位表大小
├── DT_INIT_ARRAY → 初始化函数数组
├── DT_FINI_ARRAY → 析构函数数组
└── ...
```

**核心思路**：从 `PT_DYNAMIC` 提取这些地址，反向构造对应的 Section Headers。

### 重建流程

```
┌──────────────────┐
│  dump.so (残缺)   │
│  只有 PHT        │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  1. 解析 ELF     │  ElfReader::Load()
│     Header + PHT │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  2. 修复 PHT     │  ObElfReader::FixDumpSoPhdr()
│  p_offset=p_vaddr│  (内存布局 → 文件布局)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  3. 提取动态信息  │  ElfRebuilder::ReadSoInfo()
│  解析 PT_DYNAMIC │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  4. 重建 SHT     │  ElfRebuilder::RebuildShdr()
│  构造各 Section  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  5. 修复重定位    │  ElfRebuilder::RebuildRelocs()
│  还原相对地址    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  6. 输出文件     │  ElfRebuilder::RebuildFin()
│  拼接最终 SO    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   fixed.so       │
│   PHT + SHT 完整 │
└──────────────────┘
```

---

## 架构设计

### 类图

```
┌─────────────────────────────────────────────────────────┐
│                      FileReader                         │
│  - 文件读取封装                                          │
│  - Read(addr, len, offset)                              │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                      ElfReader                          │
│  - 解析 ELF Header                                      │
│  - 解析 Program Header Table                            │
│  - 加载 Segments 到内存                                  │
├─────────────────────────────────────────────────────────┤
│  + Load()                                               │
│  + ReadElfHeader()                                      │
│  + ReadProgramHeader()                                  │
│  + LoadSegments()                                       │
│  + GetDynamicSection()                                  │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     ObElfReader                         │
│  - 继承 ElfReader                                       │
│  - 处理混淆/损坏的 SO 文件                               │
│  - 支持从原始 SO 获取 dynamic 信息                       │
├─────────────────────────────────────────────────────────┤
│  + FixDumpSoPhdr()         // 修复 PHT                  │
│  + LoadDynamicSectionFromBaseSource()  // 从原 SO 读取  │
│  + haveDynamicSectionInLoadableSegment()               │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    ElfRebuilder                         │
│  - 核心重建逻辑                                          │
│  - 从 PT_DYNAMIC 提取信息                               │
│  - 构造 Section Header Table                            │
│  - 修复重定位表                                          │
├─────────────────────────────────────────────────────────┤
│  + Rebuild()               // 主入口                    │
│  + RebuildPhdr()           // 修复 PHT                  │
│  + ReadSoInfo()            // 提取动态信息               │
│  + RebuildShdr()           // 重建 SHT                  │
│  + RebuildRelocs()         // 修复重定位                 │
│  + RebuildFin()            // 生成最终文件               │
└─────────────────────────────────────────────────────────┘
```

### 数据结构

```cpp
// soinfo - 存储从 PT_DYNAMIC 提取的所有信息
struct soinfo {
    // 基础信息
    uint8_t* base;           // SO 基地址
    uint8_t* load_bias;      // 加载偏移
    
    // 符号表
    Elf_Sym* symtab;         // 符号表指针
    const char* strtab;      // 字符串表指针
    size_t strtabsize;       // 字符串表大小
    
    // 哈希表（用于符号查找）
    uint8_t* hash;
    size_t nbucket;
    size_t nchain;
    unsigned* bucket;
    unsigned* chain;
    
    // 重定位表
    Elf_Rel* rel;            // .rel.dyn
    size_t rel_count;
    Elf_Rel* plt_rel;        // .rel.plt
    size_t plt_rel_count;
    
    // 初始化/析构函数
    void** init_array;
    size_t init_array_count;
    void** fini_array;
    size_t fini_array_count;
    
    // ARM 异常处理
    Elf_Addr* ARM_exidx;
    size_t ARM_exidx_count;
    
    // ... 更多字段
};
```

---

## 源码解析

### 1. PHT 修复 (ObElfReader::FixDumpSoPhdr)

内存 dump 后，PHT 中的 `p_offset`（文件偏移）已经没有意义，需要修复为 `p_vaddr`（虚拟地址）：

```cpp
void ObElfReader::FixDumpSoPhdr() {
    // 对于 dump 的 SO，文件偏移 = 虚拟地址
    auto phdr = phdr_table_;
    for(auto i = 0; i < phdr_num_; i++) {
        phdr->p_paddr = phdr->p_vaddr;
        phdr->p_filesz = phdr->p_memsz;  // 文件大小 = 内存大小
        phdr->p_offset = phdr->p_vaddr;  // 关键：偏移 = 虚拟地址
        phdr++;
    }
}
```

**原理**：内存 dump 时，整个内存映像被连续保存，所以文件偏移就等于虚拟地址。

### 2. 动态信息提取 (ElfRebuilder::ReadSoInfo)

遍历 `PT_DYNAMIC` 段，提取所有 `DT_*` 标签：

```cpp
bool ElfRebuilder::ReadSoInfo() {
    // 获取 dynamic 段
    elf_reader_->GetDynamicSection(&si.dynamic, &si.dynamic_count, &si.dynamic_flags);
    
    // 遍历所有 DT_* 条目
    for (Elf_Dyn* d = si.dynamic; d->d_tag != DT_NULL; ++d) {
        switch(d->d_tag) {
            case DT_SYMTAB:
                si.symtab = (Elf_Sym*)(base + d->d_un.d_ptr);
                break;
            case DT_STRTAB:
                si.strtab = (const char*)(base + d->d_un.d_ptr);
                break;
            case DT_HASH:
                si.hash = d->d_un.d_ptr + base;
                si.nbucket = ((unsigned*)(base + d->d_un.d_ptr))[0];
                si.nchain = ((unsigned*)(base + d->d_un.d_ptr))[1];
                break;
            case DT_REL:
                si.rel = (Elf_Rel*)(base + d->d_un.d_ptr);
                break;
            case DT_JMPREL:
                si.plt_rel = (Elf_Rel*)(base + d->d_un.d_ptr);
                break;
            // ... 更多标签
        }
    }
}
```

### 3. SHT 重建 (ElfRebuilder::RebuildShdr)

根据提取的地址，构造各个 Section Header：

```cpp
bool ElfRebuilder::RebuildShdr() {
    // 构造 .dynsym section
    if(si.symtab != nullptr) {
        Elf_Shdr shdr;
        shdr.sh_name = shstrtab.length();
        shstrtab.append(".dynsym");
        shstrtab.push_back('\0');
        
        shdr.sh_type = SHT_DYNSYM;
        shdr.sh_flags = SHF_ALLOC;
        shdr.sh_addr = (uintptr_t)si.symtab - (uintptr_t)base;
        shdr.sh_offset = shdr.sh_addr;  // 偏移 = 地址
        // ...
        shdrs.push_back(shdr);
    }
    
    // 构造 .dynstr section
    if(si.strtab != nullptr) {
        Elf_Shdr shdr;
        shdr.sh_name = shstrtab.length();
        shstrtab.append(".dynstr");
        // ...
        shdr.sh_addr = (uintptr_t)si.strtab - (uintptr_t)base;
        shdr.sh_size = si.strtabsize;
        shdrs.push_back(shdr);
    }
    
    // 类似地构造：
    // .hash, .rel.dyn, .rel.plt, .plt, .text, 
    // .ARM.exidx, .init_array, .fini_array, .dynamic, .data
    // ...
    
    // 最后排序并修复 section 间的链接关系
    // ...
}
```

### 4. 重定位修复 (ElfRebuilder::RebuildRelocs)

内存中的重定位已经被 linker 处理过（绝对地址），需要还原为相对地址：

```cpp
template <bool isRela>
void ElfRebuilder::relocate(uint8_t* base, Elf_Rel* rel, Elf_Addr dump_base) {
    auto type = ELF32_R_TYPE(rel->r_info);
    auto sym = ELF32_R_SYM(rel->r_info);
    auto prel = reinterpret_cast<Elf_Addr*>(base + rel->r_offset);
    
    switch (type) {
        case R_ARM_RELATIVE:
            // 相对重定位：减去 dump 基址还原
            *prel = *prel - dump_base;
            break;
            
        case 0x401:  // R_AARCH64_GLOB_DAT
        case 0x402:  // R_AARCH64_JUMP_SLOT
            // 导入符号重定位
            auto syminfo = si.symtab[sym];
            if (syminfo.st_value != 0) {
                *prel = syminfo.st_value;
            } else {
                // 外部符号：指向导入表
                const char* symname = si.strtab + syminfo.st_name;
                int nIndex = GetIndexOfImports(symname);
                *prel = load_size + nIndex * sizeof(*prel);
            }
            break;
    }
}
```

### 5. 导入表修复

原始实现有 bug：按重定位表顺序递增分配导入表索引，但重定位表顺序可能与导入表顺序不一致。

**修复方案**：先保存导入符号顺序，再按名称查找正确索引：

```cpp
void ElfRebuilder::SaveImportsymNames() {
    Elf_Sym* symtab = si.symtab;
    int nIndex = 0;
    bool start = false;
    
    while (true) {
        Elf_Sym sym = symtab[nIndex];
        
        // 跳过开头的空符号
        if (sym.st_name == 0 && !start) {
            nIndex++;
            continue;
        }
        start = true;
        
        // st_value != 0 表示进入了非导入符号区域
        if (sym.st_name != 0 && sym.st_value != 0) {
            break;
        }
        
        const char* symname = si.strtab + sym.st_name;
        mImports.push_back(symname);  // 按顺序保存
        nIndex++;
    }
}

int ElfRebuilder::GetIndexOfImports(std::string symname) {
    for (int i = 0; i < mImports.size(); i++) {
        if (mImports[i] == symname) {
            return i;
        }
    }
    return -1;
}
```

### 6. 最终文件生成 (ElfRebuilder::RebuildFin)

```cpp
bool ElfRebuilder::RebuildFin() {
    auto load_size = si.max_load - si.min_load;
    
    // 计算最终文件大小
    rebuild_size = load_size 
                 + shstrtab.length()           // .shstrtab 内容
                 + shdrs.size() * sizeof(Elf_Shdr);  // Section Headers
    
    rebuild_data = new uint8_t[rebuild_size];
    
    // 1. 复制原始内存数据
    memcpy(rebuild_data, si.load_bias, load_size);
    
    // 2. 追加 .shstrtab
    memcpy(rebuild_data + load_size, shstrtab.c_str(), shstrtab.length());
    
    // 3. 追加 Section Headers
    auto shdr_off = load_size + shstrtab.length();
    memcpy(rebuild_data + shdr_off, &shdrs[0], shdrs.size() * sizeof(Elf_Shdr));
    
    // 4. 修复 ELF Header
    auto ehdr = *elf_reader_->record_ehdr();
    ehdr.e_shnum = shdrs.size();
    ehdr.e_shoff = shdr_off;
    ehdr.e_shstrndx = sSHSTRTAB;
    memcpy(rebuild_data, &ehdr, sizeof(Elf_Ehdr));
    
    return true;
}
```

**最终文件布局**：

```
┌─────────────────────────┐  0x0
│      ELF Header         │  (修复 e_shoff, e_shnum, e_shstrndx)
├─────────────────────────┤
│   Program Headers       │
├─────────────────────────┤
│                         │
│   原始内存数据           │  (load_size 字节)
│   (.text, .data, etc)   │
│                         │
├─────────────────────────┤  load_size
│      .shstrtab          │  (section 名称字符串)
├─────────────────────────┤  load_size + shstrtab.length()
│   Section Headers       │  (重建的 SHT)
└─────────────────────────┘
```

---

## 使用指南

### 编译

```bash
mkdir build && cd build

# 32 位 SO
cmake ..
make

# 64 位 SO
cmake -DSO_64=ON ..
make
```

### Dump 内存

使用 IDA 脚本从调试中的进程 dump SO：

```python
import idaapi

start_address = 0x7DB078B000  # SO 加载基址
end_address = 0x7DB08DE000    # SO 结束地址
data_length = end_address - start_address

fp = open('dump.so', 'wb')
cur = 0
chunk_size = 0x100000  # 1MB 分块读取

while cur < data_length:
    to_read = min(chunk_size, data_length - cur)
    data = idaapi.dbg_read_memory(start_address + cur, to_read)
    fp.write(data)
    cur += to_read

fp.close()
```

### 修复 SO

```bash
# 基本用法
./SoFixer64 -s dump.so -o fixed.so -m 0x7DB078B000

# 参数说明
# -s  源文件（dump 的 SO）
# -o  输出文件（修复后的 SO）
# -m  dump 时的内存基址（16进制）
# -d  显示调试信息
# -b  原始 SO 文件（用于获取 dynamic 信息，实验性功能）
```

---

## 局限性与已知问题

### 1. .text 段边界不精确

`PT_DYNAMIC` 不包含 `.text` 段的精确边界，只能通过相邻 section 估算：

```cpp
// .text 大小 = 下一个 section 地址 - .text 起始地址
shdr.sh_size = shdrs[sNext].sh_addr - shdrs[sTEXTTAB].sh_addr;
```

### 2. 混淆 SO 可能失败

某些加壳/混淆方案会：
- 破坏 `PT_DYNAMIC` 段
- 加密/隐藏动态信息
- 运行时动态解密

这种情况下 SoFixer 无法工作。

### 3. 不支持的重定位类型

目前只处理了常见的重定位类型：
- `R_ARM_RELATIVE` / `R_386_RELATIVE`
- `R_AARCH64_GLOB_DAT` (0x401)
- `R_AARCH64_JUMP_SLOT` (0x402)
- `R_AARCH64_RELATIVE` (0x403)

其他类型会被忽略。

### 4. 32/64 位分开编译

需要根据目标 SO 的架构分别编译 32 位和 64 位版本。

---

## 参考资料

- [TK SO 修复原理](http://bbs.pediy.com/thread-191649.htm)
- [ELF 文件格式规范](https://refspecs.linuxfoundation.org/elf/elf.pdf)
- [Android Linker 源码](https://android.googlesource.com/platform/bionic/+/master/linker/)

---

<a name="english-version"></a>
# English Version

## Overview

SoFixer is a tool for repairing Android SO files dumped from process memory.

**Problem**: Memory-dumped SO files lack Section Header Table (SHT), causing reverse engineering tools like IDA Pro to fail parsing symbols and relocations.

**Solution**: Reconstruct SHT using information from `PT_DYNAMIC` segment in Program Header Table (PHT).

## How It Works

1. **Parse ELF Header & PHT** - Load the dumped SO file
2. **Fix PHT** - Set `p_offset = p_vaddr` (memory layout → file layout)
3. **Extract Dynamic Info** - Parse `PT_DYNAMIC` to get symbol table, string table, relocations, etc.
4. **Rebuild SHT** - Construct section headers from extracted addresses
5. **Fix Relocations** - Subtract dump base address to restore relative addresses
6. **Generate Output** - Append `.shstrtab` and section headers to create final file

## Usage

```bash
# Build
cmake -DSO_64=ON .. && make

# Fix SO
./SoFixer64 -s dump.so -o fixed.so -m 0x7DB078B000
```

## Limitations

- `.text` section boundaries are estimated, not precise
- Obfuscated/packed SOs may not work
- Only common relocation types are supported
- Separate builds required for 32-bit and 64-bit targets

---

*Author: F8LEFT (Original), Import fix contribution by community*
*License: See LICENSE file*
