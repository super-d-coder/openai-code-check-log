# AI 代码评审记录

- 评审工程: `super-d-coder/openai-code-check`
- 来源分支: `master`
- 提交备注: auto-run
- 提交SHA: `unknown-`
- 生成时间: 2026-04-08 16:31:04 CST

## 评审结论

### 代码评审意见

**1. 错误：目标分支配置不一致**
*   **错误位置**：`.github/workflows/main-remote-jar.yml` 第 100 行
*   **错误提示**：工作流触发分支为 `master`，但环境变量 `AI_REVIEW_TARGET_BRANCH` 设置为 `main`，这会导致代码审查工具针对错误的目标分支进行比对。
*   **修改建议**：将目标分支修改为 `master` 或使用变量动态获取当前分支。
*   **修改后的代码片段**：
    ```yaml
          AI_REVIEW_TARGET_BRANCH: master
    ```

**2. 错误：工作流名称重复**
*   **错误位置**：`.github/workflows/main-remote-jar.yml` 第 1 行
*   **错误提示**：新文件的 `name` 字段与 `.github/workflows/main-maven-jar.yml` 完全一致，在 GitHub Actions 界面中无法区分是哪个工作流在运行。
*   **修改建议**：修改名称以明确该工作流针对 master 分支。
*   **修改后的代码片段**：
    ```yaml
    name: Build and Run OpenAiCodeCheck By Master Branch
    ```

**3. 建议：文件命名与内容不符**
*   **建议位置**：`.github/workflows/main-remote-jar.yml` 文件名
*   **建议说明**：文件名包含 `remote-jar`，通常暗示会下载远程 Jar 包运行，但实际内容却是执行 `mvn clean package` 进行本地构建。建议将文件重命名为 `main-maven-jar-master.yml` 以避免误导。

### 总结
代码存在配置错误（目标分支不一致）和命名问题（工作流名称重复、文件名误导），建议修正环境变量配置并重命名工作流名称及文件名，以确保逻辑正确且易于维护。

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
new file mode 100644
index 0000000..69192fa
--- /dev/null
+++ b/.github/workflows/main-remote-jar.yml
@@ -0,0 +1,105 @@
+name: Build and Run OpenAiCodeCheck By Main Maven Jar
+
+on:
+  # push 到 master 时：执行完整流程（自检 + 发布评审结果）。
+  push:
+    branches:
+      - master
+  # PR 到 master 时：仅执行自检，不发布。
+  pull_request:
+    branches:
+      - master
+
+permissions:
+  contents: read
+
+jobs:
+  build:
+    runs-on: ubuntu-latest
+
+    steps:
+      # 拉取仓库代码。fetch-depth=2 便于本项目的 diff 逻辑读取最近提交。
+      - name: Checkout repository
+        uses: actions/checkout@v4
+        with:
+          fetch-depth: 2
+
+      # 安装并缓存 JDK 11 与 Maven 依赖。
+      - name: Set up JDK 11
+        uses: actions/setup-java@v4
+        with:
+          distribution: temurin
+          java-version: '11'
+          cache: maven
+
+      # 配置 git 身份，供后续可能的 commit/push 使用。
+      - name: Configure Git identity
+        run: |
+          git config --global user.name "github-actions[bot]"
+          git config --global user.email "github-actions[bot]@users.noreply.github.com"
+
+      # 仅 push 事件需要写日志仓库，因此只在 push 时准备 SSH。
+      - name: Prepare SSH for pushing review results
+        if: github.event_name == 'push'
+        env:
+          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
+          SSH_PASSPHRASE: ${{ secrets.SSH_PASSPHRASE }}
+        run: |
+          mkdir -p ~/.ssh
+          chmod 700 ~/.ssh
+
+          # 启动 agent，并将 socket 信息写入 GITHUB_ENV，供后续步骤继承。
+          eval "$(ssh-agent -s)"
+          echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK" >> "$GITHUB_ENV"
+          echo "SSH_AGENT_PID=$SSH_AGENT_PID" >> "$GITHUB_ENV"
+
+          # 写入私钥文件并收紧权限。
+          printf '%s\n' "$SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
+          chmod 600 ~/.ssh/id_ed25519
+
+          # 如果私钥有 passphrase，则通过 SSH_ASKPASS 非交互解锁。
+          if [ -n "$SSH_PASSPHRASE" ]; then
+            printf '#!/bin/sh\necho "%s"\n' "$SSH_PASSPHRASE" > /tmp/ssh-askpass.sh
+            chmod +x /tmp/ssh-askpass.sh
+            DISPLAY=:0 SSH_ASKPASS=/tmp/ssh-askpass.sh SSH_ASKPASS_REQUIRE=force setsid -w ssh-add ~/.ssh/id_ed25519 </dev/null
+          else
+            ssh-add ~/.ssh/id_ed25519
+          fi
+
+          # 预写入 github.com 主机指纹，避免首次连接交互确认。
+          ssh-keyscan github.com >> ~/.ssh/known_hosts
+          chmod 600 ~/.ssh/known_hosts
+
+      # 打包 SDK 模块（含依赖），供后续 java -jar 直接运行。
+      - name: Build shaded jar
+        run: mvn -B -ntp -pl openai-code-check-sdk -am clean package -DskipTests
+
+      # 定位可运行 jar：排除 original-*，优先取最终产物。
+      - name: Locate shaded jar
+        id: jar
+        run: |
+          JAR_FILE=$(find openai-code-check-sdk/target -maxdepth 1 -name '*.jar' ! -name 'original-*' | head -n 1)
+          if [ -z "$JAR_FILE" ]; then
+            echo "No packaged jar found under openai-code-check-sdk/target" >&2
+            ls -lah openai-code-check-sdk/target >&2 || true
+            exit 1
+          fi
+          echo "jar_file=$JAR_FILE" >> "$GITHUB_OUTPUT"
+
+      # 运行主程序：
+      # - push：注入远端日志仓库地址，允许发布
+      # - PR：远端地址为空，仅执行自检
+      - name: Run Code Check
+        env:
+          AI_REVIEW_REMOTE_URL: ${{ github.event_name == 'push' && secrets.AI_REVIEW_REMOTE_URL || '' }}
+          WECHAT_APP_ID: ${{ secrets.WECHAT_APP_ID }}
+          WECHAT_APP_SECRET: ${{ secrets.WECHAT_APP_SECRET }}
+          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
+          WX_TO_USER: ${{ secrets.WX_TO_USER }}
+          WX_TEMPLATE_ID: ${{ secrets.WX_TEMPLATE_ID }}
+          DEFAULT_REVIEW_LOG_URL: ${{ secrets.DEFAULT_REVIEW_LOG_URL}}
+          AI_REVIEW_TARGET_BRANCH: main
+          AI_REVIEW_TIME_ZONE: Asia/Shanghai
+          AI_REVIEW_COMMIT_NOTE: ${{ github.event.head_commit.message || github.event.pull_request.title || 'auto-run' }}
+          AI_REVIEW_COMMIT_SHA: ${{ github.sha }}
+        run: java -jar "${{ steps.jar.outputs.jar_file }}"
diff --git a/openai-code-check-sdk/src/main/java/com/superd/domain/model/EnvKeys.java b/openai-code-check-sdk/src/main/java/com/superd/domain/model/EnvKeys.java
index e456039..03c6b95 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/domain/model/EnvKeys.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/domain/model/EnvKeys.java
@@ -12,8 +12,8 @@ public final class EnvKeys {
     public static final String WX_TO_USER = "WX_TO_USER";
     //微信模板 id
     public static final String WX_TEMPLATE_ID = "WX_TEMPLATE_ID";
-    //日志记录仓库
+    //日志记录仓库（SHA）
     public static final String AI_REVIEW_REMOTE_URL = "AI_REVIEW_REMOTE_URL";
-    // 当本次未生成 md URL 时的兜底地址
+    // 当本次未生成 md URL 时的兜底地址（URL）
     public static final String DEFAULT_REVIEW_LOG_URL = "DEFAULT_REVIEW_LOG_URL";
 }
```
