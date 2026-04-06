# AI 代码评审记录

- 来源分支: `master`
- 提交备注: 06-v4(回复sha相关)
- 提交SHA: `c54f5f39`
- 生成时间: 2026-04-07 00:22:14 CST

## 评审结论

### 代码评审结果

#### 1. 错误提示：提交了构建产物
*   **错误位置**：`openai-code-check-sdk/target/classes/META-INF/MANIFEST.MF`
*   **错误说明**：`target/` 目录下的文件通常是 Maven 构建生成的产物，不应提交到版本库中。这会增加仓库体积，且不同环境构建可能会导致不必要的冲突。
*   **修改建议**：请将该文件从 Git 追踪中移除，并建议在 `.gitignore` 文件中添加 `/target/` 或 `*.class` 忽略规则。
*   **修改后的代码片段**：（需执行 Git 操作，而非代码修改）
    ```bash
    git rm --cached openai-code-check-sdk/target/classes/META-INF/MANIFEST.MF
    ```

#### 2. 代码逻辑评审
*   **逻辑正确**：新增加的 `AI_REVIEW_COMMIT_SHA` 环境变量获取逻辑、`toShortSha` 方法的截取逻辑以及 `readHeadCommitSha` 的回退机制设计合理，能够保证文件名的唯一性和可追溯性。
*   **格式调整**：`OpenaiCodeCheck.java` 中的注释位置调整更加规范，符合代码可读性要求。

### 总结
代码逻辑实现正确，功能改动合理。唯一的缺陷是误提交了构建产物文件 `MANIFEST.MF`，建议将其移除并配置 `.gitignore`。

## 代码差异

```diff
diff --git a/.github/workflows/main-maven-jar.yml b/.github/workflows/main-maven-jar.yml
index bebbae7..ecd4782 100644
--- a/.github/workflows/main-maven-jar.yml
+++ b/.github/workflows/main-maven-jar.yml
@@ -95,4 +95,5 @@ jobs:
           AI_REVIEW_TARGET_BRANCH: main
           AI_REVIEW_TIME_ZONE: Asia/Shanghai
           AI_REVIEW_COMMIT_NOTE: ${{ github.event.head_commit.message || github.event.pull_request.title || 'auto-run' }}
+          AI_REVIEW_COMMIT_SHA: ${{ github.sha }}
         run: java -jar "${{ steps.jar.outputs.jar_file }}"
diff --git a/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java b/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
index 86e4d8e..7029d2a 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
@@ -48,13 +48,14 @@ public class OpenaiCodeCheck {
 
             //1. 输出代码差异结果
             System.out.println(result.getStdout().isEmpty() ? "(没有检测到代码差异)" : result.getStdout());
+
+            //2. 调用 AI 进行代码评审
             String rules = "以下代码是我为项目新加入的代码,请阅读并对代码进行评审，并给出相应的建议，要求尽量简洁回答\n" +
                     "如果代码有误，需要指出相应错误及位置，如果认为代码应该修改，需要先指出建议，在给出修改后的代码片段，要求修改后的代码片段只包含修改部分，且不需要重复原有代码\n" +
                     "如果代码没有问题，直接回复代码没有问题即可\n" +
                     "如果代码有错误，请给出错误提示，并给出错误位置，并给出错误代码片段，并给出错误代码片段的修改建议，并给出修改后的代码片段，要求修改后的代码片段只包含修改部分，且不需要重复原有代码\n" +
                     "最后需要进行总结，方便快速得到结论\n" +
                     "请注意，以下是代码差异内容：\n";
-            //2. 调用 AI 进行代码评审
             System.out.println("开始进行代码检查...");
             String reviewText = CodeReviewClient.chatClm(rules + result.getStdout());
 
diff --git a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
index 082d815..bc0aaf2 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
@@ -26,7 +26,6 @@ public final class GitCloudRepositoryPublisher {
 
     private static final String REVIEW_TIME_ZONE_ENV = "AI_REVIEW_TIME_ZONE";
     private static final ZoneId DEFAULT_REVIEW_ZONE = ZoneId.of("Asia/Shanghai");
-    private static final DateTimeFormatter TIME_FORMATTER = DateTimeFormatter.ofPattern("HH-mm-ss");
     private static final DateTimeFormatter DISPLAY_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss z");
     // AI 仓库地址环境变量名称
     private static final String AI_REVIEW_REMOTE_URL_ENV = "AI_REVIEW_REMOTE_URL";
@@ -102,7 +101,7 @@ public final class GitCloudRepositoryPublisher {
      * <p>生成规则：</p>
      * <ul>
      *     <li>目录：yyyy/MM/dd</li>
-     *     <li>文件名：分支名-备注-HH-mm-ss.md</li>
+     *     <li>文件名：分支名-备注-shortSha.md</li>
      * </ul>
      *
      * @param remoteUrl    远端仓库地址
@@ -111,6 +110,7 @@ public final class GitCloudRepositoryPublisher {
      * @param commitRemark 提交备注（写入 markdown 与文件名）
      * @param reviewText   AI 评审正文
      * @param codeText     被评审的代码差异
+     * @param commitSha    本次评审对应的提交 SHA（允许传空，内部会兜底）
      * @return 本次评审 markdown 的可访问 URL
      */
     public static String publishReviewMarkdown(String remoteUrl,
@@ -118,7 +118,8 @@ public final class GitCloudRepositoryPublisher {
                                              String sourceBranch,
                                              String commitRemark,
                                              String reviewText,
-                                               String codeText) {
+                                              String codeText,
+                                              String commitSha) {
         requireNotBlank(sourceBranch, "sourceBranch");
         requireNotBlank(commitRemark, "commitRemark");
 
@@ -126,16 +127,18 @@ public final class GitCloudRepositoryPublisher {
         String safeRemark = sanitizeFilePart(commitRemark);
         String safeReview = reviewText == null ? "(无评审内容)" : reviewText;
         String safeCode = codeText == null || codeText.trim().isEmpty() ? "(没有检测到代码差异)" : codeText;
+        String shortSha = toShortSha(commitSha);
 
         ZoneId reviewZone = resolveReviewZoneId();
         ZonedDateTime now = ZonedDateTime.now(reviewZone);
         String folder = String.format("%04d/%02d/%02d", now.getYear(), now.getMonthValue(), now.getDayOfMonth());
-        String fileName = safeBranch + "-" + safeRemark + "-" + TIME_FORMATTER.format(now) + ".md";
+        String fileName = safeBranch + "-" + safeRemark + "-" + shortSha + ".md";
         String relativePath = folder + "/" + fileName;
 
         String markdown = "# AI 代码评审记录\n\n"
                 + "- 来源分支: `" + sourceBranch + "`\n"
                 + "- 提交备注: " + commitRemark + "\n"
+                + "- 提交SHA: `" + shortSha + "`\n"
                 + "- 生成时间: " + DISPLAY_TIME_FORMATTER.format(now) + "\n\n"
                 + "## 评审结论\n\n"
                 + safeReview + "\n\n"
@@ -144,11 +147,22 @@ public final class GitCloudRepositoryPublisher {
                 + safeCode + "\n"
                 + "```\n";
 
-        String pushCommitMessage = "docs: add ai review " + safeBranch + " " + TIME_FORMATTER.format(now);
+        String pushCommitMessage = "docs: add ai review " + safeBranch + " " + shortSha;
         publishSingleFile(remoteUrl, targetBranch, relativePath, markdown, pushCommitMessage);
         return buildGithubFileUrl(remoteUrl, targetBranch, relativePath);
     }
 
+    /**
+     * 将提交 SHA 规范化为短 SHA，便于文件命名与快速定位。
+     */
+    private static String toShortSha(String commitSha) {
+        if (isBlank(commitSha)) {
+            return "unknown-sha";
+        }
+        String normalized = commitSha.trim();
+        return normalized.length() <= 8 ? normalized : normalized.substring(0, 8);
+    }
+
     /**
      * 将仓库地址转换为 GitHub 页面地址，并拼接指定分支与文件路径。
      */
@@ -311,6 +325,12 @@ public final class GitCloudRepositoryPublisher {
             commitRemark = readLatestCommitRemark();
         }
 
+        // 提交 SHA 优先使用 CI 注入，缺省回退到当前仓库 HEAD。
+        String commitSha = System.getenv("AI_REVIEW_COMMIT_SHA");
+        if (isBlank(commitSha)) {
+            commitSha = readHeadCommitSha();
+        }
+
         // 统一交给发布器处理：生成 markdown、按日期归档并推送到目标仓库。
         return GitCloudRepositoryPublisher.publishReviewMarkdown(
                 remoteUrl,
@@ -318,7 +338,8 @@ public final class GitCloudRepositoryPublisher {
                 sourceBranch,
                 commitRemark,
                 reviewText,
-                diffText
+                diffText,
+                commitSha
         );
     }
 
@@ -367,6 +388,17 @@ public final class GitCloudRepositoryPublisher {
         return result.getStdout().trim();
     }
 
+    /**
+     * 读取当前 HEAD 的完整提交 SHA。
+     */
+    private static String readHeadCommitSha() throws Exception {
+        CommandResult result = CommandExecutor.run(List.of("git", "rev-parse", "HEAD"));
+        if (result.getExitCode() != 0 || isBlank(result.getStdout())) {
+            return "unknown-sha";
+        }
+        return result.getStdout().trim();
+    }
+
     /**
      * 判断字符串是否为 null 或仅包含空白字符。
      *
diff --git a/openai-code-check-sdk/target/classes/META-INF/MANIFEST.MF b/openai-code-check-sdk/target/classes/META-INF/MANIFEST.MF
new file mode 100644
index 0000000..29e4dae
--- /dev/null
+++ b/openai-code-check-sdk/target/classes/META-INF/MANIFEST.MF
@@ -0,0 +1,3 @@
+Manifest-Version: 1.0
+Main-Class: com.superd.OpenaiCodeCheck
+
```
