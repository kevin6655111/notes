# Git 使用筆記

> Git 儲存庫的運作，是將工作目錄裡的變化，透過更新索引的方式，將資料寫入成 Git 物件。

## 目錄

- [📚 參考資料](#-參考資料)
- [🗂️ 三個區域](#-三個區域)
- [⚡ 常用指令](#-常用指令)
- [🔑 公私鑰](#-公私鑰)
- [🚀 初始化儲存庫](#-初始化儲存庫)
- [➕ Add 暫存區](#-add-暫存區)
- [📝 Commit 提交](#-commit-提交)
- [🔄 Push / Fetch / Pull](#-push--fetch--pull)
- [📦 Stash 暫存版](#-stash-暫存版)
- [🌿 Branch 分支](#-branch-分支)
- [🔀 Merge 合併](#-merge-合併)
- [⚔️ 解決衝突](#-解決衝突)
- [🏷️ Tag 標籤](#-tag-標籤)
- [🔍 查看紀錄](#-查看紀錄)
- [🆚 Diff 比對](#-diff-比對)
- [🗑️ rm 刪除](#-rm-刪除)
- [✏️ mv 更名](#-mv-更名)
- [♻️ Restore 還原檔案](#-restore-還原檔案)
- [⏪ Reset 重置](#-reset-重置)
- [🩹 修改 commit 歷史紀錄](#-修改-commit-歷史紀錄)
- [↩️ Revert 還原](#-revert-還原)
- [🧬 Rebase](#-rebase)
- [🍒 Cherry-pick](#-cherry-pick)
- [🧭 符號參照](#-符號參照)
- [🙈 .gitignore](#-gitignore)
- [👥 多人協作專案](#-多人協作專案)
- [🛠️ 專案流程](#-專案流程)
- [⚙️ 設定](#-設定)

---

## 📚 參考資料

- [Git 入門教學](https://backlog.com/git-tutorial/tw/reference/basic.html)
- [30 天精通 Git 版本控管 - 進階版](https://github.com/doggy8088/Learn-Git-in-30-days/blob/master/zh-tw/README.md)
- [Learn Git Branching!](https://learngitbranching.js.org/?locale=zh_TW)
- [Git 面試題 - Git 教學 | 高見龍 (gitbook.tw)](https://gitbook.tw/interview)
- [Git 版本控制：如何進行多人協作 & 同步分支 - HackMD](https://hackmd.io/@Heidi-Liu/git-workflow)
- [fork 與 pull request 版控流程](https://github.com/doggy8088/Learn-Git-in-30-days/blob/master/zh-tw/28.md)
- [Pro Git 繁體中文版](https://git-scm.com/book/zh-tw/v2)

---

## 🗂️ 三個區域

```
工作目錄 (Working Directory)  --add-->  暫存區 / 索引 (Staging Area / Index)  --commit-->  儲存庫 (Repository)
```

| 區域 | 說明 |
|------|------|
| 工作目錄 | 實際看得到、編輯中的檔案 |
| 暫存區（索引） | `git add` 後的內容，也就是下一次 commit 的內容，存在 `.git/index` |
| 儲存庫 | `git commit` 後的歷史紀錄，存在 `.git/objects` |

---

## ⚡ 常用指令

**提交最新版**

```shell
git add .                   # 將當前目錄下所有異動暫存到暫存區（Staging Area）
git commit -m "message"     # 將暫存區的修改提交到本地版本庫
git push                    # 將本地儲存庫推送到遠端儲存庫
```

**取得最新版**

```shell
git pull                    # 從遠端儲存庫的最新版合併到本地（= fetch + merge）
git pull --rebase           # 從遠端取回後改用 rebase，避免產生多餘的 merge commit
```

**查看狀態**

```shell
git status                  # 目前工作目錄狀態
git log --oneline --graph   # 精簡圖形化歷史
git diff                    # 尚未 add 的變更
```

---

## 🔑 公私鑰

**簡單版**

```shell
ssh-keygen                                  # 建立公私鑰（存在 ~/.ssh/，一路 Enter 即可）
cat ~/.ssh/id_rsa.pub                       # 取公鑰，貼到 GitHub / GitLab 的 SSH Keys
```

**指定演算法版**

```shell
ssh-keygen -t ed25519 -C "your@email.com"   # 建立公私鑰（建議 ed25519，存在 ~/.ssh/）
ssh-keygen -t rsa -b 4096 -C "your@email.com"  # 舊系統不支援 ed25519 時改用 RSA
cat ~/.ssh/id_ed25519.pub                   # 取公鑰（RSA 則是 id_rsa.pub），貼到 GitHub / GitLab 的 SSH Keys
ssh -T git@github.com                       # 測試連線是否成功
rm -rf ~/.ssh/*                             # 刪除現有的公私鑰（小心，會連 known_hosts、config 一起刪）
```

> - `-t` 指定演算法。不加的話由 OpenSSH 版本決定：9.5 以前預設 RSA 3072（檔名 `id_rsa`），9.5 以後預設 ed25519（檔名 `id_ed25519`）。用 `ssh -V` 可查版本。
> - `-C` 只是公鑰的註解，方便在 GitHub 上辨認是哪台機器的鑰匙，不影響安全性。不加則預設為 `使用者@主機名稱`。
> - ed25519 金鑰更短、更快，安全性相當於 RSA 3072 以上，是目前的建議選擇。

---

## 🚀 初始化儲存庫

```shell
git init                               # 初始化儲存庫
git remote add origin <your-repo-url>  # 新增遠端 URL
git remote -v                          # 查看目前遠端 URL
git clone <url>                        # 遠端儲存庫複製到本地
git clone <url> <folder>               # 複製到指定資料夾
git clone --depth 1 <url>              # 只取最新一版（淺層複製，速度快）
```

```shell
git remote remove origin                   # 刪除遠端
git remote set-url origin <your-repo-url>  # 直接更新遠端 URL
git remote rename origin upstream          # 重新命名遠端
git branch -M main                         # 更改目前分支名稱（-M 強制覆蓋同名分支）
```

### 第一次新增本地專案到 Git（GitHub 上是新建的空專案）

```shell
git init                                  # 初始化 .git 本地儲存庫
git remote add origin <your-repo-url>     # 新增 URL
git push -u origin main                   # -u 與 --set-upstream 同理

git push -f -u origin main                # 強制 push（會覆蓋遠端歷史，請小心）
git push -uf origin main                  # 同上
```

> **說明**
> - `-u` / `--set-upstream` 是設定本地分支與遠端分支的追蹤關係。
> - `origin` 是遠端儲存庫的名稱，`main` 是分支的名稱。
> - 設定 upstream 後，後續推送/拉取只需使用 `git push` / `git pull`。

![](https://hackmd.io/_uploads/rJuw49EP3.png)

---

## ➕ Add 暫存區

```shell
git add .              # 將當前目錄下「所有異動」（新增、修改、刪除）新增至暫存區
git add -A             # 整個儲存庫的所有異動（不受目前所在目錄限制）
git add -u             # 只將「已追蹤」檔案的更新或刪除加入暫存區，不含新檔案
git add -p             # 互動式逐段 (hunk) 選擇要加入的變更
git add app/*          # 指定目錄
git add *.txt          # 指定副檔名
```

---

## 📝 Commit 提交

將「索引檔」與「目前最新版」中的資料比對出差異，把差異部分提交變更成一個 commit 物件。

```shell
git commit -m "message"
git commit -am "message"     # 已追蹤檔案自動 add 後提交（新檔案不會被加入）
git commit --allow-empty -m "message"  # 建立空的 commit（如觸發 CI）
```

**Commit 訊息慣例（Conventional Commits）**

```
feat:     新功能
fix:      修 bug
docs:     文件
style:    格式（不影響程式邏輯，如空白、分號）
refactor: 重構（既不是新功能也不是修 bug）
perf:     效能改善
test:     測試
build:    建置系統或外部依賴（npm、gradle、Dockerfile）
ci:       CI 設定（GitHub Actions、GitLab CI）
chore:    雜項（不影響 src 與測試的其他變更）
revert:   還原某次 commit
```

**版更（release）沒有專用類型，慣例上歸在 `chore`：**

```
chore: release v1.2.0
chore(release): 1.2.0
chore: bump version to 1.2.0
```

**格式與其他慣例**

```
<type>(<scope>): <subject>

feat(login): 新增記住我功能            # scope 標示影響範圍，可省略
fix!: 修正 API 回傳格式                # ! 表示破壞性變更 (BREAKING CHANGE)，對應主版號 +1
feat: update <rev.1.2.230510> (#單號)  # 尾端可帶單號方便追蹤
```

> 對應語意化版本 (Semantic Versioning)：`fix` → 修訂號 +1（1.0.**1**）、`feat` → 次版號 +1（1.**1**.0）、`!` 或 `BREAKING CHANGE` → 主版號 +1（**2**.0.0）。

---

## 🔄 Push / Fetch / Pull

```shell
git push origin main        # 將本地 main 推到名為 origin 的遠端
git push -u origin <branch> # 第一次推送新分支並設定追蹤
git push origin <tagName>   # 推送單一標籤（--tags 會推所有本地標籤，容易誤推）
git push origin --delete <branch>  # 刪除遠端分支
git push -d origin <branch>        # 同上

git fetch                   # 只下載遠端更新到本地追蹤分支 (origin/xxx)，不合併
git fetch --prune           # 同時清除遠端已刪除的分支參照
git pull                    # = git fetch + git merge
git pull --rebase           # = git fetch + git rebase
```

> `fetch` 與 `pull` 的差別：`fetch` 只是把遠端內容抓下來放在 `origin/<branch>`，工作目錄不會變動；`pull` 會直接合併進目前分支。

---

## 📦 Stash 暫存版

stash 的參照存在 `.git/refs/stash`，每一筆 stash 都是一個 commit 物件。

```shell
git stash list              # 列出暫存

git stash                   # 將所有「已列入追蹤」(tracked) 的異動建立暫存版
git stash -u                # 連「未追蹤」(untracked) 的檔案也一起建立暫存版
git stash push -m <message> # 加上訊息備註（建議用法）
git stash save <message>    # 舊寫法，已被 stash push 取代（仍可用）

git stash pop               # 取回最新暫存並套用，同時將該 stash 刪除
git stash apply             # 取回最新暫存並套用，stash 不會刪除
git stash apply "stash@{1}" # 取回指定的暫存，stash 不會刪除
git stash show -p "stash@{0}"  # 查看某筆 stash 的差異內容

git stash drop "stash@{1}"  # 將特定暫存版刪除
git stash clear             # 清除所有暫存
```

```shell
git cat-file -p stash       # 查看 stash 物件內容
```

透過 `cat-file` 可看到 stash commit 有多個 parent，是由以下幾個地方合併而成：

- 『工作目錄的 HEAD 版本』（分別建立以下兩個暫時的分支，產生 commit 物件）
  - 『索引中的內容』（已 add 的變更）
  - 『工作目錄中的內容』（未 add 的變更；若使用 `-u` 會再多一個 parent 存放未追蹤檔案）

建立 stash 後，會將整個「工作目錄」重置為 HEAD 版本，把這些變更與新增的檔案都還原，多的檔案也會被移除。

---

## 🌿 Branch 分支

分支都在 `.git/refs/heads` 下。

```shell
git branch                          # 查看本地分支
git branch -a                       # 顯示所有「本地分支」與「遠端追蹤分支」
git branch -r                       # 只顯示遠端追蹤分支
git branch -vv                      # 顯示每個分支追蹤的遠端分支
git ls-remote                       # 顯示遠端儲存庫的參照名稱、遠端分支與遠端標籤

git checkout -b feature/<newBranch> # 建新分支並切換到新分支
git switch -c feature/<newBranch>   # 同上（Git 2.23+ 新指令，語意更清楚）
git checkout <branch>               # 切換分支
git switch <branch>                 # 同上
git switch -                        # 切回上一個分支

git branch -d <branch>              # 刪除分支（已合併才允許刪除）
git branch -D <branch>              # 強制刪除分支（未合併也刪）
git branch -m <old> <new>           # 分支更名
```

### 救回誤刪的分支

1. 利用 `git reflog` 找出該分支最後一個版本的 object id（SHA1 格式的物件絕對名稱）
2. 執行以下指令

```shell
git branch feature/<branch> <SHA1>
```

---

## 🔀 Merge 合併

```shell
git merge feature/<branch>          # 合併，若可以會使用 fast-forward（不產生 merge commit）
git merge feature/<branch> --no-ff  # 一律產生 merge commit，保留分支歷史
git merge --squash feature/<branch> # 將對方分支所有 commit 壓成一次變更放入暫存區，再自行 commit
git merge --abort                   # 合併發生衝突時，放棄合併回到合併前狀態
```

> **fast-forward vs no-ff**
> - fast-forward：目前分支只是單純落後，直接把指標往前移，歷史是一直線。
> - `--no-ff`：強制建立 merge commit，在歷史圖上可以清楚看到「這個功能分支在哪裡合進來」。團隊協作通常建議使用 `--no-ff`。

---

## ⚔️ 解決衝突

1. 找出衝突的檔案

   ```shell
   git status
   git ls-files -u            # 列出索引中處於未合併 (unmerged) 狀態的檔案
   git diff --name-only --diff-filter=U
   ```

2. 比對衝突的檔案

   ```shell
   git diff <filepath>
   ```

   衝突標記格式：

   ```
   <<<<<<< HEAD
   目前分支的內容
   =======
   要合併進來的分支的內容
   >>>>>>> feature/xxx
   ```

3. 修改衝突地方（或直接選擇某一邊）

   ```shell
   git checkout --ours <file>     # 保留目前分支的版本
   git checkout --theirs <file>   # 保留對方分支的版本
   ```

4. 重新 add 與 commit 再 push

   ```shell
   git add <file>
   git commit                     # merge 衝突時不用加 -m，會自動帶入合併訊息
   ```

---

## 🏷️ Tag 標籤

用途是用來標記某一個「版本」。分為輕量標籤（只是指標）與標示標籤（annotated，帶作者、時間、訊息，是完整的 tag 物件）。

```shell
git tag                           # 列出所有標籤
git tag -l "v1.*"                 # 篩選標籤
git tag <tagName>                 # 建立輕量標籤
git tag -a <tagName> -m "message" # 建立標示標籤（建議發版時使用）
git tag -a <tagName> <commit_id>  # 對過去的 commit 補標籤

git show <tagName>                # 查看標籤內容與所指向的 commit
git cat-file -t <tagName>         # 查看物件類型（輕量標籤是 commit，標示標籤是 tag）
git cat-file -p <tagName>         # 查看標籤訊息

git tag -d <tagName>              # 刪除本地標籤
git push origin <tagName>         # 推送單一標籤（建議）
git push origin <branch> <tagName>  # 分支與標籤一起推
git push --follow-tags            # 只推送「被推送 commit 所指向」的標示標籤，不會帶到無關的舊標籤
git push --tags                   # 推送本地所有標籤（含別人的、測試用的，容易誤推，不建議）
git push origin --delete <tagName>  # 刪除遠端標籤
```

> 標籤預設不會隨 `git push` 一起推上去，要另外指定標籤名稱。避免用 `--tags`，它會把本地所有標籤一次推上去。

---

## 🔍 查看紀錄

### log / status / reflog

```shell
git log                      # 查看版本的歷史紀錄
git log -<n>                 # 查看前 n 筆歷史紀錄
git log --oneline            # 精簡的 commit 歷史紀錄（HashID 只顯示部分字元）
git log --oneline --graph --all --decorate  # 圖形化顯示所有分支
git log --pretty=oneline     # 精簡顯示，但 HashID 顯示完整
git log --pretty=oneline --abbrev-commit    # 同上，但 HashID 只顯示部分字元
git log -p                   # 顯示每次 commit 的 diff
git log --stat               # 顯示每次 commit 異動的檔案與行數
git log --author="kevin"     # 篩選作者
git log --since="2 weeks ago"  # 篩選時間
git log -- <file>            # 只看某個檔案的歷史
git log -S "keyword"         # 找出新增/刪除了某關鍵字的 commit

git status                   # 取得工作目錄 (working tree) 下狀態（修改、暫存、追蹤）
git status -s                # 簡潔顯示狀態

git reflog                   # 查看 HEAD 移動的紀錄（reset、checkout 等都會留下紀錄）
git log -g                   # log 紀錄加上 reflog 更詳細資訊
```

> **`git status -s` 符號說明**（兩欄：左邊是暫存區狀態，右邊是工作目錄狀態）
> - `??`：未追蹤的檔案
> - `A `：新檔案已加入暫存區
> - `M `：已修改且已加入暫存區
> - ` M`：已修改但尚未加入暫存區
> - `D `：已刪除且已加入暫存區

### show / blame

```shell
git show <commit_id>         # 查看某次 commit 的完整內容與 diff
git show <commit_id>:<file>  # 查看某版本的某個檔案內容
git blame <file>             # 逐行顯示是誰、在哪次 commit 修改的
```

### hash / cat-file

```shell
git hash-object <fileName>   # 計算檔案的 Hash 碼（不寫入 .git）
git cat-file -p main         # 查看 main 最新 commit 物件的內容（含 Tree 物件 Hash ID）
git cat-file -p <HashID>     # 查看某個 HashID 的物件內容（commit / tree / blob）
git cat-file -t <HashID>     # 查看物件類型

git ls-files                 # 列出所有目前已經儲存在「索引檔」中的檔案路徑
```

### 列出每個人的 commit 次數

```shell
git shortlog -sne
```

### 刪除未被追蹤的檔案

```shell
git clean -n     # 預覽要刪除的檔案（dry run）
git clean -f     # 刪除未追蹤的檔案
git clean -fd    # 連未追蹤的目錄一起刪
git clean -fdx   # 連 .gitignore 忽略的檔案也刪（如 node_modules、build 產物）
```

> `git clean` 刪掉的檔案無法從 Git 救回，執行前務必先 `-n` 預覽。

---

## 🆚 Diff 比對

```shell
git diff                # 「工作目錄」 vs 「暫存區」：尚未 add 的變更
git diff <filename>     # 同上，只看單一檔案

git diff --cached       # 「暫存區」 vs 「HEAD」：已 add 但尚未 commit 的變更
git diff --staged       # 同上，--staged 是 --cached 的別名

git diff HEAD           # 「工作目錄」 vs 「HEAD」：所有尚未 commit 的變更（含已 add 與未 add）

git diff HEAD^ HEAD     # 「上一個提交(父)」 vs 「當前提交」的差異
git diff <commit1> <commit2>   # 任意兩個 commit 的差異
git diff main..feature/xxx   # 兩個分支的差異
git diff --stat         # 只顯示異動檔案與行數統計
git diff --name-only    # 只列出異動的檔案名稱
```

---

## 🗑️ rm 刪除

```shell
git rm <fileName>            # 從索引和工作目錄一併刪除
git rm --cached <fileName>   # 只從索引移除（不再追蹤），保留工作目錄的實體檔案
git rm -r --cached <folder>  # 取消追蹤整個資料夾（常用於誤 commit 的 node_modules 等）
git rm '*.txt'
git rm 'app/*.html'
```

> **常見情境**：檔案已經被 commit 進去，事後才加進 `.gitignore`，此時 `.gitignore` 不會生效。
> 需要先 `git rm --cached <file>` 取消追蹤，再 commit，之後才會被忽略。

---

## ✏️ mv 更名

```shell
git mv <oldName> <newName>  # 更改檔案或目錄名稱（= mv + git add，Git 會偵測為 rename）
```

---

## ♻️ Restore 還原檔案

Git 2.23+ 新增的指令，取代 `git checkout -- <file>` 與 `git reset HEAD <file>` 這兩種容易混淆的用法。

```shell
git restore <file>            # 丟棄工作目錄的修改，還原成暫存區（或 HEAD）的版本
git restore .                 # 丟棄所有未 add 的修改
git restore --staged <file>   # 把檔案從暫存區移出（unstage），保留工作目錄的修改
git restore --source=<commit> <file>  # 從指定 commit 還原某個檔案

# 舊寫法（仍可用）
git checkout -- <file>        # 同 git restore <file>
git checkout main <file>      # 從 main 分支還原某個檔案
git reset HEAD <file>         # 同 git restore --staged <file>
```

> 用 `git restore` 或 `git checkout <file>` 只還原單一檔案，可以避免使用 `git reset --hard` 一次把所有檔案都還原。

---

## ⏪ Reset 重置

`git reset` 會把 HEAD（與目前分支）移到指定的 commit，三種模式差在「暫存區」與「工作目錄」要不要跟著動：

| 模式 | HEAD | 暫存區 | 工作目錄 | 說明 |
|------|------|--------|----------|------|
| `--soft` | 移動 | 保留 | 保留 | 取消 commit，變更留在暫存區 |
| `--mixed`（預設） | 移動 | 重置 | 保留 | 取消 commit 與 add，變更留在工作目錄 |
| `--hard` | 移動 | 重置 | 重置 | 全部丟棄，變更消失 |

**取消最近一次 commit，變更保留在暫存區**

```shell
git reset --soft HEAD^
```

**取消最近一次 commit，變更保留在工作目錄（但已 unstage）**

```shell
git reset HEAD^
git reset --mixed HEAD^        # 同上
```

**丟棄所有未 commit 的變更，回到 HEAD 的乾淨狀態**（無法恢復）

```shell
git reset --hard
```

**回到上一個版本（當前版本的父版本），並丟棄所有變更**

```shell
git reset --hard HEAD^
git reset --hard HEAD~         # 同上
git reset --hard HEAD~3        # 回到 3 個版本前
```

**如果已經 merge 合併，可以使用 ORIG_HEAD 回到合併前的狀態**

```shell
git reset --hard ORIG_HEAD     # ORIG_HEAD 指向上一次執行 reset / merge / rebase 等
                               # 「危險操作」之前的 HEAD 位置
```

**用 reflog 回到任意一個 HEAD 曾經到過的位置**

```shell
git reflog                     # 找出目標位置，例如 HEAD@{1}
git reset --hard HEAD@{1}
# reset 本身也會留下 reflog，所以做錯了再 reset 回去即可
```

> **reset 過的 commit 還救得回來嗎？**
> 可以。commit 物件不會馬上被刪除，透過 `git reflog` 找到 SHA1 後 `git reset --hard <SHA1>` 或 `git branch <name> <SHA1>` 就能救回。但 `--hard` 丟棄的「尚未 commit」的變更是真的救不回來。

---

## 🩹 修改 commit 歷史紀錄

修改訊息文字，或該版本忘了 add 某個檔案就 commit 了，想事後補救這次變更：

```shell
git add <fileName>
git commit --amend             # 把暫存區內容併入最新的 commit，並開啟編輯器修改訊息
git commit --amend -m "新訊息"  # 直接指定新訊息
git commit --amend --no-edit   # 只補檔案，訊息維持不變
```

> - `--amend` 實際上是建立一個新的 commit 取代原本的，Hash 會改變。
> - 如果該 commit 已經 push 出去，amend 後需要 `git push -f`，多人協作時務必先確認。

---

## ↩️ Revert 還原

[講義](https://github.com/doggy8088/Learn-Git-in-30-days/blob/master/zh-tw/20.md)

- revert 是建立一個「反向」的新 commit，把指定 commit 的變更抵銷掉，原本的歷史不會被改寫
- 適合用在已經 push 出去、不能改寫歷史的情況（reset 則適合尚未 push 的本地操作）
- 執行 revert 前，確保工作目錄乾淨，如有改到一半的檔案，可透過 stash 建立暫存版

```shell
git revert <commit_id>        # 還原指定 commit 的變更，並建立新的 commit
git revert <commit_id> -n     # 套用還原變更到暫存區，但不執行 commit
git revert HEAD               # 還原最新一次 commit
git revert -m 1 <merge_commit>  # 還原 merge commit（-m 1 表示保留第一個 parent，即主線）
git revert --abort            # 發生衝突時放棄 revert
```

---

## 🧬 Rebase

[講義](https://github.com/doggy8088/Learn-Git-in-30-days/blob/master/zh-tw/22.md)

rebase 是把目前分支的 commit「重新接」到另一個基準點上，讓歷史保持一直線。

```shell
git checkout feature/xxx
git rebase main               # 把 feature/xxx 的 commit 重新接在 main 最新版之後
git rebase --continue         # 解決衝突後繼續
git rebase --skip             # 跳過目前這個 commit
git rebase --abort            # 放棄 rebase，回到 rebase 前的狀態
```

**互動式 rebase：整理尚未 push 的 commit**

```shell
git rebase -i HEAD~3          # 整理最近 3 個 commit
```

編輯器中每一行可用的指令：

```
pick    保留這個 commit
reword  保留但修改訊息
edit    停下來修改內容
squash  併入前一個 commit，並合併訊息
fixup   併入前一個 commit，丟棄這個 commit 的訊息
drop    刪除這個 commit
```

> **merge vs rebase**
> - merge：保留完整的分叉歷史，會產生 merge commit。
> - rebase：歷史變成一直線，比較乾淨，但會改寫 commit（Hash 改變）。
> - **黃金法則**：不要對已經 push 出去、別人可能已經拿到的 commit 做 rebase。

---

## 🍒 Cherry-pick

[講義](https://github.com/doggy8088/Learn-Git-in-30-days/blob/master/zh-tw/21.md)

從其它分支「挑選」任意一個或多個版本，套用在目前分支的最新版上。

```shell
git log feature/<test> --oneline -5   # 列出其它分支的 log 訊息
git cherry-pick <commit_id>           # 挑選其它分支的 commit 到當前分支
git cherry-pick <id1> <id2>           # 一次挑多個
git cherry-pick <id1>..<id3>          # 挑選連續範圍（不含 id1，含 id3）
git cherry-pick <commit_id> -n        # 套用變更到暫存區，但不 commit
git cherry-pick <commit_id> -x        # 在訊息中加上「(cherry picked from commit xxx)」來源資訊
git cherry-pick --abort               # 發生衝突時放棄
```

---

## 🧭 符號參照

```shell
cat .git/HEAD             # 指向目前所在分支（例如 ref: refs/heads/main）
cat .git/ORIG_HEAD        # 指向上次執行 reset / merge / rebase 等危險操作之前的 HEAD
cat .git/FETCH_HEAD       # 上次 git fetch 抓回來的遠端各分支最新版

git show-ref              # 顯示在 .git/refs 底下的所有參照
git update-ref refs/<newRef> <object_id>  # 建立新的參照名稱對應 HashID
```

相對名稱特殊符號：`^` 與 `~`

| 寫法 | 意義 |
|------|------|
| `HEAD^` / `HEAD~` / `HEAD~1` | 第一個父提交（上一版） |
| `HEAD^^` / `HEAD~2` | 往上兩代 |
| `HEAD^2` | 第二個父提交（只有 merge commit 才有，表示被合併進來的那條線） |
| `HEAD~2^2` | 往上兩代後，取其第二個父提交 |

<p align="center">
  <img src="https://hackmd.io/_uploads/BkgawEuv3.png" alt="相對名稱表示法 ^ 與 ~ 的差異">
  <br>
  相對名稱表示法 ^ 與 ~ 的差異
</p>

```shell
git rev-parse main        # 把任意「參考名稱」或「相對名稱」解析出「絕對名稱」(SHA1)
git rev-parse HEAD
git rev-parse ORIG_HEAD
git rev-parse HEAD^
git rev-parse HEAD~5
git rev-parse --short HEAD   # 顯示短 Hash
```

---

## 🙈 .gitignore

放在專案根目錄，列出不要被追蹤的檔案。

```gitignore
# 註解
node_modules/        # 忽略整個資料夾
*.log                # 忽略所有 .log
!important.log       # 例外：不忽略這個檔案
/build               # 只忽略根目錄的 build，不影響 src/build
**/temp              # 任意層級的 temp 資料夾
.env
.vscode/
```

```shell
git check-ignore -v <file>   # 檢查某個檔案是被哪一條規則忽略的
git status --ignored         # 顯示被忽略的檔案
```

> 已經被追蹤的檔案，加進 `.gitignore` 不會生效，要先 `git rm --cached <file>`。
> 各語言常用範本：<https://github.com/github/gitignore>

---

## 👥 多人協作專案

### 分支策略

- **main / master**：穩定、可部署的版本
- **development**：開發主線，功能分支從這裡分出、合併回來，穩定後再合併到 main
- **feature/xxx**：增加新功能，從 development 分出，完成後合併回 development
- **hotfix/xxx**：「正式機」的系統出現嚴重錯誤，但在「開發分支」裡又包含尚未完成的功能，這時可以從 main 分支緊急建立一個「修正分支」，修完後合併回 main 並補標籤
- **tag 標示標籤**：如果 main 趨於穩定版本，可以建立一個「標示標籤」(annotated tag)。切換到 main 分支後輸入 `git tag -a 1.0.0-beta1 -m "V1.0.0-beta1 created"` 即可建立一個名為 `1.0.0-beta1` 的標示標籤，並透過 `-m` 賦予標籤一個說明訊息。

### 同步遠端分支

```shell
git fetch --prune                     # 抓取遠端並清掉已刪除的遠端分支
git checkout -b <branch> origin/<branch>  # 從遠端分支建立本地分支並追蹤
git switch <branch>                   # 若遠端有同名分支，直接 switch 也會自動建立追蹤
git branch -u origin/<branch>         # 為現有本地分支設定追蹤的遠端分支
```

---

## 🛠️ 專案流程

```shell
git pull                                          # 先更新到最新
git switch -c feature/<newbranch>                 # 開功能分支
git add .
git commit -m "feat: update <rev.1.2.230510> (#單號)"
git push -u origin feature/<newbranch>            # 推上遠端並設定追蹤
git switch development
git pull                                          # 合併前再確認 development 是最新的
git merge feature/<newbranch> --no-ff             # 合併，保留分支歷史
git tag -a <rev.1.2.230510> -m "release <rev.1.2.230510>"
git push origin development <rev.1.2.230510>      # 推送 development 與指定標籤
git branch -d feature/<newbranch>                 # 合併完可刪除本地功能分支
git push origin --delete feature/<newbranch>      # 刪除遠端功能分支（視需要）
```

---

## ⚙️ 設定

設定檔位置與優先順序（後者覆蓋前者）：

| 層級 | 參數 | 位置 |
|------|------|------|
| 系統 | `--system` | Windows：`<Git 安裝目錄>\etc\gitconfig`；Linux：`/etc/gitconfig` |
| 使用者 | `--global` | Windows：`%USERPROFILE%\.gitconfig`；Linux：`~/.gitconfig` |
| 專案 | `--local`（預設） | `<專案>/.git/config` |

```shell
git config --list                         # 列出所有設定
git config --list --show-origin           # 列出設定並顯示來自哪個檔案

git config --global user.name "kevin"     # 設定使用者名稱
git config --global user.email "your@email.com"
git config --global core.editor "code --wait"   # 預設編輯器改為 VS Code
git config --global init.defaultBranch main     # git init 預設分支名稱
git config --global pull.rebase false     # git pull 預設行為（false = merge，true = rebase）

git config --global alias.st status       # 建立別名：git st
git config --global alias.lg "log --oneline --graph --all --decorate"
```

### 斷行字元 CRLF (Windows) 與 LF (Linux / macOS)

1. 查詢設定

   ```shell
   git config core.autocrlf
   ```

2. 依作業系統設定

   ```shell
   git config --global core.autocrlf true    # Windows：commit 時轉 LF，checkout 時轉 CRLF
   git config --global core.autocrlf input   # Linux / macOS / WSL：commit 時轉 LF，checkout 不轉換
   ```

> - `true` 只適合 Windows；在 Linux / WSL 上設 `true` 會讓 checkout 出來的檔案變成 CRLF，反而造成問題。
> - 原筆記用 `--system` 設定，若無權限或想只影響自己，用 `--global` 即可。
> - 更好的做法是在專案根目錄放 `.gitattributes` 統一規範，不受每個人的設定影響：
>
>   ```gitattributes
>   * text=auto eol=lf
>   *.bat text eol=crlf
>   ```
