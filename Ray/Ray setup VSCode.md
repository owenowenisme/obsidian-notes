## 前言
最近想說來寫看看ray core 寫一些比較硬核的，在用VSCode時，發現VScCode自帶的C++ intellisense非常的慢，最常使用的goto definition就要花20-30秒，嚴重影響開發體驗，在網路上爬了一下文嘗試把換成intellisense換成clangd。

中文社群幾乎沒有這方面的資料，加上Ray是用Bazel在build東西（不是cmake），所以就紀錄一下，或許也可以幫助到要開發Ray的人。
## 環境設置
### Step0: Install dependency and Build
假設已經照[Building Ray from Source](https://docs.ray.io/en/master/ray-contribute/development.html#preparing-to-build-ray-on-linux)有把Bazel等dependencies 裝好，也有把Ray build 好了。
### Step1: Disable intellisense extension
  因為我們現在不用VSCode 的 intellisense，跟clang 同時使用的話會衝突，所以我們把他disable。
  在VSCode setting JSON 加入這行，（或是也可以直接把C/C++ extenstion 解除安裝）
  `"C_Cpp.intelliSenseEngine": "disabled"`
### Step2: install clangd
Ray的docs中使用clang-12 所以我們也用clangd-12，然後順便做個softlink 
```bash
sudo apt install clangd-12
sudo ln -sf /usr/bin/clangd-12 /usr/bin/clangd
```

### Step3: 安裝vscode Clangd extension

![[Screenshot 2025-06-28 at 4.44.48 PM.png]]
### Step4: compile `compile_commands.json`
`compile_commands.json` 是一個可以讓language server 了解codebase info 的檔案。一般來說要自己在BUILD.bazel加入一些config，不過Ray的repo中已經有那些config（詳見：[bazel-compile-commands-extractor](https://github.com/hedronvision/bazel-compile-commands-extractor）），所以我們可以直接跑指令來build) )
所以我們可以直接來build`compile_commands.json`:
```bash
bazel run :refresh_compile_commands
```
你會看到project root 有`compile_commands.json` 出現。

### Step5: 設定一下Clangd 的arguments
在vscode setting中加入下面config:
```json
"clangd.arguments": [
	"--header-insertion=never",
	"--compile-commands-dir=${workspaceFolder}",
	"--query-driver=**"
],
```
### Step6: Reload Window
重新reload 一下VSCode，讓clangd index一下，就可以用了。
## 結果 
可以看到現在hover 會顯示對應的definition & description，也能正常的jump to definition
![[Screenshot 2025-06-28 at 1.17.21 PM.png]]
## 尚未解決
- Ray 的 codebase目前不確定為何會有很多editor的錯誤，不確定是他們沒改或是我有什麼設定忘了加，目前除了眼睛看了有點暈之外其他不影響，所以暫時就先這樣。