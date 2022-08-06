---
sidebar: auto
---

# GUIDE

## 安装 pnpm

```shell
npm install -g pnpm

pnpm add pnpm -wD
```

## 工程初始化

创建目录

```shell

mkdir pnpm-monorepo-playground && cd pnpm-monorepo-playground

```

推荐一个 vscode 实用插件[gitignore](https://marketplace.visualstudio.com/items?itemName=codezombiech.gitignore)

快捷键 `ctrl + shift + p` 输入 `add gitignore` 搜索 `node` 选择，再搜索 `macos` or `windows` 选择 `append`。

```shell

mkdir packages && cd packages

mkdir pkg1 pkg2

cd pkg1 && pnpm init && mkdir src && cd src && touch index.ts && cd ..

cd pkg2 && pnpm init && mkdir src && cd src && touch index.ts && cd ../../

git init && pnpm init

```

修改根目录 `packages.json` 添加 `"private": true`，防止根目录给 npm 发布出去，添加 `preinstall`，限制只允许使用 `pnpm` 安装依赖。

```json
// packages.json
{
  "name": "pnpm-monorepo-playground",
  "private": true,
  "scripts": {
    "preinstall": "npx only-allow pnpm",
    ...
  }
  ...
}
```

创建 workspaces

```shell

code pnpm-workspace.yaml

# pnpm-workspace.yaml input:
packages:
  - 'packages/**'
# pnpm-workspace.yaml END.

# 建立索引
pnpm i
```

此时目录结构

```shell
│  .gitignore
│  package.json
│  pnpm-workspace.yaml
│
└─packages
    ├─pkg1
    │  │  package.json
    │  │
    │  └─src
    │          index.ts
    │
    └─pkg2
        │  package.json
        │
        └─src
                index.ts
```

添加 typescript 到项目中

```shell
git add .

pnpm add typescript -wD

pnpm exec tsc --init

# 改个名
mv tsconfig.json tsconfig.base.json

code tsconfig.json

```

tsconfig.json

```json
// tsconfig.json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "baseUrl": "."
  }
}
```

```json
// /packages/pkg1/tsconfig.json and /packages/pkg2/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "include": ["src"],
  "compilerOptions": {
    "outDir": "dist",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

package.json

```json
// /packages/pkg1/package.json and /packages/pkg2/package.json
{
  ...,
  "main": "./dist/index.mjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "require": "./dist/index.cjs",
      "import": "./dist/index.mjs",
      "types": "./dist/index.d.ts"
    }
  },
  ...
}
```

pkg1/src/index.ts

```typescript
// /packages/pkg1/src/index.ts
export function logpkg1() {
  console.log('ADI-LOG => pkg1');
}
```

pkg2/src/index.ts

```typescript
// /packages/pkg2/src/index.ts
import { logpkg1 } from 'pkg1';

export function logpkg2() {
  console.log('ADI-LOG => pkg2');
}

function main() {
  logpkg1();
  logpkg2();
}
main();
```

把 pkg1 作为依赖添加到 pkg2 的开发依赖中

```shell
pnpm add pkg1 -F pkg2 --workspace -D
```

此时打开 `/packages/pkg2/src/index.ts` ts 会提示找不到模块 `pkg1`，我们通过添加 `paths`，告诉 ts 怎么解析我们的 `require/import`

```json
// /packages/pkg2/package.json
{
  ...,
  "compilerOptions": {
    ...,
    "paths": {
      ...,
      "pkg1": [
        "../pkg1/src/index.ts"
      ]
    }
  }
}

```

接下来我们用 `tsc` 简单打包一下我们项目，查看一下效果

```shell
pnpm -F "./packages/*" exec tsc
```

接下来我们用 [`unbuild`](https://github.com/unjs/unbuild) 来打包我们的项目

```shell
pnpm add unbuild -wD
```

创建 `build.config.ts` 到 `packages/*` 下

```typescript
// /packages/pkg1/build.config.ts
// /packages/pkg2/build.config.ts
import { defineBuildConfig } from 'unbuild';

export default defineBuildConfig({
  entries: ['src/index'],
  declaration: true,
  clean: true,
  rollup: {
    emitCJS: true,
    inlineDependencies: true,
  },
});
```

添加打包指令到 `packages/*` 下

```json
// /packages/pkg1/package.json
// /packages/pkg2/package.json
{
  ...,
  "scripts": {
    "build": "unbuild",
    ...
  },
  ...
}

```

修改根目录的`package.json`

```json
// /package.json
{
  ...,
  "scripts": {
    "build": "pnpm -r --filter \"./packages/*\" build",
    ...
  },
  ...
}

```

让我们打包看看效果吧

```shell
pnpm build

node .\packages\pkg2\dist\index.cjs
node .\packages\pkg2\dist\index.mjs
# output
# ADI-LOG => pkg1
# ADI-LOG => pkg2
```

## 配置格式化风格

#### editorconfig

```shell
code .editorconfig

# 添加下面内容

root = true
[*]
charset = utf-8
indent_style = space
indent_size = 2
end_of_line = lf
trim_trailing_whitespace = true
insert_final_newline = true

```

#### eslint

```shell
git add .

pnpm add eslint -wD

pnpm create @eslint/config

```

添加/修改 `.eslint.js`, `.eslintignore`

```json
// /.eslint.js
module.exports = {
  env: {
    es2021: true,
    node: true,
  },
  extends: ["eslint:recommended", "plugin:@typescript-eslint/recommended"],
  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaVersion: "latest",
    sourceType: "module",
  },
  plugins: ["@typescript-eslint"],
  rules: {
    "no-console": ["error", { allow: ["log", "warn"] }],
    "comma-dangle": ["error", "only-multiline"],
  },
};
```

```yaml
# /.eslintignore
dist node_modules pnpm-lock.yaml
```

trytry， 用 `eslint` 修复一下

```shell
git add .
pnpm exec eslint --cache --fix .
```

添加脚本到`package.json`中

```json
"scripts": {
    ...,
    "lint:eslint": "eslint .",
    "format:eslint": "eslint --cache --fix .",
    ...
  },
```

#### prettier

```shell
git add .

pnpm add prettier

eslint-config-prettier

eslint-plugin-prettier -wD

echo {}> .prettierrc.json

echo dist .prettierignore
```

添加/修改 `.prettierrc.json`, `.prettierignore`

```json
// /.prettierrc.json
{
  "tabWidth": 2,
  "semi": true,
  "singleQuote": true,
  "quoteProps": "consistent",
  "trailingComma": "all",
  "bracketSpacing": true,
  "arrowParens": "avoid",
  "proseWrap": "never",
  "endOfLine": "lf"
}
```

```shell
# /.prettierignore
dist
node_modules
pnpm-lock.yaml
```

```javascript
// /.eslintrc.js
module.exports = {
  ...,
  extends: [
    ...,
    'plugin:prettier/recommended',
  ],
  rules: {
    'prettier/prettier': 'error',
    ...,
  },
  ...
};
```

trytry， 用 `prettier` 检查代码，再自动修复一下

```shell
git add .

# 检查代码
pnpm prettier --check .

# output
# $ Code style issues found in 12 files. Forgot to run Prettier?

# 修复代码
prettier --write .
```

添加脚本到 `package.json` 中

```json
"scripts": {
    ...,
    "lint:style": "prettier --check .",
    "format:style": "prettier --write .",
    ...
  },
```

#### styleLint

```shell
git add .

pnpm add stylelint stylelint-config-html stylelint-config-recommended-scss stylelint-config-recommended-vue stylelint-config-standard stylelint-config-standard-scss stylelint-order postcss postcss-html stylelint-config-rational-order stylelint-config-prettier -wD

code .stylelintrc.js
```

```javascript
// /.stylelintrc.js
module.exports = {
  extends: [
    'stylelint-config-standard',
    'stylelint-config-html/vue',
    'stylelint-config-standard-scss',
    'stylelint-config-recommended-vue/scss',
    'stylelint-config-prettier',
  ],
  plugins: ['stylelint-order', 'stylelint-config-rational-order/plugin'],
  rules: {
    'scss/dollar-variable-empty-line-before': [
      'always',
      {
        except: ['first-nested', 'after-comment', 'after-dollar-variable'],
      },
    ],
    'declaration-empty-line-before': null,
    'rule-empty-line-before': [
      'always',
      {
        except: ['first-nested', 'after-rule'],
        ignore: ['after-comment'],
      },
    ],
    'at-rule-empty-line-before': [
      'always',
      {
        except: ['after-same-name', 'first-nested'],
        ignore: ['after-comment'],
      },
    ],
    'plugin/rational-order': [
      true,
      {
        'border-in-box-model': true,
        'empty-line-between-groups': true,
      },
    ],
    'no-descending-specificity': null,
    'function-url-quotes': 'always',
    'string-quotes': 'double',
    'indentation': 2,
    'unit-case': null,
    'color-hex-case': 'lower',
    'color-hex-length': 'long',
    'rule-empty-line-before': 'never',
    'font-family-no-missing-generic-family-keyword': null,
    'block-opening-brace-space-before': 'always',
    'property-no-unknown': null,
    'no-empty-source': null,
    'selector-pseudo-class-no-unknown': [
      true,
      {
        ignorePseudoClasses: ['deep'],
      },
    ],
    'order/properties-order': [
      'position',
      'top',
      'right',
      'bottom',
      'left',
      'z-index',
      'display',
      'justify-content',
      'align-items',
      'float',
      'clear',
      'overflow',
      'overflow-x',
      'overflow-y',
      'margin',
      'margin-top',
      'margin-right',
      'margin-bottom',
      'margin-left',
      'padding',
      'padding-top',
      'padding-right',
      'padding-bottom',
      'padding-left',
      'width',
      'min-width',
      'max-width',
      'height',
      'min-height',
      'max-height',
      'font-size',
      'font-family',
      'font-weight',
      'border',
      'border-style',
      'border-width',
      'border-color',
      'border-top',
      'border-top-style',
      'border-top-width',
      'border-top-color',
      'border-right',
      'border-right-style',
      'border-right-width',
      'border-right-color',
      'border-bottom',
      'border-bottom-style',
      'border-bottom-width',
      'border-bottom-color',
      'border-left',
      'border-left-style',
      'border-left-width',
      'border-left-color',
      'border-radius',
      'text-align',
      'text-justify',
      'text-indent',
      'text-overflow',
      'text-decoration',
      'white-space',
      'color',
      'background',
      'background-position',
      'background-repeat',
      'background-size',
      'background-color',
      'background-clip',
      'opacity',
      'filter',
      'list-style',
      'outline',
      'visibility',
      'box-shadow',
      'text-shadow',
      'resize',
      'transition',
    ],
  },
};
```

trytry `stylelint`

```shell
code code .\packages\pkg1\test.scss
```

```scss
.adi {
  color: #ccc;
  z-index: 1;
  display: inline-block;
}
```

```shell
git add .

pnpm exec stylelint "**/*.(scss)" --fix
```

添加脚本到`package.json`中

```json
"scripts": {
    ...,
    "lint:stylelint": "stylelint \"**/*.{vue,less,postcss,css,scss}\" --cache --cache-location node_modules/.cache/stylelint/",
    "format:stylelint": "stylelint --fix \"**/*.{vue,less,postcss,css,scss}\" --cache --cache-location node_modules/.cache/stylelint/",
    ...
  },
```

## 配置工具链

#### npm-run-all

```shell
pnpm add npm-run-all -wD
```

添加脚本到`package.json`中

```json
"scripts": {
    ...,
    "lint": "run-p lint:*",
    "format": "run-p format:*",
    ...
  },
```

trytry

```shell
git add .

pnpm lint
# output: has error

pnpm format
# output: successfully!

pnpm lint
# output: All matched files use Prettier code style!
```

#### husky

```shell
pnpm add lint-staged husky -wD
```

添加脚本到 `package.json`

```json
scripts": {
  ...,
  "prepare": "husky install"
},
"lint-staged": {
  "*.{js,jsx,ts,tsx}": [
    "eslint --cache --fix .",
    "prettier --write .",
    "git add"
  ],
  "*.{json,md}": [
    "prettier --write .",
    "git add"
  ],
  "*.{scss,less,styl,html}": [
    "stylelint --fix \"**/*.{vue,less,postcss,css,scss}\" --cache --cache-location node_modules/.cache/stylelint/",
    "prettier --write .",
    "git add -A"
  ]
}
```

初始化 `husky`

```shell
pnpm prepare

npx husky add .husky/pre-commit "npx --no-install lint-staged"
```

#### commitizen and cz-conventional-changelog

```shell
pnpm add commitizen cz-conventional-changelog -wD

```

配置 `package.json`

```json
// package.json
{
  ...,
  "scripts": {
    ...,
    "commit": "cz"
  },
  ...,
}
```

后面就可以用 `pnpm commit` 替代 `git commit`，规范 git 提交。

#### commitlint

使用 commitlint 来校验提交时 commit message 是否符合 conventional commit 规范。

```shell
pnpm add @commitlint/cli @commitlint/config-conventional commitizen cz-conventional-changelog -wD

code commitlint.config.js
```

```javascript
// /commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
};
```

```shell
npx husky add .husky/commit-msg "npx --no -- commitlint --edit $1"
```

trytry

```shell
echo console.log('adi'); > index.js

git commit -m "test commitlint" -a

# output:
# ⧗   input: test commitlint
# ✖   subject may not be empty [subject-empty]
# ✖   type may not be empty [type-empty]
# ✖   found 2 problems, 0 warnings
# # ⓘ   Get help: https://github.com/conventional-changelog/commitlint/#what-is-commitlint
# husky - commit-msg hook exited with code 1 (error)

rm index.js
```

#### changesets

```shell
pnpm add @changesets/cli -wD

pnpm changeset init
```

配置 `.changeset/config.json`

```json
// .changeset/config.json
{
  "$schema": "https://unpkg.com/@changesets/config@2.0.0/schema.json",
  // 生成 Changelog 规则。
  "changelog": "@changesets/cli/changelog",
  // 当配置该字段为 true 时，在执行 change 和 bump 命令时，将自动执行提交代码操作。
  "commit": false,
  // 用于 monorepo 中对包进行分组，相同分组中的包版本号将进行绑定，每次执行 bump 命令时，同一分组中的包只要有一个升级版本号，其他会一起升级。 支持使用正则匹配包名称。
  "fixed": [],
  // 和 fixed 类似，也是对 monorepo 中对包进行分组，但是每次执行 bump 命令时，只有和 changeset 声明的变更相关的包才会升级版本号，同一分组的变更包的版本号将保持一致。 支持使用正则匹配包名称。
  "linked": [],
  // npm包发布地址，私有推荐restricted ，开源推荐public
  "access": "public",
  // 项目主分支
  "baseBranch": "main",
  // 用于声明更新内部依赖的版本号规则。
  "updateInternalDependencies": "patch",
  // 用于声明执行 bump 命令时忽略的包
  "ignore": [],
  // 针对于 peerDependence 依赖的升级策略配置，默认针对 peerDependence 在 minor 和 major 版本升级时，当前包会升级大版本。该配置设置为 true 时，仅当 peerDependence 声明包依赖超出声明范围时才更新版本。
  "___experimentalUnsafeOptions_WILL_CHANGE_IN_PATCH": {
    "onlyUpdatePeerDependentsWhenOutOfRange": true
  }
}
```

配置 `package.json`

```json
// package.json
{
  ...,
  "scripts": {
    ...,
    // 创建一个 changeset，执行完成该命令后会自动在 .changeset 目录生成一个 changeset 文件。
    "change": "changeset",
    // 简写 version-packages
    "vp": "pnpm version-packages",
    // 消耗changeset，生成新版本
    "version-packages": "changeset version",
    // 打包、发布到 NPM
    "release": "pnpm build && pnpm release:only",
    "release:only": "changeset publish --registry=https://registry.npmjs.com/"
  },
  // npm发布模式, public/restricted
  "publishConfig": {
    "access": "public"
  },
  ...,
}
```

**项目发布流: **

1. 代码开发
2. 使用 `pnpm commit` 提交 commit
3. 使用 `pnpm change` 填写变更集
4. 项目 owner 使用 `pnpm vp` 消耗变更集，由 changesets 自动提升子包版本、生成 changelog
5. 使用 `pnpm release` 构建项目并完成发包

#### [truborepo](https://turborepo.org/docs/getting-started#create-turbojson)

```shell
pnpm add turbo -wD
```

create `turbo.json`

```json
{
  "$schema": "https://turborepo.org/schema.json",
  "pipeline": {
    "test": {
      "dependsOn": ["build"],
      "outputs": [],
      "inputs": ["src/**/*.ts", "test/**/*.ts", "src/**/*.tsx", "test/**/*.tsx"]
    },
    "lint": {
      "dependsOn": ["^lint"],
      "outputs": []
    },
    "dev": {
      "cache": false
    },
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["./dist/**"]
    }
  }
}
```

trytry，让我们先来试试用`turbo`接管我们的`build`

```shell
pnpm exec turbo run build

# output:
# Packages in scope: pkg1, pkg2
# • Running build in 2 packages
# Building pkg1
# cache miss, executing
# ...
# Build succeeded
```

我们在`build`一次，看看`turbo`的缓存效果。

```shell
pnpm exec turbo run build

# output:
# Packages in scope: pkg1, pkg2
#  Tasks:    2 successful, 2 total
# Cached:    2 cached, 2 total
#   Time:    351ms >>> FULL TURBO
```

接下来我们把 `pnpm exec turbo run build` 写成 `script`

```shell
pnpm add esno @umijs/utils rimraf -wD

# 创建文件
code scripts/turbo.ts
```

```typescript
// scripts/turbo.ts
import * as logger from '@umijs/utils/dist/logger';
import spawn from '@umijs/utils/compiled/cross-spawn';
import yArgs from '@umijs/utils/compiled/yargs-parser';
import { join } from 'path';

(async () => {
  const args = yArgs(process.argv.slice(2));
  const filter = args.filter || './packages/*';
  const extra = (args._ || []).join(' ');

  await turbo({
    cmd: args.cmd,
    filter,
    extra,
    cache: args.cache,
    parallel: args.parallel,
  });
})();

async function cmd(command: string) {
  const result = spawn.sync(command, {
    stdio: 'inherit',
    shell: true,
    cwd: join(__dirname, '../'),
  });
  if (result.status !== 0) {
    // sub package command don't stop when execute fail.
    // display exit
    logger.error(`Execute command error (${command})`);
    process.exit(1);
  }
  return result;
}

async function turbo(opts: {
  filter: string;
  cmd: string;
  extra?: string;
  cache?: boolean;
  parallel?: boolean;
}) {
  const extraCmd = opts.extra ? `-- -- ${opts.extra}` : '';
  const cacheCmd = opts.cache === false ? '--no-cache --force' : '';
  const parallelCmd = opts.parallel ? '--parallel' : '';

  const options = [
    opts.cmd,
    `--cache-dir=".turbo"`,
    `--filter="${opts.filter}"`,
    `--no-deps`,
    `--include-dependencies`,
    cacheCmd,
    parallelCmd,
    extraCmd,
  ]
    .filter(Boolean)
    .join(' ');

  return cmd(`turbo run ${options}`);
}
```

修改 `package.json` 里的 `scripts`

```json
// package.json

  "scripts": {
    // "build": "pnpm -r --filter \"./packages/*\" build",
    "build": "esno scripts/turbo.ts --cmd build",
    // 用于清除 trubo 缓存
    "clean:turbo": "rimraf .turbo"
  }

```

将 `.turbo` 添加到 `.gitignore`、`.eslintignore`、`.prettierignore`中

```
.turbo
```

trytry

```shell

pnpm clean:turbo

pnpm build

```

## Unit Testing

#### vitest

install:

```shell
pnpm add -wD vitest c8 jsdom @testing-library/jest-dom
```

修改 `package.json`

```json
// package.json
{
  "scripts": {
    "coverage": "c8 vitest run --coverage",
    "test": "vitest -w"
  }
}
```

创建 `vite.config.ts` 文件:

```typescript
// vite.config.ts

/// <reference types="vitest" />
/// <reference types="vite/client" />

import path from 'path';
import { defineConfig } from 'vite';

export default defineConfig({
  // https://github.com/vitest-dev/vitest
  test: {
    globals: true,
    environment: 'jsdom',
  },
});
```

测试一下效果，根目录新建 `__test__` 文件夹

```shell
mkdir __test__ && cd __test__

code sum.spec.ts
```

```typescript
// sum.spec.ts

test.concurrent('sum => success', () => {
  expect(1 + 1).toBe(2);
});
```

shell 执行 `pnpm test`，查看一下效果

```shell
pnpm test

# output
#  ✓ __test__/sum.spec.ts (1)
# Test Files  1 passed (1)
#      Tests  1 passed (1)
#       Time  3.64s (setup 0ms, collect 37ms, tests 4ms)
#  PASS  Waiting for file changes...
```

接来下试试测试一下 DOM:

创建 `dom.spec.ts`

```shell
code dom.spec.ts
```

```typescript
// dom.spec.ts

import '@testing-library/jest-dom';

describe('dom jest', () => {
  test.concurrent('jest-dom', async () => {
    const div = document.createElement('div');
    div.id = 'adm-mask';

    expect(div).not.toBeNull();
    expect(div).toBeDefined();
    expect(div).toBeInstanceOf(HTMLDivElement);

    await document.body.appendChild(div);
    const mask = document.querySelector('#adm-mask');

    expect(mask).toBeInTheDocument();

    div.remove();
    expect(mask).not.toBeInTheDocument();
  });
});
```

Vitest 智能搜索模块依赖树并只重新运行相关测试:

```shell
# output
#  ✓ dom.spec.ts (1)
# Test Files  2 passed (2)
#      Tests  2 passed (2)
#       Time  1.20s
```

可以自己尝试给 `pkg1` 和 `pkg2` 添加一下单元测试。

#### Coverage

通过 c8 来输出代码测试覆盖率。

```shell
pnpm coverage

# output
# Test Files  4 passed (4)
#      Tests  4 passed (4)
#       Time  5.51s (setup 2ms, collect 1.31s, tests 47ms)
# ------------|---------|----------|---------|---------|-------------------
# File        | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
# ------------|---------|----------|---------|---------|-------------------
# All files   |     100 |      100 |     100 |     100 |
#  pkg1/dist  |     100 |      100 |     100 |     100 |
#   index.mjs |     100 |      100 |     100 |     100 |
#  pkg1/src   |     100 |      100 |     100 |     100 |
#   index.ts  |     100 |      100 |     100 |     100 |
#  pkg2/src   |     100 |      100 |     100 |     100 |
#   index.ts  |     100 |      100 |     100 |     100 |
# ------------|---------|----------|---------|---------|-------------------
```

