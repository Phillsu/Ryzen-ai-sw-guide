
---

# Ryzen AI Software Setup Guide (Windows)

> 本指南記錄了在 **AMD Ryzen AI 7 350W** 設備上成功安裝與啟用 **Ryzen AI 軟體堆疊（含 NPU 驅動）** 的完整流程，適用於 Windows 11 系統。

---

## 📌 官方參考
- [AMD Ryzen AI 安裝文件](https://ryzenai.docs.amd.com/en/latest/inst.html)

---

## ✅ 前置條件

### 1. 系統需求
- **處理器**：AMD Ryzen AI 7 350W（含 NPU）
- **作業系統**：Windows 11（Build ≥ 22621.3527）
- **Miniforge**（推薦 Python 發行版）

### 2. 安裝 Miniforge
1. 從 [Miniforge 官網](https://conda-forge.org/miniforge/) 下載 **Miniforge3 for Windows (x86_64)**。
2. 安裝時 **務必勾選 `Add Miniforge to my PATH environment variable`**（或手動設定 PATH）。
3. 驗證安裝：
   ```bash
   conda --version
   python --version
   ```

### 3. 手動檢查/設定 PATH（若安裝時未自動加入）
請確認以下路徑已加入 **系統環境變數 → PATH**（非僅使用者變數）：
```
C:\Users\<YourName>\miniforge3
C:\Users\<YourName>\miniforge3\Scripts
C:\Users\<YourName>\miniforge3\condabin
```
> 路徑中的 `<YourName>` 請替換為你的實際使用者名稱。

> 💡 設定路徑：  
> `設定 → 系統 → 關於 → 進階系統設定 → 環境變數 → 系統變數 → Path → 編輯`

---

## 🚀 安裝步驟

### 1. 安裝 NPU 驅動
- 下載並安裝最新 NPU 驅動（版本 ≥ 32.0.203.280）。
- 解壓縮後，**以系統管理員身分執行** `npu_sw_installer.exe`。
- 安裝完成後，可透過 **工作管理員 → 效能 → NPU0** 確認驅動是否載入。

### 2. 安裝 Ryzen AI 軟體套件
- 下載 `ryzenai-lt-1.6.1.exe`（或其他最新版本）。
- 執行安裝程式，按提示操作（建議使用預設選項）：
  - 安裝路徑：`C:\Program Files\RyzenAI\1.6.1`
  - Conda 環境名稱：`ryzen-ai-1.6.1`（可自訂）
- 安裝過程約需 10 分鐘。

---

## ▶️ 啟用與測試

### 啟動 Ryzen AI 環境
打開 **Miniforge Prompt**（或任何終端機），執行：
```bash
conda activate ryzen-ai-1.6.1
```
> 請將 `1.6.1` 替換為你實際安裝的版本號。

### 執行快速測試
```bash
cd %RYZEN_AI_INSTALLATION_PATH%\quicktest
python quicktest.py
```

✅ 成功輸出範例：
```
[Vitis AI EP] No. of Operators :   NPU   398 VITIS_EP_CPU     2
[Vitis AI EP] No. of Subgraphs :   NPU     1 Actually running on NPU     1
Test Passed
```
表示模型已成功在 **NPU 上執行**！

---

## 🛠 常見注意事項
- 若 `conda` 指令無效，請重新啟動終端機或檢查 PATH。
- 確保 Windows 已更新至支援 NPU 的版本（建議 24H2）。
- Ryzen AI 軟體僅支援 **ONNX 模型**，訓練仍建議使用 CPU 或雲端 GPU。

---

## 🛠 常見問題：`UnicodeDecodeError`（非英文系統）

在中文、日文等非英文 Windows 系統上執行 `quicktest.py` 時，可能遇到以下錯誤：

```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xbf in position 158: invalid start byte
```

### 🔍 原因
`pnputil /enum-devices` 命令的輸出使用系統本地 ANSI 編碼（如繁體中文 Windows 使用 `cp950`），但原始腳本強制以 UTF-8 解碼，導致解碼失敗。

### ✅ 解決方法
修改 `quicktest.py` 中的 `get_npu_info()` 函數，使用系統預設編碼進行解碼。

#### 步驟 1：在檔案開頭加入 `import locale`
確保 import 區塊包含：
```python
import locale
import subprocess
```

#### 步驟 2：替換 `get_npu_info()` 函數為以下版本：
```python
def get_npu_info():
    command = r'pnputil /enum-devices /bus PCI /deviceids'
    process = subprocess.Popen(
        command,
        shell=True,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    stdout, stderr = process.communicate()

    # Use system's preferred encoding (e.g., cp950 for Traditional Chinese Windows)
    encoding = locale.getpreferredencoding()
    try:
        output = stdout.decode(encoding)
    except (UnicodeDecodeError, LookupError):
        # Fallback: ignore problematic bytes
        output = stdout.decode('utf-8', errors='ignore')

    npu_type = ''
    if 'PCI\\VEN_1022&DEV_1502&REV_00' in output:
        npu_type = 'PHX/HPT'
    elif any(dev in output for dev in [
        'PCI\\VEN_1022&DEV_17F0&REV_00',
        'PCI\\VEN_1022&DEV_17F0&REV_10',
        'PCI\\VEN_1022&DEV_17F0&REV_11'
    ]):
        npu_type = 'STX'
    elif 'PCI\\VEN_1022&DEV_17F0&REV_20' in output:
        npu_type = 'KRK'
    return npu_type
```

> 💡 此修正會自動適配你的系統語言環境，並在極端情況下安全降級，避免崩潰。

保存後重新執行：
```bash
conda activate ryzen-ai-1.6.1
cd %RYZEN_AI_INSTALLATION_PATH%\quicktest
python quicktest.py
```

應可正常看到 `Test Passed` 輸出。

## 📬 貢獻與回饋
歡迎提交 Issue 或 PR 改進本指南！  
作者：Phillip Su  
設備：AMD Ryzen AI 7 350W + Windows 11

--- 
