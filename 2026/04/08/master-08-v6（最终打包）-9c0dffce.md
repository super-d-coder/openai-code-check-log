# AI 代码评审记录

- 评审工程: `super-d-coder/openai-code-check-source`
- 来源分支: `master`
- 提交备注: 08-v6（最终打包）
- 提交SHA: `9c0dffce`
- 生成时间: 2026-04-08 17:28:08 CST

## 评审结论

### 代码评审意见

**1. 错误提示**
文件 `.github/workflows/main-maven-jar.yml` 中的分支触发配置有误，将导致 `master` 分支的代码推送和 PR 无法触发该工作流。

**2. 错误位置**
文件：`.github/workflows/main-maven-jar.yml`
行号：7, 11

**3. 错误代码片段**
```yaml
on:
  # push 到 master 时：执行完整流程（自检 + 发布评审结果）。
  push:
    branches:
      - master-close
  # PR 到 master 时：仅执行自检，不发布。
  pull_request:
    branches:
      - master-close
```

**4. 修改建议**
除非项目特意要求仅在 `master-close` 分支运行该工作流，否则应将触发分支恢复为 `master`，以保证主分支的 CI/CD 正常运行。

**5. 修改后的代码片段**
```yaml
on:
  # push 到 master 时：执行完整流程（自检 + 发布评审结果）。
  push:
    branches:
      - master
  # PR 到 master 时：仅执行自检，不发布。
  pull_request:
    branches:
      - master
```

### 总结
1. `main-maven-jar.yml` 中的分支配置被错误地修改为 `master-close`，导致主分支 CI 失效，建议修正回 `master`。
2. `main-remote-jar.yml` 中的 URL 替换和分支配置调整（指向 `master`）看起来是正确的更新。

## 代码差异

```diff
diff --git a/.github/workflows/main-maven-jar.yml b/.github/workflows/main-maven-jar.yml
index 69192fa..6b0e847 100644
--- a/.github/workflows/main-maven-jar.yml
+++ b/.github/workflows/main-maven-jar.yml
@@ -4,11 +4,11 @@ on:
   # push 到 master 时：执行完整流程（自检 + 发布评审结果）。
   push:
     branches:
-      - master
+      - master-close
   # PR 到 master 时：仅执行自检，不发布。
   pull_request:
     branches:
-      - master
+      - master-close
 
 permissions:
   contents: read
diff --git a/.github/workflows/main-remote-jar.yml b/.github/workflows/main-remote-jar.yml
index e071fde..ef895c8 100644
--- a/.github/workflows/main-remote-jar.yml
+++ b/.github/workflows/main-remote-jar.yml
@@ -4,11 +4,11 @@ on:
   # push 到 master 时：执行完整流程（自检 + 发布评审结果）。
   push:
     branches:
-      - master-close
+      - master
   # PR 到 master 时：仅执行自检，不发布。
   pull_request:
     branches:
-      - master-close
+      - master
 
 permissions:
   contents: read
@@ -71,11 +71,10 @@ jobs:
           chmod 600 ~/.ssh/known_hosts
 
       # 从 GitHub Release 下载 SDK jar。
-      # TODO: 将下面的 URL 替换成你自己的 Release 资产地址。
       - name: Download SDK jar from Release
         id: jar
         env:
-          SDK_JAR_URL: https://github.com/<OWNER>/<REPO>/releases/download/<TAG>/openai-code-check-sdk.jar
+          SDK_JAR_URL: https://github.com/super-d-coder/openai-code-check-release/releases/download/v1.0/openai-code-check-sdk-1.0.jar
         run: |
           mkdir -p .sdk-release
           JAR_FILE=.sdk-release/openai-code-check-sdk.jar
```
