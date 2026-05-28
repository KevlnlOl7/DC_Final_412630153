# 通用無損壓縮演算法 — Claude UCC

> 淡江大學 資料壓縮期末報告
> 資管 3C｜412630153｜張傢寧

---

## 演算法架構

採用 **Context Mixing + Arithmetic Coding**，所有檔案（`.txt`、`.bmp`）統一視為 1D Byte Stream，不做任何前處理。

```
Input Bytes → 21 個預測模型 → Mixer → APM ×5 → Arithmetic Coding → 壓縮輸出
```

### 21 個預測模型組成

| 類型 | 數量 | 說明 |
|------|------|------|
| Order-1~8 Context SM | 8 | 看前 n 個 bytes 預測下一個 bit |
| Sparse Context SM | 6 | 跳格取樣，捕捉週期性規律 |
| MatchModel | 3 | 長/中/短距離字串比對 |
| Run / Word / Indirect | 3 | 重複序列、英文單字、間接 context |
| Order-0 Histogram | 1 | 全域 byte 頻率統計 |

---

## 核心優化

### Ring Buffer — O(n) → O(1)

```python
# 原本（Gemini 版本）
history = []
history.append(new_byte)
if len(history) > 16:
    del history[0]        # ❌ O(n)，每次搬移所有元素

# 改良（Claude）
ring = bytearray(32)      # 固定大小，不需搬移
ring[rpos & 31] = byte    # ✅ O(1)
val  = ring[(rpos-k) & 31]# ✅ O(1)
```

### Cython — Python 直譯 → C 編譯

```python
# cython: boundscheck=False
# cython: wraparound=False
# cython: initializedcheck=False
# cython: cdivision=True

cdef int predict(self, int bc):   # C 型別宣告
    cdef int nb, temp, conf       # 迴圈速度接近原生 C
    ...
```

---

## 實驗結果

對比：Gemini UCC（同架構，不同實作）

| 檔案 | 原始 | Gemini | Claude v3 | 結果 |
|------|------|--------|-----------|------|
| test1.txt | 35 B | 1.06x | 1.06x | ≈ |
| test2.txt | 2,638 B | 2.58x | 2.57x | ≈ |
| test3.txt | 5,349 B | 2.65x | 2.63x | ≈ |
| Cameraman.bmp | 66,616 B | 1.73x | 1.72x | ≈ |
| Lenna.bmp | 263,224 B | 1.55x | 1.54x | ≈ |
| **壓縮時間** | — | **33.32s** | **19.56s** | 🏆 快 1.70x |

壓縮率兩者差距 < 0.5%，**速度快 1.70x**。

> **為什麼壓縮率差不多？**
> Context Mixing 是邊壓縮邊學習的演算法，測資最大只有 263KB，模型還在暖機階段。
> 實驗將 Cameraman.bmp 重複疊加 ×4 後，Claude v3 領先 Gemini +0.34x，架構差異才真正顯現。

---

## 專案結構

```
compression-final/
├── README.md
├── TestCases/                           # 測試資料
├── DC_Final_Cython_SubmitVer.ipynb      # 主程式，含編譯與測試
├── DC_Final_Cython_ComparisonVer.ipynb  # 對照組的程式
```

---

## 使用方式

### 0. 環境需求（Cython 需要 C 編譯器）

**Linux / macOS**
```bash
# 通常已內建 gcc，確認版本即可
gcc --version
```

**Windows**
需要安裝 Microsoft C++ Build Tools，擇一即可：

- 方法一（推薦）：安裝 [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)，勾選「C++ 建置工具」
- 方法二：安裝 [Visual Studio Community](https://visualstudio.microsoft.com/vs/community/)，勾選「使用 C++ 的桌面開發」

安裝完成後**重新開啟終端機**再執行以下步驟。

---

### 1. 開啟 Notebook

直接開啟 `DC_Final_Cython_SubmitVer.ipynb`，**依序執行所有 cell**，Notebook 內部會自動完成編譯。

### 2. 測試

Notebook 最後一個 cell 供直接測試使用：

```python
run_benchmark("你的測資路徑")

# 範例
run_benchmark("test1.txt")
run_benchmark("Cameraman.bmp")
```

執行後會輸出原始大小、壓縮後大小、壓縮倍率、節省百分比、壓縮時間，以及無損驗證結果。

---

## References

1. Ring Buffer Animation — Wikimedia Commons, CC BY-SA 3.0
   https://commons.wikimedia.org/wiki/File:Circular_Buffer_Animation.gif

2. Mahoney, M. (2005). *Adaptive Weighing of Context Models for Lossless Data Compression*. Florida Institute of Technology.

3. Cython Documentation — https://cython.readthedocs.io
