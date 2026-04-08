# AI 代码评审记录

- 评审工程: `super-d-coder/openai-code-check`
- 来源分支: `master`
- 提交备注: auto-run
- 提交SHA: `unknown-`
- 生成时间: 2026-04-08 16:50:32 CST

## 评审结论

### 代码评审意见

**1. 错误提示**
文件 `.github/workflows/main-remote-jar.yml` 中存在配置错误。

**2. 错误位置**
第 77-79 行

**3. 错误代码片段**
```yaml
        env:
          SDK_JAR_URL: https://github.com/<OWNER>/<REPO>/releases/download/<TAG>/openai-code-check-sdk.jar
```

**4. 修改建议**
环境变量 `SDK_JAR_URL` 中包含 `<OWNER>`、`<REPO>`、`<TAG>` 占位符，直接运行会导致 `curl` 下载失败（404 错误）。建议使用 GitHub 上下文变量动态构建 URL，或替换为实际的 Release 地址。

**5. 修改后的代码片段**
```yaml
        env:
          SDK_JAR_URL: https://github.com/${{ github.repository }}/releases/download/v1.0.0/openai-code-check-sdk.jar
```
*(注：请将 `v1.0.0` 替换为您实际的 Release Tag 版本号)*

---

### 总结
1. `pom.xml` 中添加 Lombok `provided` scope 是正确的优化，无问题。
2. `.github/workflows/main-maven-jar.yml` 分支修改无问题。
3. `.github/workflows/main-remote-jar.yml` 中 `SDK_JAR_URL` 包含未替换的占位符，会导致 CI 运行失败，**需要修改**。

## 代码差异

```diff
diff --git a/.github/workflows/main-maven-jar.yml b/.github/workflows/main-maven-jar.yml
index 6b0e847..69192fa 100644
--- a/.github/workflows/main-maven-jar.yml
+++ b/.github/workflows/main-maven-jar.yml
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
diff --git a/.github/workflows/main-remote-jar.yml b/.github/workflows/main-remote-jar.yml
index 69192fa..e071fde 100644
--- a/.github/workflows/main-remote-jar.yml
+++ b/.github/workflows/main-remote-jar.yml
@@ -1,14 +1,14 @@
-name: Build and Run OpenAiCodeCheck By Main Maven Jar
+name: Build and Run OpenAiCodeCheck By Remote Release Jar
 
 on:
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
@@ -70,20 +70,21 @@ jobs:
           ssh-keyscan github.com >> ~/.ssh/known_hosts
           chmod 600 ~/.ssh/known_hosts
 
-      # 打包 SDK 模块（含依赖），供后续 java -jar 直接运行。
-      - name: Build shaded jar
-        run: mvn -B -ntp -pl openai-code-check-sdk -am clean package -DskipTests
-
-      # 定位可运行 jar：排除 original-*，优先取最终产物。
-      - name: Locate shaded jar
+      # 从 GitHub Release 下载 SDK jar。
+      # TODO: 将下面的 URL 替换成你自己的 Release 资产地址。
+      - name: Download SDK jar from Release
         id: jar
+        env:
+          SDK_JAR_URL: https://github.com/<OWNER>/<REPO>/releases/download/<TAG>/openai-code-check-sdk.jar
         run: |
-          JAR_FILE=$(find openai-code-check-sdk/target -maxdepth 1 -name '*.jar' ! -name 'original-*' | head -n 1)
-          if [ -z "$JAR_FILE" ]; then
-            echo "No packaged jar found under openai-code-check-sdk/target" >&2
-            ls -lah openai-code-check-sdk/target >&2 || true
+          mkdir -p .sdk-release
+          JAR_FILE=.sdk-release/openai-code-check-sdk.jar
+          curl -fL "$SDK_JAR_URL" -o "$JAR_FILE"
+          if [ ! -s "$JAR_FILE" ]; then
+            echo "SDK jar download failed or file is empty: $JAR_FILE" >&2
             exit 1
           fi
+          ls -lah .sdk-release
           echo "jar_file=$JAR_FILE" >> "$GITHUB_OUTPUT"
 
       # 运行主程序：
diff --git a/pom.xml b/pom.xml
index 57a2578..39cf071 100644
--- a/pom.xml
+++ b/pom.xml
@@ -34,6 +34,7 @@
                 <groupId>org.projectlombok</groupId>
                 <artifactId>lombok</artifactId>
                 <version>${lombok.version}</version>
+                <scope>provided</scope>
             </dependency>
             <dependency>
                 <groupId>com.google.guava</groupId>
```
