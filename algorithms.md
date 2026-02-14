---
layout: default
title: 算法分析
---

# 算法分析與 FPGA 實現考量

## DEFLATE (gzip)

### 算法結構
```
DEFLATE = LZ77 + Huffman Coding
```

### LZ77 滑動窗口

| 參數 | 標準值 | 低內存值 | 說明 |
|------|--------|---------|------|
| wbits | 15 | 9 | 窗口大小位數 |
| 窗口大小 | 32 KB | 512 B | 2^wbits |
| BRAM 需求 | 高 | **極低** | FPGA 資源 |

### 狀態機設計

```
┌─────────┐
│  IDLE   │
└────┬────┘
     ↓
┌─────────┐     ┌──────────┐
│  SEARCH │────→│  MATCH   │
└────┬────┘     └────┬─────┘
     ↓               ↓
┌─────────┐     ┌──────────┐
│ LITERAL │     │  ENCODE  │
└────┬────┘     └────┬─────┘
     └───────┬───────┘
             ↓
       ┌──────────┐
       │  OUTPUT  │
       └──────────┘
```

### FPGA 實現要點
- 使用 Block RAM 實現滑動窗口
- Hash 表可用 distributed RAM
- 流水線化 Huffman 編碼器

---

## 🚫 為什麼 xz/zstd 不適合 FPGA？

### XZ (LZMA2) 的問題

#### 算法複雜度
- ❌ **Range Encoder** — 需要高精度除法運算，硬件實現昂貴
- ❌ **超大字典** — 默認 8MB，最大 1.5GB，遠超 FPGA BRAM 容量
- ❌ **自適應編碼器** — 動態調整概率參數，狀態機極度複雜
- ❌ **2304 種狀態組合** — 12 × 192 的上下文依賴

#### 資源需求估算
```
內存需求：  ~8MB+（滑動窗口 + 字典）
邏輯單元：  估計 >100K LUT（Xilinx 7 系列）
時鐘週期：  單個塊壓縮 >10M cycles
依賴鏈：    每個 symbol 必須等前一個完成
```

#### 致命問題：Range Coder
```
Range Encoding 每步需要：
1. 高精度乘法 (32×32 bit)
2. 除法運算（硬件最昂貴的操作）
3. 概率表更新
4. 狀態轉移

→ 無法有效流水線化
→ 吞吐量極低
```

---

### ZSTD 的問題

#### 算法複雜度
- ❌ **FSE (Finite State Entropy)** — ANS 變體，需要大型狀態轉換表
- ❌ **3 張動態 Huffman 表** — Literal Lengths / Match Lengths / Offsets
- ❌ **反向解碼** — 必須緩衝整個 block 才能開始解碼
- ❌ **動態表生成** — 每個 block 的編碼表都可能不同

#### 資源需求估算
```
內存需求：  128KB+ 字典 + 多張編碼表
狀態表：    每張 FSE 表 256-4096 entries
BRAM 端口：  需要同時訪問多張表（端口競爭）
Block 緩衝：  需要完整 block 才能解碼
```

#### 致命問題：無法流式處理
```
ZSTD Block 結構：
├── Block Header
├── Literals Section
│   ├── Huffman 表（需先完整讀取）
│   └── Literals 數據
├── Sequences Section
│   ├── 3 張 FSE 表（需先完整讀取）
│   └── Sequences 數據（反向編碼）
└── 必須解析完整 block header 才知道表結構

→ 無法邊收邊解
→ 需要大緩衝區
→ 延遲高
```

---

### 對比總結

| 特性 | gzip (wbits=9) | xz (LZMA2) | zstd |
|------|---------------|------------|------|
| 字典大小 | **512 B** | 8+ MB | 128+ KB |
| 狀態數 | ~10 | ~2300 | ~1000 |
| 編碼方式 | Huffman | Range | FSE |
| 硬件除法 | 不需要 | **需要** | 不需要 |
| 流式處理 | ✅ 可以 | ⚠️ 困難 | ❌ 不行 |
| 動態表 | 固定 | 自適應 | **每 block 變** |
| 典型 FPGA | 入門級可實現 | 高端都困難 | 中高端困難 |
| **結論** | ✅ 推薦 | ❌ 不適用 | ❌ 不適用 |

> **核心觀點：** 現代壓縮算法追求的是 CPU 上的壓縮率/速度平衡，
> 而非硬件友好性。DEFLATE 雖然「老」，但其簡單的 LZ77+Huffman 結構
> 恰好適合 FPGA 的並行流水線架構。

---

## 📊 壓縮算法梯度選擇

### bzip2：中間選項

如果需要比 gzip 更高的壓縮率，**bzip2** 是可考慮的中間方案：

| 特性 | 說明 |
|------|------|
| 算法 | BWT (Burrows-Wheeler) + MTF + Huffman |
| 壓縮率 | 85-88%（介於 gzip 和 xz 之間）|
| 複雜度 | 比 gzip 高，但比 LZMA 簡單很多 |
| FPGA 可行性 | ⭐⭐⭐ 中等（需要較多資源）|

**BWT 的特點：**
- Block-based 處理（典型 900KB block）
- 可並行化的排序操作
- 無需複雜的算術編碼

### 選擇建議

```
┌─────────────────────────────────────────────────────────┐
│            FPGA 壓縮算法梯度選擇                         │
├──────────┬────────────┬─────────────┬──────────────────┤
│ 等級     │ 算法        │ 資源需求    │ 適用場景          │
├──────────┼────────────┼─────────────┼──────────────────┤
│ 入門級   │ gzip       │ 512B 窗口   │ 資源極度受限      │
│          │ (wbits=9)  │             │ 簡單實現          │
├──────────┼────────────┼─────────────┼──────────────────┤
│ 標準級   │ gzip -9    │ 32KB 窗口   │ 通用方案          │
│          │            │             │ 成熟參考實現多    │
├──────────┼────────────┼─────────────┼──────────────────┤
│ 進階級   │ bzip2      │ ~1MB block  │ 需要更高壓縮率    │
│          │            │             │ 資源充足的 FPGA   │
├──────────┼────────────┼─────────────┼──────────────────┤
│ 不推薦   │ xz/zstd    │ 8MB+ 字典   │ ❌ 僅限軟件場景   │
│          │            │             │ FPGA 不現實       │
└──────────┴────────────┴─────────────┴──────────────────┘
```

### 適用場景總結

| 場景 | 推薦算法 | 原因 |
|------|---------|------|
| 資源受限 FPGA | gzip wbits=9 | 512B 窗口，極低資源 |
| 通用 FPGA 壓縮 | gzip -1/-9 | 成熟、參考多、夠用 |
| 高壓縮率需求 | bzip2 | 比 gzip 好 3-5%，仍可實現 |
| 極致壓縮率 | 軟件 xz/zstd | FPGA 不適合，交給 CPU |

---

## AES

### 算法結構
```
AES-128: 10 輪
AES-256: 14 輪

每輪:
  SubBytes → ShiftRows → MixColumns → AddRoundKey
```

### S-Box 實現

| 方式 | 資源 | 速度 |
|------|------|------|
| 查表 (LUT) | 256 entries | 1 cycle |
| 組合邏輯 | 較多 LUT | 1 cycle |
| 分時共享 | 較少 | 多 cycles |

### FPGA 實現要點
- S-Box 用 Block RAM 或 LUT
- 可展開所有輪次（高吞吐）
- 或用單輪重複（省資源）

### 參考實現
- [secworks/aes](https://github.com/secworks/aes) — 完整 Verilog

---

## RSA

### 算法結構
```
加密: C = M^e mod n
解密: M = C^d mod n
```

### 核心運算
- **Montgomery 乘法** — 模乘優化
- **模指數運算** — 平方-乘法算法

### FPGA 實現難度：高

| 挑戰 | 原因 |
|------|------|
| 大數運算 | 2048-bit 乘法器 |
| 資源消耗 | DSP/BRAM 需求高 |
| 時序收斂 | 長進位鏈 |

### 建議
- 使用 HLS 工具 (Vitis HLS)
- 或只實現 Montgomery 乘法器
- 考慮使用 FPGA 供應商 IP

---

## 工具鏈

### 開源仿真
```bash
# 安裝
sudo apt install verilator iverilog gtkwave

# 仿真
iverilog -o sim top.v testbench.v
vvp sim
gtkwave dump.vcd
```

### HLS 工具

| 工具 | 適用 | 授權 |
|------|------|------|
| Vitis HLS | Xilinx FPGA | 免費版可用 |
| Intel HLS | Intel FPGA | Quartus 內建 |
| Bambu | 通用 | 開源 |

---

## 參考資源

### 論文
- *A High-Performance FPGA-Based Implementation of the DEFLATE Compression Algorithm*
- *Efficient Hardware Implementation of AES*

### 開源項目
- [ZipCPU/gzip-fpga](https://github.com/ZipCPU/gzip-fpga)
- [secworks/aes](https://github.com/secworks/aes)
- [ultraembedded/rsa](https://github.com/ultraembedded/rsa)

---

[← 返回首頁](index.html)

*最後更新: 2026-02-14*
