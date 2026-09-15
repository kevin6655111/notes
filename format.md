# 程式碼統一格式化

> 以 husky + lint-staged 在 commit 前自動格式化，前端與設定檔用 Prettier，C# 用 CSharpier，JS / TS 語法檢查用 ESLint。適用 .NET + Node.js 混合專案。

## 目錄

- [🪝 觸發環境：husky & lint-staged](#husky)
- [🎨 Prettier](#prettier)
- [🔁 Gulp（整批格式化）](#gulp)
- [🟣 CSharpier（.NET）](#csharpier)
- [🧹 ESLint](#eslint)
- [🆕 新專案快速上手：eslint --init 與 VS Code 套件](#quickstart)
- [🧩 Rider 格式化 .cshtml](#rider)
- [🚀 套用到既有專案](#apply)
- [📚 參考資料](#ref)

---

<a id="husky"></a>

## 🪝 觸發環境：husky & lint-staged

husky 負責掛 git hook，lint-staged 只對本次 staged 的檔案執行對應指令。

```shell
yarn init
yarn add -D husky@^7.0.4
yarn add -D lint-staged@^12.3.3
touch .lintstagedrc.mjs
```

`package.json`：`yarn install` 時會自動執行 `husky install` 建立 hook

```json
"scripts": {
  "prepare": "husky install"
}
```

```shell
yarn install
```

`.husky/pre-commit`：先確認 prettier 與 csharpier 都存在，再跑 lint-staged

```shell
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

if ! yarn prettier -v; then
    echo "not found prettier"
    exit 1
fi

if ! dotnet csharpier --version; then
    echo "not found csharpier"
    exit 1
fi

yarn lint-staged
```

`.lintstagedrc.mjs`：依副檔名分派給 Prettier 或 CSharpier

```js
export default {
  "**/*.+(js|mjs|json|ts|htm|html|yml|css|md|less|xml|resx|csproj|svg|config)": (filenames) => {
    return `yarn prettier --config .prettierrc.json --write ${filenames.join(" ")}`;
  },
  "**/*.cs": (filenames) => {
    return `dotnet csharpier ${filenames.join(" ")}`;
  },
};
```

---

<a id="prettier"></a>

## 🎨 Prettier

處理 `.js` `.json` `.ts` `.htm` `.html` `.yml` `.css` `.md` `.less` `.xml` `.resx` `.csproj` `.config`

```shell
yarn add -D prettier
yarn add -D @prettier/plugin-xml@^1.2.0     # 讓 Prettier 支援 xml / resx / csproj / config
touch .prettierrc.json
```

`.prettierrc.json`

```json
{
  "trailingComma": "es5",
  "semi": true,
  "tabWidth": 2,
  "singleQuote": false,
  "printWidth": 115,
  "endOfLine": "lf",
  "overrides": [
    {
      "files": "*.dbml",
      "options": {
        "parser": "xml"
      }
    }
  ]
}
```

| 選項 | 說明 |
|------|------|
| `trailingComma` | 多行逗號分隔的程式碼，盡量補上結尾逗號 |
| `semi` | 語句末尾補分號 |
| `tabWidth` | 每層縮排的空格數 |
| `singleQuote` | 用單引號取代雙引號 |
| `printWidth` | 每行最大長度 |
| `endOfLine` | 換行字元，統一 `lf`（見 [Git 換行字元 EOL / CRLF 規範](git-eol.md)） |
| `overrides` | 對特定檔案套用不同設定 |

- [Prettier Options](https://prettier.io/docs/en/options.html)
- [Prettier Configuration](https://prettier.io/docs/en/configuration.html)

---

<a id="gulp"></a>

## 🔁 Gulp（整批格式化）

lint-staged 只處理 staged 檔案，整個專案一次格式化或在 CI 檢查時用 gulp。

```shell
yarn add -D gulp@^4.0.2
yarn add -D gulp-prettier@^4.0.0
touch gulpfile.js
touch .prettierinclude.js
```

`.prettierinclude.js`：要納入格式化的檔案範圍

```js
const prettierinclude = [
  "**/*.ts",
  "**/*.json",
  "**/*.mjs",
  "**/*.js",
  "!./node_modules/**",
];

module.exports = { prettierinclude };
```

`gulpfile.js`

```js
const gulp = require("gulp");
const prettier = require("gulp-prettier");

const { prettierinclude } = require("./.prettierinclude");

const prettierFile = () =>
  gulp
    .src(prettierinclude, { base: "./", removeBOM: false, dot: true })
    .pipe(prettier({ editorconfig: true }))
    .pipe(gulp.dest("./"));

const prettierFileCheck = () =>
  gulp.src(prettierinclude, { base: "./" }).pipe(prettier.check({ editorconfig: true }));

exports.fmtPrettier = prettierFile;
exports.fmtPrettierCheck = prettierFileCheck;
```

`package.json`

```json
"scripts": {
  "fmt:prettier": "gulp fmtPrettier",
  "fmt:prettier:check": "gulp fmtPrettierCheck"
}
```

---

<a id="csharpier"></a>

## 🟣 CSharpier（.NET）

處理 `.cs`

```shell
dotnet tool install --global csharpier
```

`.csharpierrc.json`

```json
{
  "printWidth": 115,
  "useTabs": false,
  "tabWidth": 4,
  "trimInitialLines": true
}
```

| 選項 | 說明 |
|------|------|
| `printWidth` | 每行最大長度 |
| `useTabs` | 縮排用 tab 還是空格 |
| `tabWidth` | 每層縮排長度 |
| `trimInitialLines` | 移除檔案開頭的空白行 |

`package.json`

```json
"scripts": {
  "fmt:cs": "dotnet csharpier ."
}
```

### 呼叫不到 dotnet tool

global tool 安裝路徑沒在 PATH，需加入環境變數：

| OS | 路徑 |
|----|------|
| Linux / macOS | `$HOME/.dotnet/tools` |
| Windows | `%USERPROFILE%\.dotnet\tools` |

```shell
# ~/.zshrc 或 ~/.bashrc
export PATH="$PATH:$HOME/.dotnet/tools"
```

- [CSharpier](https://csharpier.com/)

---

<a id="eslint"></a>

## 🧹 ESLint

`.js` `.ts` 語法檢查（Prettier 只管排版，不管語法）

```shell
# ESLint 語法檢查工具
yarn add -D eslint@^8.8.0

# TypeScript 編譯器
yarn add -D typescript@^4.5.5

# ESLint 預設用 Espree 解析，不認得 TypeScript 語法，改用 typescript-eslint 的 parser
yarn add -D @typescript-eslint/parser@^5.11.0

# 補上 parser 對部分 ESLint 規則支援不佳的地方
yarn add -D @typescript-eslint/eslint-plugin@^5.11.0

# 關閉所有會與 Prettier 衝突的排版規則
yarn add -D eslint-config-prettier@^8.3.0
```

`.eslintrc.js`

```js
module.exports = {
  root: true,
  env: {
    es2017: true,
    jquery: true,
    browser: true,
    node: true,
  },
  parserOptions: {
    ecmaVersion: 5,
  },
  extends: ["eslint:recommended", "prettier"],
  rules: {
    complexity: ["warn", 10],
    curly: ["error"],
    semi: ["warn", "always"],
    "no-shadow": ["warn"],
    "jsx-quotes": ["warn", "prefer-double"],
    quotes: ["warn", "double", { allowTemplateLiterals: true }],
    "max-params": ["warn", 5],
  },
  ignorePatterns: ["WebApp.EPS/wwwroot/js/pages/**/*.js"],
  overrides: [
    {
      files: ["./**/*.ts", "./**/*.tsx"],
      parser: "@typescript-eslint/parser",
      plugins: ["@typescript-eslint"],
      extends: [
        "eslint:recommended",
        "plugin:@typescript-eslint/recommended",
        "plugin:@typescript-eslint/recommended-requiring-type-checking",
        "prettier",
      ],
      rules: {
        complexity: ["warn", 10],
        curly: ["error"],
        semi: ["warn", "always"],
        "no-shadow": ["warn"],
        "jsx-quotes": ["warn", "prefer-double"],
        quotes: ["warn", "double", { allowTemplateLiterals: true }],
        "max-params": ["warn", 5],
        "@typescript-eslint/explicit-module-boundary-types": "off",
        "@typescript-eslint/prefer-regexp-exec": "off",
      },
      parserOptions: {
        tsconfigRootDir: __dirname,
        project: ["./tsconfig.json"],
      },
    },
  ],
};
```

| 選項 | 說明 |
|------|------|
| `root` | ESLint 預設會往父目錄一路找設定檔到根目錄，設 `true` 表示到此為止 |
| `env` | 指定執行環境，帶入該環境的全域變數 |
| `parserOptions` | 指定 JavaScript 語言版本與風格 |
| `plugins` | 載入外掛，名稱可省略 `eslint-plugin-` 前綴 |
| `extends` | 繼承現成的規則集 |
| `rules` | 覆蓋 extends 或 plugins 帶進來的規則 |
| `overrides` | 對特定檔案（此處為 .ts / .tsx）套用不同的 parser 與規則 |

`package.json`

```json
"scripts": {
  "check-js": "yarn eslint . --ext .js",
  "check-ts": "yarn eslint . --ext .ts"
}
```

- [@typescript-eslint/parser](https://www.npmjs.com/package/@typescript-eslint/parser?activeTab=readme)
- [ESLint Configuring](https://eslint.org/docs/latest/use/configure/)
- [ESLint Demo](https://eslint.org/demo)

---

<a id="quickstart"></a>

## 🆕 新專案快速上手：eslint --init 與 VS Code 套件

上一節是手寫 `.eslintrc.js`；全新專案可用 `eslint --init` 互動式產生設定檔，並搭配 VS Code 套件在存檔時即時提示。

### VS Code 套件

| 套件 | 功能 |
|------|------|
| ESLint (dbaeumer.vscode-eslint) | 編輯時即時標示語法問題 |
| Prettier - Code formatter (esbenp.prettier-vscode) | 存檔時自動排版 |

`.vscode/settings.json`

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  }
}
```

### eslint --init

```shell
npm install -g eslint      # 全域安裝，方便直接下 eslint 指令
npx eslint --init          # 互動式建立設定檔
```

互動問答（ESLint 8 的畫面）：

```text
? How would you like to use ESLint? ...
  To check syntax only
> To check syntax and find problems
  To check syntax, find problems, and enforce code style

? What type of modules does your project use? ...
> JavaScript modules (import/export)
  CommonJS (require/exports)
  None of these

? Which framework does your project use? ...
> React
  Vue.js
  None of these

? Does your project use TypeScript? > No / Yes

? Where does your code run? ...
> Browser
  Node

? What format do you want your config file to be in? ...
  JavaScript
  YAML
> JSON

? Would you like to install them now? No / > Yes

? Which package manager do you want to use? ...
> npm
  yarn
  pnpm
```

> ESLint 9 起預設改用 flat config（`eslint.config.js`），問答選項也不同；沿用本篇 `.eslintrc.js` 寫法請鎖定 ESLint 8。

- [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript)
- [VS Code Prettier 套件筆記](https://hackmd.io/@Heidi-Liu/note-prettier)

---

<a id="rider"></a>

## 🧩 Rider 格式化 .cshtml

Prettier 與 CSharpier 都不處理 `.cshtml`，交給 JetBrains Rider：

1. 在 IDE 搜尋 Action `Reformat File`。
2. 加入新的格式化設定（Code Style → C# / Razor），與 CSharpier 對齊：縮排 4 空格、每行 115。
3. 對整個專案執行 Reformat。

---

<a id="apply"></a>

## 🚀 套用到既有專案

設定完成後，先把現有程式碼整批格式化一次並 commit，之後才由 pre-commit 維持：

```shell
yarn fmt:prettier
yarn fmt:cs
# .cshtml 用 Rider 格式化
```

---

<a id="ref"></a>

## 📚 參考資料

- [husky](https://typicode.github.io/husky/)
- [lint-staged](https://github.com/lint-staged/lint-staged)
- [Prettier](https://prettier.io/)
- [CSharpier](https://csharpier.com/)
- [ESLint](https://eslint.org/)
