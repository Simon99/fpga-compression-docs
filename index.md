---
layout: default
title: 首頁
---

# FPGA 壓縮算法研究

評估不同壓縮/加密算法在 FPGA 實現上的可行性。

## 📊 核心結論

| 演算法 | 壓縮率 | 內存需求 | FPGA 適用性 |
|--------|-------|---------|------------|
| gzip -1 (fast) | 78.9% | 32KB | ⚠️ 中等 |
| gzip -9 (best) | 85.7% | 32KB | ⚠️ 中等 |
| **gzip wbits=9** | 69.1% | **512B** | ✅ 最佳 |
| xz (LZMA2) | 91.9% | >1MB | ❌ 不適用 |
| zstd | 90.6% | >128KB | ❌ 不適用 |

## 🔑 關鍵發現

- **wbits=9 模式最適合 FPGA** — 滑動窗口僅需 512 bytes
- **gzip -9 vs -1** — 體積減少 32%，但時間增加 18 倍
- **xz/zstd 不建議** — 算法複雜，內存消耗高

## 📄 文檔

- [完整實驗報告](report.html) — 詳細測試數據與分析
- [算法分析](algorithms.html) — DEFLATE/AES/RSA 實現考量

## 🛠️ 相關資源

| 代碼庫 | 用途 |
|--------|------|
| [zlib](https://github.com/madler/zlib) | gzip/DEFLATE 官方實現 |
| [tiny-AES-c](https://github.com/kokke/tiny-AES-c) | AES 極簡 C 實現 |
| [mbedtls](https://github.com/Mbed-TLS/mbedtls) | RSA 工業級實現 |
| [gzip-fpga](https://github.com/ZipCPU/gzip-fpga) | FPGA gzip 參考實現 |
| [aes (secworks)](https://github.com/secworks/aes) | FPGA AES 參考實現 |

---

*最後更新: 2026-02-14*
