## FE-Link 是什么

它是一个本地 npm 包调试工具, 解决 npm link 玄学问题。

## 背景：本地调试 npm 包有多痛苦

前端开发中经常遇到这种场景：组内维护了一个公共组件库或工具库，发布到 npm 上供多个项目使用。某天业务项目发现了 bug，需要在本地调试这个 npm 包。

这时候你有几个选择 ：

### 方案一：改一行发一版

改代码 → 发 npm → 项目里 npm install → 发现还有问题 → 再改 → 再发...

这不是调试，这是折磨。而且污染了版本号，npm 上多了一堆 `1.0.1-beta.1`、`1.0.1-beta.2`。

### 方案二：npm link

官方推荐的方案。在包目录执行 `npm link`，在项目目录执行 `npm link my-package`。

听起来很美好，实际上：

- 全局污染，link 了就忘了 unlink
- 多个项目 link 同一个包会互相覆盖
- pnpm 的 link 行为和 npm 不一样
- monorepo 里 link 经常出问题
- 有时候就是不生效，玄学

### 方案三：file: 协议

在 package.json 里直接写本地路径：

```json
{
  "dependencies": {
    "my-utils": "file:../my-utils"
  }
}
```

这个方案相对靠谱，但也有问题：

- 路径是相对的，换台电脑就挂了
- 提交代码前要记得改回来
- pnpm 对 `file:` 的处理和 npm 完全不同（后面细说）

## FE-Link 的解决思路

既然现有方案都有问题，那就自己造一个。核心思路很简单：

1. **本地仓库**：把要调试的包存到统一的地方 `~/.fe-link/`
2. **项目副本**：在项目里创建 `.fe-link/` 目录，把包复制进去
3. **修改依赖**：自动改 package.json，指向本地副本
4. **文件监听**：开启监听后，包的改动自动同步到所有使用它的项目

这样做的好处：

- 包和项目解耦，路径不再是问题
- 可以同时在多个项目里调试同一个包
- 改动实时同步，不用手动操作
- 不污染 npm 版本号

## 踩坑记录

### 坑一：pnpm 的 file: 协议不是符号链接

npm 处理 `file:` 协议时，会创建一个符号链接（symlink）指向目标目录。你改了源文件，项目里立刻生效。

pnpm 不一样。pnpm 的 `file:` 协议会**复制**文件到 node_modules，而不是创建符号链接。这意味着你改了源文件，项目里的代码纹丝不动，必须重新 `pnpm install` 才能看到变化。

**解决方案**：根据包管理器使用不同协议。

```json
// npm/yarn 项目
"my-utils": "file:.fe-link/my-utils"

// pnpm 项目
"my-utils": "link:.fe-link/my-utils"
```

为什么这样设计？
- pnpm 的 `link:` 协议会创建符号链接，改动立刻生效，Vite 能监听到变化
- npm/yarn 的 `file:` 协议本身就是符号链接，行为一致
- Push 操作会自动执行 `pnpm install` 创建符号链接，确保更新生效


### 坑二：Vite 的依赖缓存

解决了 pnpm 的问题，又遇到新问题：Vite 项目里 push 了代码，页面还是旧的。

原因是 Vite 会预构建 node_modules 里的依赖，缓存在 `node_modules/.vite` 目录。即使源文件变了，Vite 还是用的缓存。

**解决方案**：push 的时候顺手把 `.vite` 目录删掉。

### 坑三：批量操作的性能问题

在实际使用中发现，当一个项目依赖多个本地包时，批量移除操作会非常慢。

**问题根源**：每次 remove 操作都会执行一次完整的 `npm/pnpm install`。如果要移除 5 个包，就要执行 5 次 install，每次耗时 10-30 秒，总共需要 50-150 秒。

**解决方案**：实现批量移除接口 `RemoveBatch`，将所有包的清理操作合并，只在最后执行一次 install。

```go
// 批量处理流程
for each package {
  - 删除 .fe-link/package
  - 删除 node_modules/package  
  - 更新 package.json
  - 更新 fe-link.lock
}
// 只执行一次 install
npm install
```


## 核心功能详解

### 1. 发布（Publish）

将本地 npm 包发布到 FE-Link 仓库。

**流程**：
1. 读取 package.json 获取包名和版本
2. 创建存储目录 `~/.fe-link/packages/包名/版本号/`
3. 复制包文件（排除 node_modules、.git 等）
4. 生成文件签名（MD5）
5. 保存源码路径到 `.fe-link-source` 文件

**签名机制**：
- 遍历所有文件计算 MD5
- 用于检测包是否有变化
- Push 时对比签名决定是否需要更新

### 2. 添加（Add）

将已发布的包添加到项目。

**流程**：
1. 从 `~/.fe-link/packages/` 查找包的最新版本
2. 复制到项目的 `.fe-link/包名/` 目录
3. 检测包管理器（pnpm/yarn/npm）
4. 更新 package.json：
   - 所有项目统一使用：`"pkg": "file:.fe-link/pkg"`
5. 自动执行 `npm/pnpm/yarn install`
6. 更新 `fe-link.lock` 文件
7. 记录安装信息到 `~/.fe-link/installations.json`

**为什么需要 install**：
- npm/yarn：`file:` 协议需要 install 创建符号链接
- pnpm：`file:` 协议需要 install 复制文件到 node_modules（首次添加）

### 3. 移除（Remove）

从项目中移除包。

**流程**：
1. 删除 `.fe-link/包名/` 目录
2. 清理空的作用域目录（如 `@scope/`）
3. 从 package.json 的 dependencies/devDependencies 中删除
4. 删除 `node_modules/包名/` 目录
5. 清除 Vite 缓存（`.vite` 目录）
6. 更新 `fe-link.lock` 文件
7. 自动执行 `npm/pnpm install` 清理依赖
8. 从安装记录中移除

**完整清理**：确保不残留任何文件，避免影响后续开发。

### 4. 推送（Push）

发布包并推送到所有使用它的项目。

**流程**：
1. 执行 Publish 发布最新版本
2. 从 `installations.json` 获取所有安装了此包的项目
3. 对每个项目：
   - 更新 `.fe-link/包名/` 目录
   - 检测包管理器，pnpm 项目切换协议为 `link:`
   - 更新 package.json 中的协议
   - 清除 Vite 缓存
   - 更新 `fe-link.lock`
   - 执行 `npm/pnpm/yarn install` 触发更新
     - npm/yarn：符号链接自动生效
     - pnpm：重新创建符号链接
4. 统计并报告更新的项目数量

**智能同步**：
- 检测包是否真的有变化（对比签名）
- 只更新安装了此包的项目
- pnpm 项目自动切换到 `link:` 协议，确保 Vite 能监听到变化

### 5. 监听（Watch）

监听包的源码目录，自动推送变更。

**实现细节**：
- 使用 `fsnotify` 监听文件系统事件
- 防抖机制：1 秒内的多次变更合并为一次推送
- 忽略特定文件：`node_modules`、`.git`、`dist` 等
- 递归监听所有子目录
- 支持同时监听多个包

**防抖逻辑**：
```go
debounceTimer := time.NewTimer(1 * time.Second)
for {
  select {
  case event := <-watcher.Events:
    debounceTimer.Reset(1 * time.Second)  // 重置定时器
  case <-debounceTimer.C:
    push()  // 1 秒无活动后执行推送
  }
}
```

### 6. 分支包发布（Branch Package）

快速验证包功能，无需发布正式版本。自动创建临时分支并推送到公共仓库，触发 CI 发布带分支标识的测试版本。

**流程**：
1. 读取本地包的 package.json
2. 生成分支标识：`包名-时间戳-随机hash`
3. 生成新版本号：`原版本-时间戳-随机hash`
4. Clone 公共仓库到临时目录
5. 创建新分支
6. 读取公共仓库的 package.json（保留 scripts 和 dependencies）
7. 清理仓库内容（保留 .git）
8. 复制本地包文件到仓库
9. 修改 package.json，更新版本号
10. 提交更改到 Git
11. 推送分支到远程仓库
12. 触发 CI/CD 流水线自动发布

**使用场景**：
- 需要在真实环境中快速验证包功能
- 不想污染主版本号
- 需要多人协作测试同一个包的不同改动
- 在正式发版前进行最后验证

**优势**：
- 自动化流程，无需手动操作
- 版本号带时间戳和随机标识，不会冲突
- 保留公共仓库的构建配置
- 支持查看 CI 流水线状态

## 技术架构

### 数据流

**发布流程**：
- 源码目录 → Publish → 存储到 ~/.fe-link/packages/

**监听与推送**：
- 源码目录 → Watch 监听变化 → Push 推送到所有项目
- 推送目标：项目A/.fe-link/、项目B/.fe-link/、项目C/.fe-link/
- 每个项目：更新 package.json → 更新 node_modules/ → 执行 npm install

### 包管理器兼容性

**npm/yarn 项目**：
- 协议：`file:`
- 行为：创建符号链接
- Add 操作：执行 install 创建符号链接
- Push 更新：自动生效（符号链接直接指向 .fe-link）

**pnpm 项目**：
- 协议：`link:`
- 行为：创建符号链接
- Add 操作：执行 install 创建符号链接
- Push 更新：执行 install 重新创建符号链接



## 使用场景

### 场景一：组件库开发

**问题**：维护一个 UI 组件库，需要在多个业务项目中实时调试。

**解决方案**：
1. Publish 组件库到 FE-Link
2. 在各业务项目中 Add 组件库
3. 开启 Watch 监听
4. 修改组件代码，所有项目自动更新

**效果**：
- 一次修改，多项目同步
- 无需发布 npm 版本
- 实时查看效果

### 场景二：工具库调试

**问题**：工具库出现 bug，需要在实际项目中复现和修复。

**解决方案**：
1. Publish 工具库到 FE-Link
2. 在出问题的项目中 Add 工具库
3. 修改代码，实时验证修复效果
4. 确认无误后发布正式版本

**效果**：
- 快速定位问题
- 在真实环境中验证
- 避免污染 npm 版本号

### 场景三：Monorepo 包调试

**问题**：Monorepo 中的包需要在外部项目中测试。

**解决方案**：
1. Publish Monorepo 中的包
2. 在外部项目中 Add
3. 开启 Watch 实时同步

**效果**：
- 打破 Monorepo 边界
- 在真实场景中测试
- 保持开发效率

### 场景四：快速验证包功能

**问题**：改了包的代码，想在真实项目中验证，但不想发正式版本。

**解决方案**：
1. 点击「发布分支包」
2. 选择要发布的包
3. 输入公共仓库的 Git 地址
4. 等待 CI 构建完成
5. 在项目中安装临时版本：`npm install 包名@版本号`

**效果**：
- 无需本地调试环境
- 可以分享给其他人测试
- 不污染正式版本号
- CI 自动构建和发布


## 与其他工具对比

### vs npm link

**FE-Link 的优势**：
- 无全局污染（npm link 会全局污染）
- 完美支持多项目（npm link 多项目会互相覆盖）
- 完美兼容 pnpm（npm link 在 pnpm 下有问题）
- 无路径依赖（npm link 依赖路径）
- 支持自动同步（npm link 不支持）

### vs yalc

**FE-Link 的优势**：
- 有可视化界面（yalc 纯命令行）
- 内置文件监听（yalc 需配合其他工具）
- 批量操作优化（yalc 较慢）
- 自动检测包管理器（yalc 需手动处理）
- 性能提升 5-15 倍（yalc 性能一般）

### vs file: 协议

**FE-Link 的优势**：
- 自动路径管理（file: 需手动管理）
- 可跨电脑使用（file: 路径固定）
- 无提交风险（file: 容易误提交路径）
- 自动处理 pnpm（file: 需手动切换协议）

## 使用方式

### 快速开始

1. **发布包**
   - 将要调试的 npm 包目录拖入应用
   - 点击「发布」按钮
   - 包会被存储到 `~/.fe-link/packages/`

2. **添加到项目**
   - 将项目目录拖入应用
   - 选择要添加的包
   - 点击「添加」按钮
   - 自动修改 package.json 并执行 install

3. **开启监听**（可选）
   - 在包列表中点击「开启监听」
   - 修改包的源码会自动推送到所有项目
   - 实时查看效果

4. **移除包**
   - 在项目的依赖列表中点击「移除」
   - 自动清理所有相关文件
   - 批量移除性能更优

5. **发布分支包**（可选）
   - 点击「发布分支包」按钮
   - 选择要发布的包
   - 输入公共仓库 Git 地址（支持记住地址）
   - 等待发布完成，获取临时版本号
   - 在其他项目中通过 npm 安装临时版本
