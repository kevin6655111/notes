# GitLab CI/CD

> 以 git tag 驅動部署的 pipeline 怎麼組成，以及 `.gitlab-ci.yml` 裡每個關鍵字實際在做什麼。適用於一份原始碼要部署到多台伺服器、且測試環境與正式環境權限需要分離的情境。

## 目錄

- [🧱 組成](#parts)
- [🏃 Runner 與 executor](#runner)
- [🎯 派工：tags](#tags)
- [🚦 觸發條件：rules](#rules)
- [♻️ 共用範本：extends](#extends)
- [🔑 變數與機密](#variables)
- [🌐 environment](#environment)
- [🚧 並行控制：resource_group](#resource-group)
- [🛡️ protected 三件套](#protected)
- [📋 常用預定義變數](#predefined)
- [🔁 tag 驅動的發版流程](#release)
- [🚨 疑難排解](#trouble)
- [📚 參考資料](#ref)

---

<a id="parts"></a>

## 🧱 組成

| 名詞 | 說明 |
|------|------|
| pipeline | 一次完整的流程，由某個事件觸發（push、打 tag、排程、手動） |
| stage | 階段，同一 stage 的 job 平行跑，跑完才進下一個 stage |
| job | 最小的執行單位，內容是一段 shell |
| runner | 真正執行 job 的程式，裝在要幹活的機器上 |

設定檔預設是專案根目錄的 `.gitlab-ci.yml`。

> 專案的 CI/CD 功能被關掉、或設定檔路徑被改過時，push 進來**不會建立 pipeline，也不會有任何錯誤訊息**。遇到「完全沒反應」先查這兩項。

---

<a id="runner"></a>

## 🏃 Runner 與 executor

Runner 裝在部署目標機器上，主動向 GitLab 詢問有沒有活可接。常見的 executor：

| executor | 說明 |
|----------|------|
| shell | 直接在該機器上跑指令，能操作本機的 docker 與檔案系統 |
| docker | 每個 job 開一個乾淨容器，環境隔離但拿不到宿主機狀態 |

部署類的 job 通常用 **shell executor**，因為要在該機器上 `docker compose up`。

註冊時要用 `sudo`，否則會變成 user-mode，關掉終端機就失效。裝完還要讓它能用 docker：

```shell
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
```

---

<a id="tags"></a>

## 🎯 派工：tags

job 的 `tags` 與 Runner 的標籤比對，決定這個 job 在哪台機器上跑。**與 git tag 無關**，是同名不同物。

```yaml
deploy-test:
  tags:
    - test-server      # 只有帶 test-server 標籤的 Runner 會接
```

註冊 Runner 時的「Run untagged jobs」不要勾，否則它會去接所有沒指定 tag 的 job。

---

<a id="rules"></a>

## 🚦 觸發條件：rules

`rules` 由上而下比對，**第一個符合的生效**，都不符合就不建立這個 job。

```yaml
deploy-test:
  rules:
    - if: $CI_COMMIT_TAG =~ /^rev\./        # 打 rev. 開頭的 tag，自動跑
    - if: $CI_PIPELINE_SOURCE == "web"      # 在網頁上手動觸發
      when: manual
```

```yaml
deploy-prod:
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\./
      when: manual                          # 建立 job 但等人按按鈕
    - if: $CI_PIPELINE_SOURCE == "web"
      when: manual
  allow_failure: false                      # 讓未按的手動 job 擋住 pipeline
```

幾個容易踩的點：

- **正規表示式裡的 `\.` 是跳脫過的字面點。** `/^v\./` 只認 `v.2.0.260827`，`v2.0.260827` 不會匹配。要兩種都通寫 `/^v\.?/`。
- **tag 不記錄自己是在哪個分支打的**，tag pipeline 裡沒有 `$CI_COMMIT_BRANCH`。「`v.` 來自 main」純粹是命名約定，在別的分支打一樣會觸發。真要擋得在 `before_script` 自己加 `git merge-base --is-ancestor` 檢查。
- **`when: manual` 的 job 預設 `allow_failure: true`**，沒按不會擋 pipeline，狀態直接算成功。要它顯示 blocked 得明寫 `allow_failure: false`。

改成定時觸發：CI/CD → Schedules 新增 cron，再把 rules 換成 `- if: $CI_PIPELINE_SOURCE == "schedule"`。

> 排程等於沒人看著的情況下自動跑資料庫 migration。含結構變更的版本建議維持手動。

---

<a id="extends"></a>

## ♻️ 共用範本：extends

以 `.` 開頭的 job 是**隱藏 job**，不會被執行，只當範本用。

```yaml
.deploy:
  stage: deploy
  resource_group: $CI_ENVIRONMENT_NAME
  variables:
    SERVICES: web              # 要更新的服務，多個用空白分隔
  before_script:
    - test -n "$DEPLOY_ENV" || { echo "❌ DEPLOY_ENV not set"; exit 1; }
    - cp "$DEPLOY_ENV" .env
    - chmod 600 .env
    - test -f "config/${SITE}.prod.yaml" || { echo "❌ config for ${SITE} not found"; exit 1; }
    - cp "config/${SITE}.prod.yaml" config/config.prod.yaml
    - echo "📦 SITE=${SITE} VERSION=${CI_COMMIT_TAG:-Untagged} commit=${CI_COMMIT_SHORT_SHA}"
  script:
    - docker compose up -d --build $SERVICES

deploy-test:
  extends: .deploy
  tags: [test-server]
  environment:
    name: test
  variables:
    SITE: test
    DEPLOY_ENV: $ENV_TEST
```

每個部署目標只覆寫三件事：**跑在哪台**（`tags`）、**用哪份設定**（`SITE`）、**用哪份機密**（`DEPLOY_ENV`）。共用的流程全部收在範本裡，新增一個部署目標就是多貼這七行。

兩個值得抄的習慣：

- **`before_script` 先做前置檢查再繼續。** 變數沒帶到、設定檔不存在這類問題會停在看得懂的訊息上，而不是部署到一半才炸。
- **開頭印一行摘要。** 版本、commit、用了哪份設定都在 log 第一屏，出事時不用翻 GitLab UI 對照。

---

<a id="variables"></a>

## 🔑 變數與機密

在 **Settings → CI/CD → Variables** 建立，兩種型別：

| Type | 變數的值是 | 適用 |
|------|------------|------|
| Variable | 字串本身 | 一般設定 |
| File | **臨時檔案的路徑**，內容被寫進該檔 | 憑證、`.env`、金鑰 |

File 型別是關鍵：變數拿到的是路徑，所以用法是 `cp "$DEPLOY_ENV" .env` 而不是 `echo`。多行內容不會被 shell 跳脫問題弄壞。

三個勾選項：

| 選項 | 作用 |
|------|------|
| Protect variable | 只有 protected 的 branch / tag 觸發時才拿得到值 |
| Mask variable | 在 log 中遮蔽。有字元集與長度限制，多行內容通常勾不了 |
| Expand variable reference | 值裡的 `$xxx` 是否展開，機密建議不勾 |

> **沒勾 Protect 的變數，所有 Developer 都讀得到。** 在自己的分支加一行把變數送去外部再跑一次 pipeline 就行，不需要任何人核准。所以**測試環境的憑證不能與正式環境共用** —— 第三方 API key、網芳帳密、SMTP 這類最容易不小心共用。共用的話那就不是測試機外洩，是正式機外洩。

---

<a id="environment"></a>

## 🌐 environment

```yaml
deploy-test:
  environment:
    name: test
```

宣告這個 job 部署到哪個環境。GitLab 會因此：

- 在 **Deployments → Environments** 追蹤每個環境目前跑的是哪一版
- 提供 `$CI_ENVIRONMENT_NAME` 變數
- 支援 **Prevent outdated deployment jobs**（Settings → CI/CD → Deployments）

最後這項建議打開：比目前已部署版本更舊的 job 會被自動跳過，避免舊的蓋掉新的。

---

<a id="resource-group"></a>

## 🚧 並行控制：resource_group

```yaml
.deploy:
  resource_group: $CI_ENVIRONMENT_NAME
```

同一個 `resource_group` 的 job **跨 pipeline 一次只跑一個**，其餘排隊，狀態顯示 `Waiting for resource`。

為什麼需要：Runner 的 `concurrent > 1` 時，兩個 job 會拿到不同的 workspace，但 `docker-compose.yml` 裡的 `container_name` 是寫死的。兩邊同時 `up --build` 會互相把對方的容器砍掉重建。兩個人前後腳打 tag 就會踩到。

> 預設排隊順序是 unordered，不保證先進先出。配合上面的 Prevent outdated deployment jobs 就不需要嚴格順序，舊的會被跳過。

> Runner 設定裡的 `concurrent`（同時執行幾個 job）與 `request_concurrency`（同時向 GitLab 詢問有沒有活的連線數）是兩件事，調後者不會讓 job 並行。

---

<a id="protected"></a>

## 🛡️ protected 三件套

要讓正式環境只有 Maintainer 能部署，三個環節缺一不可：

| 環節 | 設定位置 | 作用 |
|------|----------|------|
| Runner 勾 **Protected** | Runner 註冊 / 編輯 | 非 protected ref 的 job 配不到 Runner |
| tag 設為 **protected tag** | Settings → Repository → Protected tags | 只有指定角色建得出來 |
| 變數勾 **Protect variable** | Settings → CI/CD → Variables | 非 protected ref 拿不到值 |

「protected」在 GitLab 裡是 **ref 的屬性，branch 與 tag 分開設定**。只設了 protected branch 不會讓 tag 也受保護。

漏設時的症狀不會告訴你原因：

| 漏了什麼 | 症狀 |
|----------|------|
| tag 沒設 protected，但 Runner 勾了 Protected | job 永遠卡 `Pending`，`no runners matched` |
| tag 沒設 protected，但變數勾了 Protect | job 起跑後停在 `DEPLOY_ENV not set` |

建立 wildcard 的方式：點 **Select tag or create wildcard** 下拉 → 直接輸入 `v.*` → 選出現的 `Create wildcard v.*` → 選 Allowed to create → Protect。既有 tag 清單裡不會有，要自己打。

> wildcard 的點要跟 rules 的 regex 對齊。`v.*` 只保護 `v.` 開頭的，不保護 `v2.0.260827`。
>
> 設之前先確認自己是 Maintainer 以上。設完只有該層級能建那個前綴的 tag，**包含你自己**。

---

<a id="predefined"></a>

## 📋 常用預定義變數

| 變數 | 內容 |
|------|------|
| `CI_COMMIT_TAG` | tag 名稱。非 tag pipeline 時為空 |
| `CI_COMMIT_BRANCH` | 分支名稱。**tag pipeline 時為空** |
| `CI_COMMIT_SHORT_SHA` | 短 commit SHA |
| `CI_PIPELINE_SOURCE` | 觸發來源：`push` / `web` / `schedule` / `api` / `trigger` |
| `CI_ENVIRONMENT_NAME` | `environment.name` 的值 |
| `CI_PROJECT_DIR` | workspace 路徑 |

另外兩個控制取碼行為的：

```yaml
variables:
  GIT_STRATEGY: fetch       # 沿用既有 workspace 增量抓取，比 clone 快
  GIT_CLEAN_FLAGS: -ffd     # 不加 -x，保留 gitignore 的檔案（例如憑證）
```

---

<a id="release"></a>

## 🔁 tag 驅動的發版流程

**只有打 tag 才部署**，push 到分支完全沒有動作。tag 前綴決定部署到哪一種環境。

| tag | 來自 | 目標 | 時機 |
|-----|------|------|------|
| `rev.*` | development | 測試機 | 自動跑 |
| `v.*` | main | 正式機 | 手動按按鈕 |

> feature 分支怎麼建立、commit、合併回 development 的細節見 [Git 使用筆記的專案流程](git.md#workflow)。這裡接著那個流程之後，講合併進 development／main 之後怎麼打 tag 觸發部署。

### 階段一：發測試版

```shell
git switch development
git pull
git merge feature/xxx --no-ff      # 合併，保留分支歷史

git tag -a rev.2.0.260827 -m "release rev.2.0.260827"
git push origin development rev.2.0.260827   # 一次推送分支與標籤，後者才觸發部署
```

### 階段二：升到正式版

```shell
git switch main
git pull
git merge development --no-ff

git tag -a v.2.0.260827 -m "release v.2.0.260827"
git push origin main v.2.0.260827  # 一次推送分支與標籤，等你按按鈕
```

好處是 main 的歷史就是上線過的版本。代價是多一次 merge 與一次 tag。

> **一律用 `git tag -a`（annotated tag），不要用不帶參數的輕量 tag。** 兩者對 `rules: - if: $CI_COMMIT_TAG =~ /^v\./` 這種比對沒有差別，但輕量 tag 只是一個指向 commit 的指標，不記錄是誰、何時、為什麼打的；annotated tag 是一個完整物件，帶 tagger、時間戳與訊息，`git show <tag>` 才看得到這些資訊，`git describe` 等工具預設也只認 annotated tag。發版這種要事後追溯的操作，補上 `-m` 訊息幾乎零成本，出事時才不用回頭問「這個 tag 是誰打的、對應哪次改動」。

> 正式機部署的 commit SHA 跟測試機驗過的不是同一個，多了 main 的 merge commit。main 沒有其他變更時檔案內容完全相同，實務上沒差。但若有人直接對 main 推 hotfix，那個 merge 會混入沒被驗過的東西，這是流程紀律，CI 擋不了。

### 回滾

翻到舊 tag 的 pipeline，重按那一版的部署 job。

**不要用「打一個新 tag 指向舊 commit」的方式**，版本號與程式內容的對應會亂掉，之後查問題對不起來。

### 改到 `.gitlab-ci.yml` 本身時

不要直接打 tag。tag 是不可變的參照，CI 設定寫錯就得刪 tag 重打或跳號，而測試機可能已經被弄壞一半。

先用 **CI/CD → Pipelines → New pipeline** 選功能分支跑一次，這就是 rules 裡那條 `CI_PIPELINE_SOURCE == "web"` 存在的理由，不用改設定檔也不用製造多餘的 commit。

---

<a id="trouble"></a>

## 🚨 疑難排解

| 症狀 | 原因 / 解法 |
|------|-------------|
| push tag 後 Pipelines 沒有新項目 | tag 前綴不符 rules。`rev-2.0`、`v2.0` 都不會觸發，而且**不會有任何錯誤訊息** |
| push 分支後沒有新項目 | 正常，只有 tag 會觸發部署 |
| 連 tag 也完全沒反應 | 專案的 CI/CD 功能被關，或設定檔路徑被改。兩者都不報錯 |
| 長時間停在 `Created` | GitLab 背景工作沒在消化，查 `/admin/background_jobs`，`sudo gitlab-ctl restart sidekiq` |
| 卡在 `Pending`、`no runners matched` | job 的 `tags` 沒有對應的 Runner，或 Runner 被 Paused、勾了 Protected 但 ref 不是 protected |
| 顯示 `Waiting for resource` | `resource_group` 在排隊，同一個環境有另一個部署還沒跑完。正常行為 |
| `DEPLOY_ENV not set` | 變數勾了 Protect，但觸發的 ref 不是 protected |
| `permission denied ... docker daemon` | `usermod -aG docker gitlab-runner` 後沒 restart |
| Runner 關掉終端機就失效 | 註冊時漏了 `sudo`，變成 user-mode |
| build 失敗 | 程式問題，與 CI 無關。**舊容器仍在跑**，線上服務沒事 |

---

<a id="ref"></a>

## 📚 參考資料

- [GitLab CI/CD YAML 語法參考](https://docs.gitlab.com/ee/ci/yaml/)
- [rules](https://docs.gitlab.com/ee/ci/yaml/#rules)
- [預定義變數一覽](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)
- [resource_group](https://docs.gitlab.com/ee/ci/resource_groups/)
- [Protected tags](https://docs.gitlab.com/ee/user/project/protected_tags.html)
- [GitLab Runner](https://docs.gitlab.com/runner/)