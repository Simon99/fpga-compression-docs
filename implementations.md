---
layout: default
title: 推薦 C 實作
---

# 推薦 C 代碼實作

FPGA 開發的參考實現，適合作為 Verilog 轉換的基礎。

---

## 壓縮算法：zlib

### 基本資訊

| 項目 | 內容 |
|------|------|
| **GitHub** | [madler/zlib](https://github.com/madler/zlib) |
| **授權** | zlib License（非常寬鬆）|
| **語言** | C |
| **總代碼量** | ~15,000 行 |

### 核心文件

```
zlib/
├── deflate.c    # 壓縮核心（2186 行）⭐ 重點
├── inflate.c    # 解壓核心（1413 行）
├── trees.c      # Huffman 編碼（1119 行）
├── adler32.c    # 校驗和
├── crc32.c      # CRC32 計算
└── zutil.c      # 工具函數
```

### FPGA 轉換要點

**deflate.c 狀態機：**
```c
// 主要狀態（適合直接轉 FSM）
#define INIT_STATE    0
#define BUSY_STATE    1  
#define FINISH_STATE  2

// 壓縮等級對應的策略
level 1-3:  快速匹配（簡單 hash）
level 4-6:  標準匹配
level 7-9:  最大匹配（複雜但壓縮率高）
```

**關鍵函數：**
```c
// 初始化
deflateInit2(strm, level, method, windowBits, memLevel, strategy);

// 設置低內存模式（FPGA 推薦）
windowBits = 9;  // 512B 窗口，而非默認 32KB

// 核心壓縮循環
deflate(strm, flush);
```

### 簡化範例

```c
#include <zlib.h>

// 低內存壓縮（適合 FPGA 參考）
int compress_low_mem(const uint8_t *in, size_t in_len,
                     uint8_t *out, size_t *out_len) {
    z_stream strm = {0};
    
    // windowBits=9 → 512B 窗口
    // memLevel=1  → 最小內存
    deflateInit2(&strm, 6, Z_DEFLATED, 9, 1, Z_DEFAULT_STRATEGY);
    
    strm.next_in = (uint8_t*)in;
    strm.avail_in = in_len;
    strm.next_out = out;
    strm.avail_out = *out_len;
    
    deflate(&strm, Z_FINISH);
    *out_len = strm.total_out;
    
    deflateEnd(&strm);
    return 0;
}
```

---

## 加密算法：tiny-AES-c

### 基本資訊

| 項目 | 內容 |
|------|------|
| **GitHub** | [kokke/tiny-AES-c](https://github.com/kokke/tiny-AES-c) |
| **授權** | Unlicense（公共領域）|
| **語言** | C |
| **總代碼量** | **572 行** ⭐ 極簡 |

### 文件結構

```
tiny-AES-c/
├── aes.c      # 完整實現（572 行）
├── aes.h      # API 定義
└── test.c     # 測試範例
```

### 支援模式

| 模式 | 宏定義 | 說明 |
|------|--------|------|
| ECB | `#define ECB 1` | 基礎模式（不推薦單獨使用）|
| CBC | `#define CBC 1` | 鏈接模式（常用）|
| CTR | `#define CTR 1` | 計數器模式（流加密）|

### 支援密鑰長度

```c
#define AES128 1  // 128-bit 密鑰，10 輪
#define AES192 1  // 192-bit 密鑰，12 輪
#define AES256 1  // 256-bit 密鑰，14 輪
```

### API 說明

```c
// 數據結構
struct AES_ctx {
    uint8_t RoundKey[176];  // AES-128: 176 bytes
    uint8_t Iv[16];         // 初始向量
};

// 初始化
void AES_init_ctx(struct AES_ctx* ctx, const uint8_t* key);
void AES_init_ctx_iv(struct AES_ctx* ctx, const uint8_t* key, const uint8_t* iv);

// ECB 模式（16 bytes 一次）
void AES_ECB_encrypt(const struct AES_ctx* ctx, uint8_t* buf);
void AES_ECB_decrypt(const struct AES_ctx* ctx, uint8_t* buf);

// CBC 模式（需要 padding）
void AES_CBC_encrypt_buffer(struct AES_ctx* ctx, uint8_t* buf, size_t len);
void AES_CBC_decrypt_buffer(struct AES_ctx* ctx, uint8_t* buf, size_t len);

// CTR 模式（無需 padding）
void AES_CTR_xcrypt_buffer(struct AES_ctx* ctx, uint8_t* buf, size_t len);
```

### 使用範例

```c
#include "aes.h"

void aes_example(void) {
    // 密鑰和 IV
    uint8_t key[16] = {0x00, 0x01, 0x02, ...};
    uint8_t iv[16]  = {0x00, 0x01, 0x02, ...};
    
    // 初始化
    struct AES_ctx ctx;
    AES_init_ctx_iv(&ctx, key, iv);
    
    // 加密
    uint8_t data[64] = "Hello, World!...";
    AES_CBC_encrypt_buffer(&ctx, data, 64);
    
    // 解密（重置 IV）
    AES_ctx_set_iv(&ctx, iv);
    AES_CBC_decrypt_buffer(&ctx, data, 64);
}
```

### FPGA 轉換優勢

1. **代碼極簡** — 只有 572 行，容易理解
2. **查表結構** — S-Box 直接映射到 Block RAM
3. **無動態分配** — 所有內存靜態分配
4. **純計算** — 無 I/O，純算術操作

---

## 非對稱加密：mbedtls

### 基本資訊

| 項目 | 內容 |
|------|------|
| **GitHub** | [Mbed-TLS/mbedtls](https://github.com/Mbed-TLS/mbedtls) |
| **授權** | Apache 2.0 / GPL 2.0 雙授權 |
| **語言** | C |
| **特點** | 工業級、模組化 |

### ⚠️ 注意事項

mbedtls 是**完整的 TLS 實現**，包含大量功能。對於 FPGA：
- 建議只提取需要的模組
- RSA 實現較複雜，可能需要 HLS 工具輔助

### RSA 相關模組

```
mbedtls/
├── tf-psa-crypto/        # 加密核心（子模組）
│   └── drivers/builtin/
│       └── src/
│           ├── bignum.c      # 大數運算 ⭐
│           ├── rsa.c         # RSA 核心
│           └── rsa_alt.c     # RSA 替代實現
└── library/
    └── pk.c                  # 公鑰封裝
```

### RSA 核心操作

```c
// 大數結構
typedef struct mbedtls_mpi {
    int s;              // 符號
    size_t n;           // limbs 數量
    mbedtls_mpi_uint *p; // limb 數組
} mbedtls_mpi;

// 核心運算（FPGA 重點）
mbedtls_mpi_mul_mpi()   // 大數乘法
mbedtls_mpi_mod_mpi()   // 模運算
mbedtls_mpi_exp_mod()   // 模指數（RSA 核心）
```

### 簡化 RSA 實現建議

對於 FPGA，建議使用更簡單的獨立實現：

| 替代方案 | GitHub | 說明 |
|----------|--------|------|
| mini-gmp | GNU 項目 | 最小化大數庫 |
| libtommath | [libtom/libtommath](https://github.com/libtom/libtommath) | 輕量數學庫 |
| BearSSL | [BearSSL](https://bearssl.org/) | 輕量 TLS |

### Montgomery 乘法（FPGA 核心）

```c
// Montgomery 模乘：計算 A * B * R^(-1) mod N
// 這是 RSA 硬件加速的關鍵

void montgomery_mul(mpi *result, const mpi *A, const mpi *B, 
                    const mpi *N, mpi_uint N_inv) {
    // 1. 計算 T = A * B
    // 2. 計算 m = (T * N_inv) mod R
    // 3. 計算 result = (T + m * N) / R
    // 4. 如果 result >= N，result -= N
}
```

---

## FPGA 參考實現

### 現成的 Verilog/VHDL 實現

| 算法 | 項目 | 說明 |
|------|------|------|
| gzip | [ZipCPU/gzip-fpga](https://github.com/ZipCPU/gzip-fpga) | 完整 DEFLATE |
| AES | [secworks/aes](https://github.com/secworks/aes) | 高質量 Verilog |
| RSA | [ultraembedded/rsa](https://github.com/ultraembedded/rsa) | 2048-bit RSA |
| SHA | [secworks/sha256](https://github.com/secworks/sha256) | SHA-256 |

### 轉換工具

| 工具 | 用途 |
|------|------|
| Vitis HLS | C → Verilog（Xilinx）|
| Verilator | Verilog 仿真 + C++ Testbench |
| Icarus Verilog | 開源仿真器 |
| GTKWave | 波形查看 |

---

## 快速開始

### 獲取代碼

```bash
# 壓縮
git clone https://github.com/madler/zlib.git

# AES
git clone https://github.com/kokke/tiny-AES-c.git

# RSA（如需完整 TLS）
git clone --recursive https://github.com/Mbed-TLS/mbedtls.git
```

### 編譯測試

```bash
# zlib
cd zlib && ./configure && make

# tiny-AES-c
cd tiny-AES-c && make

# mbedtls
cd mbedtls && cmake -B build && cmake --build build
```

---

[← 返回首頁](index.html) | [算法分析](algorithms.html)

*最後更新: 2026-02-14*
