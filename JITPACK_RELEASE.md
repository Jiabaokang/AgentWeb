# AgentWeb JitPack 发布指南

本指南详细说明如何将 `agentweb-core` 与 `agentweb-filechooser` 两个库模块通过 JitPack 发布与消费，并包含常见问题的解决策略与验证方法。

---

## 概述
- 发布目标：仅发布库模块（不构建 `sample`），生成 `aar` 与 `sources.jar`，并在 JitPack 上可下载。
- 当前坐标（基于提交版本）：
  - `com.github.Jiabaokang.AgentWeb:agentweb-core:d516bd2`
  - `com.github.Jiabaokang.AgentWeb:agentweb-filechooser:d516bd2`
- 标签版本说明：JitPack 对标签索引存在延迟。若短期内标签未被识别，可先使用提交版本坐标，稍后再切回标签版本。

---

## 前提条件
- 仓库已启用 `maven-publish` 插件，并在库模块中配置 `publishing`。
- 使用 `openjdk17`，Gradle 8.x（仓库当前为 Gradle 8.0）。
- 根目录存在 `jitpack.yml`，在 JitPack 构建的 install 阶段仅构建并发布库模块到 Maven Local，避免 `sample` 的第三方依赖问题。

---

## 模块与坐标
- 组织：`com.github.Jiabaokang.AgentWeb`
- 模块：
  - `agentweb-core`
  - `agentweb-filechooser`
- 版本：
  - 使用标签（如：`v5.1.7-androidx`）或提交（如：`d516bd2`）。

---

## 库模块 Gradle 配置要点

两个库模块的 `build.gradle` 都需：
- 应用 `maven-publish` 插件。
- 在 `android { publishing { singleVariant("release") } }` 声明发布变体（避免旧式组件访问时序问题）。
- 使用 `MavenPublication` 并直接附加 `release` 生成的 `aar` 文件；版本号使用 `project.version`（JitPack 会将其设置为请求的标签或提交）。

示例（以 `agentweb-core/build.gradle` 为例，仅展示关键段落）：

```groovy
apply plugin: 'com.android.library'
apply plugin: 'maven-publish'

android {
  // ... 省略 compileOptions / buildTypes 等
  publishing {
    singleVariant("release")
  }
}

afterEvaluate {
  publishing {
    publications {
      release(MavenPublication) {
        groupId = 'com.github.Jiabaokang.AgentWeb'
        artifactId = 'agentweb-core'
        version = project.version.toString() // JitPack 注入
        artifact("${buildDir}/outputs/aar/${project.name}-release.aar")
      }
    }
  }
}
```

> 说明：不再使用 `from components.release`（该属性在某些插件/版本时序下不可用），而是直接附加 `assembleRelease` 产出的 AAR。

`agentweb-filechooser/build.gradle` 同理：

```groovy
release(MavenPublication) {
  groupId = 'com.github.Jiabaokang.AgentWeb'
  artifactId = 'agentweb-filechooser'
  version = project.version.toString()
  artifact("${buildDir}/outputs/aar/${project.name}-release.aar")
}
```

---

## JitPack 配置（jitpack.yml）

项目根目录 `jitpack.yml`：

```yaml
jdk:
  - openjdk17
install:
  - ./gradlew :agentweb-core:assembleRelease :agentweb-filechooser:assembleRelease -x test --no-daemon --stacktrace
  - ./gradlew :agentweb-core:publishToMavenLocal :agentweb-filechooser:publishToMavenLocal -x test --no-daemon --stacktrace
```

- 先构建 `release` AAR，再发布到 MavenLocal，使 JitPack 能采集到 AAR 和 POM。
- 不构建 `sample`，避免 `com.tencent.sonic:sdk`、`top.zibin:Luban` 等第三方依赖缺失导致的失败。

---

## 发布流程

1) 推送变更到远端分支（例如：`androidx`）

```bash
git push origin HEAD
```

2) 创建并推送标签（可选，建议用于稳定版本）

```bash
git tag v5.1.7-androidx
git push origin v5.1.7-androidx
```

3) 触发 JitPack 构建
- 通过标签：访问 `https://jitpack.io/com/github/Jiabaokang/AgentWeb/v5.1.7-androidx/build.log`
- 通过提交：访问 `https://jitpack.io/com/github/Jiabaokang/AgentWeb/<shortSha>/build.log`

4) 验证构件是否发布成功
- POM（提交版示例）：
  - `https://jitpack.io/com/github/Jiabaokang/AgentWeb/agentweb-core/d516bd2/agentweb-core-d516bd2.pom`
  - `https://jitpack.io/com/github/Jiabaokang/AgentWeb/agentweb-filechooser/d516bd2/agentweb-filechooser-d516bd2.pom`
- AAR（提交版示例）：
  - `https://jitpack.io/com/github/Jiabaokang/AgentWeb/agentweb-core/d516bd2/agentweb-core-d516bd2.aar`
  - `https://jitpack.io/com/github/Jiabaokang/AgentWeb/agentweb-filechooser/d516bd2/agentweb-filechooser-d516bd2.aar`

---

## 在下游项目中使用

1) 添加 JitPack 仓库：

```groovy
repositories {
  maven { url 'https://jitpack.io' }
}
```

2) 添加依赖：
- 使用标签版本（示例，待 JitPack 索引完成后可用）：

```groovy
implementation 'com.github.Jiabaokang.AgentWeb:agentweb-core:v5.1.7-androidx'
implementation 'com.github.Jiabaokang.AgentWeb:agentweb-filechooser:v5.1.7-androidx'
```

- 使用提交版本（立即可用）：

```groovy
implementation 'com.github.Jiabaokang.AgentWeb:agentweb-core:d516bd2'
implementation 'com.github.Jiabaokang.AgentWeb:agentweb-filechooser:d516bd2'
```

> 注：提交版本坐标更适合快速验证与临时集成；稳定发布建议使用标签坐标。

---

## 常见问题与解决

- 错误：`Could not get unknown property 'release' for SoftwareComponentInternal ...`
  - 原因：在某些 Gradle/AGP 版本下，`components.release` 暴露时机不一致。
  - 解决：改为直接附加 AAR 文件（见上文 `artifact(...)`），并在 `android { publishing { singleVariant("release") } }` 声明发布变体。

- 错误：`Invalid publication 'release': artifact file does not exist: ...-release.aar`
  - 原因：尚未生成 `release` AAR。
  - 解决：在 `jitpack.yml` 的 `install` 阶段，先执行 `assembleRelease`，再执行 `publishToMavenLocal`。

- 错误：`Tag or commit not found`
  - 原因：JitPack 标签索引延迟，或标签未正确推送。
  - 解决：
    - 等待几分钟后重试标签构建日志。
    - 改用提交版本坐标进行构建与消费。

- `sample` 构建失败导致发布失败
  - 原因：`sample` 使用了未在公共仓库可用的第三方依赖。
  - 解决：在 `jitpack.yml` 仅构建并发布库模块，避免 `sample` 参与。

---

## 验证与排查命令速查

- 查看标签构建日志：

```bash
curl -L https://jitpack.io/com/github/Jiabaokang/AgentWeb/v5.1.7-androidx/build.log | tail -n 200
```

- 查看提交构建日志：

```bash
curl -L https://jitpack.io/com/github/Jiabaokang/AgentWeb/<shortSha>/build.log | tail -n 200
```

- 验证 POM / AAR 是否可下载：

```bash
curl -I https://jitpack.io/com/github/Jiabaokang/AgentWeb/agentweb-core/<version>/agentweb-core-<version>.pom
curl -I https://jitpack.io/com/github/Jiabaokang/AgentWeb/agentweb-core/<version>/agentweb-core-<version>.aar
```

---

## 维护建议
- 每次调整发布流程（`build.gradle` 或 `jitpack.yml`）后，建议创建新标签（如：`vX.Y.Z-androidx`）。
- 若需要在 POM 中声明传递依赖，可在 `MavenPublication` 中使用 `pom.withXml { ... }` 添加 `<dependencies>` 节点。
- 保持 `openjdk17` 与兼容的 Gradle/AGP 版本，避免旧版插件的行为差异导致构建时序问题。

---

## 变更记录（与本指南相关）
- 修复发布流程：直接附加 `release` AAR，避免 `components.release` 时序问题。
- 在 `jitpack.yml` 中先 `assembleRelease` 后 `publishToMavenLocal`。
- 验证发布：提交版本 `d516bd2` 已成功生成并可下载。

---

> 如需将当前提交版本切换为最终标签版本，请在 JitPack 标签索引完成后更新下游项目的依赖坐标。