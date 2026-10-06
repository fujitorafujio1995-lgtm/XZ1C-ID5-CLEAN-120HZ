# KernelSU-Next 開發與部署指南

本核心專案已整合 KernelSU-Next，並採用 Git Submodule 結構進行維護，確保換機開發或環境重灌時能快速恢復。

## 1. 新環境快速恢復流程 (New Setup)

當您在新的伺服器或目錄執行完 `repo sync` 後，請執行以下步驟：

### 步驟 A：初始化子模組
這會將託管在 `justinlin099/KernelSU-Next` 的自定義修改版本下載到本地。
```bash
cd kernel/sony/msm8998
git submodule update --init --recursive
```

### 步驟 B：運行配置腳本
這會自動修改核心的 `drivers/Kconfig` 等檔案，讓編譯系統識別 KernelSU。
```bash
# 在核心目錄執行
curl -LSs "https://raw.githubusercontent.com/KernelSU-Next/KernelSU-Next/next/kernel/setup.sh" | bash -s legacy
```

## 2. 日常開發流程 (Development)

### 修改 KernelSU 程式碼
如果您修改了 `KernelSU-Next/` 資料夾內的程式碼，請進入該目錄提交：
```bash
cd KernelSU-Next
git add .
git commit -m "ksu: 描述您的修改"
git push origin legacy
```

### 修改核心程式碼 (SELinux/VFS 等)
如果您修改了核心專案本身的檔案（如 `security/selinux/hooks.c`），請在核心根目錄提交：
```bash
cd /path/to/kernel/sony/msm8998
git add .
git commit -m "kernel: 描述您的修改"
git push justinlin099 HEAD:lineage-22.2
```

## 3. 跟進官方更新 (Update from Upstream)

當 KernelSU-Next 官方有更新且您想合併時：
```bash
cd KernelSU-Next
git remote add upstream https://github.com/KernelSU-Next/KernelSU-Next.git
git fetch upstream
git merge upstream/legacy
# 處理衝突後推送
git push origin legacy
```

## 4. 目錄結構說明
*   `KernelSU-Next/`: Git 子模組，存放 KernelSU 的原始碼（來自您的 Fork）。
*   `drivers/kernelsu/`: 指向 `../KernelSU-Next/kernel` 的符號連結。
*   `KERNEL_SU_GUIDE.md`: 本指南文件。
