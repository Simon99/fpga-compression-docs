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
