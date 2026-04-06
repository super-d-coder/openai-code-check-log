# AI 代码评审记录

- 来源分支: `master`
- 提交备注: 06-v3(取消sha相关)
- 生成时间: 2026-04-07 00:14:12 CST

## 评审结论

### 代码评审

**错误提示**：文件命名存在唯一性冲突风险，且丢失了关键追溯信息。

**错误位置**：`GitCloudRepositoryPublisher.java` 类中的 `publishReviewMarkdown` 方法，具体为文件名生成逻辑及 Markdown 内容构建逻辑。

**错误代码片段**：
```java
// 方法签名修改
public static String publishReviewMarkdown(String remoteUrl,
                                         String targetBranch,
                                         String sourceBranch,
                                         String commitRemark,
                                         String reviewText,
                                          String codeText) {
    // ...
    String fileName = safeBranch + "-" + safeRemark + "-" + TIME_FORMATTER.format(now) + ".md";
    // ...
    String markdown = "# AI 代码评审记录\n\n"
            + "- 来源分支: `" + sourceBranch + "`\n"
            + "- 提交备注: " + commitRemark + "\n"
            // 移除了 SHA 信息
            + "- 生成时间: " + DISPLAY_TIME_FORMATTER.format(now) + "\n\n"
            // ...
}
```

**修改建议**：
1.  **唯一性问题**：使用时间戳（精确到秒）作为文件名一部分，在并发或快速提交场景下极易发生文件名冲突，导致旧评审被覆盖。建议恢复使用 `commitSha`（提交SHA）作为唯一标识，因为 SHA 天然唯一且能关联到具体代码版本。
2.  **信息丢失**：评审记录中缺失了 `commitSha`，导致无法快速定位评审对应的代码快照，降低了评审记录的实用价值。建议在 Markdown 内容中恢复 SHA 信息。

**修改后的代码片段**：
```java
    public static String publishReviewMarkdown(String remoteUrl,
                                             String targetBranch,
                                             String sourceBranch,
                                             String commitRemark,
                                             String reviewText,
                                             String codeText,
                                             String commitSha) { // 恢复参数
        requireNotBlank(sourceBranch, "sourceBranch");
        requireNotBlank(commitRemark, "commitRemark");

        String safeBranch = sanitizeFilePart(sourceBranch);
        String safeRemark = sanitizeFilePart(commitRemark);
        String safeReview = reviewText == null ? "(无评审内容)" : reviewText;
        String safeCode = codeText == null || codeText.trim().isEmpty() ? "(没有检测到代码差异)" : codeText;
        String shortSha = toShortSha(commitSha); // 恢复逻辑

        ZoneId reviewZone = resolveReviewZoneId();
        ZonedDateTime now = ZonedDateTime.now(reviewZone);
        String folder = String.format("%04d/%02d/%02d", now.getYear(), now.getMonthValue(), now.getDayOfMonth());
        String fileName = safeBranch + "-" + safeRemark + "-" + shortSha + ".md"; // 恢复逻辑
        String relativePath = folder + "/" + fileName;

        String markdown = "# AI 代码评审记录\n\n"
                + "- 来源分支: `" + sourceBranch + "`\n"
                + "- 提交备注: " + commitRemark + "\n"
                + "- 提交SHA: `" + shortSha + "`\n" // 恢复信息
                + "- 生成时间: " + DISPLAY_TIME_FORMATTER.format(now) + "\n\n"
                + "## 评审结论\n\n"
                + safeReview + "\n\n"
                + "## 代码差异\n\n"
                + "```\n"
                + safeCode + "\n"
                + "```\n";

        String pushCommitMessage = "docs: add ai review " + safeBranch + " " + shortSha; // 恢复逻辑
        publishSingleFile(remoteUrl, targetBranch, relativePath, markdown, pushCommitMessage);
        return buildGithubFileUrl(remoteUrl, targetBranch, relativePath);
    }
```

### 总结
代码移除了 `commitSha` 参数改用时间戳命名，这引入了文件名冲突的风险（秒级精度不足），并导致评审记录缺失关键的代码版本追溯信息。建议恢复原有的 SHA 参数及处理逻辑，以确保文件唯一性和内容完整性。

## 代码差异

```diff
diff --git a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
index bc0aaf2..082d815 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
@@ -26,6 +26,7 @@ public final class GitCloudRepositoryPublisher {
 
     private static final String REVIEW_TIME_ZONE_ENV = "AI_REVIEW_TIME_ZONE";
     private static final ZoneId DEFAULT_REVIEW_ZONE = ZoneId.of("Asia/Shanghai");
+    private static final DateTimeFormatter TIME_FORMATTER = DateTimeFormatter.ofPattern("HH-mm-ss");
     private static final DateTimeFormatter DISPLAY_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss z");
     // AI 仓库地址环境变量名称
     private static final String AI_REVIEW_REMOTE_URL_ENV = "AI_REVIEW_REMOTE_URL";
@@ -101,7 +102,7 @@ public final class GitCloudRepositoryPublisher {
      * <p>生成规则：</p>
      * <ul>
      *     <li>目录：yyyy/MM/dd</li>
-     *     <li>文件名：分支名-备注-shortSha.md</li>
+     *     <li>文件名：分支名-备注-HH-mm-ss.md</li>
      * </ul>
      *
      * @param remoteUrl    远端仓库地址
@@ -110,7 +111,6 @@ public final class GitCloudRepositoryPublisher {
      * @param commitRemark 提交备注（写入 markdown 与文件名）
      * @param reviewText   AI 评审正文
      * @param codeText     被评审的代码差异
-     * @param commitSha    本次评审对应的提交 SHA（允许传空，内部会兜底）
      * @return 本次评审 markdown 的可访问 URL
      */
     public static String publishReviewMarkdown(String remoteUrl,
@@ -118,8 +118,7 @@ public final class GitCloudRepositoryPublisher {
                                              String sourceBranch,
                                              String commitRemark,
                                              String reviewText,
-                                              String codeText,
-                                              String commitSha) {
+                                               String codeText) {
         requireNotBlank(sourceBranch, "sourceBranch");
         requireNotBlank(commitRemark, "commitRemark");
 
@@ -127,18 +126,16 @@ public final class GitCloudRepositoryPublisher {
         String safeRemark = sanitizeFilePart(commitRemark);
         String safeReview = reviewText == null ? "(无评审内容)" : reviewText;
         String safeCode = codeText == null || codeText.trim().isEmpty() ? "(没有检测到代码差异)" : codeText;
-        String shortSha = toShortSha(commitSha);
 
         ZoneId reviewZone = resolveReviewZoneId();
         ZonedDateTime now = ZonedDateTime.now(reviewZone);
         String folder = String.format("%04d/%02d/%02d", now.getYear(), now.getMonthValue(), now.getDayOfMonth());
-        String fileName = safeBranch + "-" + safeRemark + "-" + shortSha + ".md";
+        String fileName = safeBranch + "-" + safeRemark + "-" + TIME_FORMATTER.format(now) + ".md";
         String relativePath = folder + "/" + fileName;
 
         String markdown = "# AI 代码评审记录\n\n"
                 + "- 来源分支: `" + sourceBranch + "`\n"
                 + "- 提交备注: " + commitRemark + "\n"
-                + "- 提交SHA: `" + shortSha + "`\n"
                 + "- 生成时间: " + DISPLAY_TIME_FORMATTER.format(now) + "\n\n"
                 + "## 评审结论\n\n"
                 + safeReview + "\n\n"
@@ -147,22 +144,11 @@ public final class GitCloudRepositoryPublisher {
                 + safeCode + "\n"
                 + "```\n";
 
-        String pushCommitMessage = "docs: add ai review " + safeBranch + " " + shortSha;
+        String pushCommitMessage = "docs: add ai review " + safeBranch + " " + TIME_FORMATTER.format(now);
         publishSingleFile(remoteUrl, targetBranch, relativePath, markdown, pushCommitMessage);
         return buildGithubFileUrl(remoteUrl, targetBranch, relativePath);
     }
 
-    /**
-     * 将提交 SHA 规范化为短 SHA，便于文件命名与快速定位。
-     */
-    private static String toShortSha(String commitSha) {
-        if (isBlank(commitSha)) {
-            return "unknown-sha";
-        }
-        String normalized = commitSha.trim();
-        return normalized.length() <= 8 ? normalized : normalized.substring(0, 8);
-    }
-
     /**
      * 将仓库地址转换为 GitHub 页面地址，并拼接指定分支与文件路径。
      */
@@ -325,12 +311,6 @@ public final class GitCloudRepositoryPublisher {
             commitRemark = readLatestCommitRemark();
         }
 
-        // 提交 SHA 优先使用 CI 注入，缺省回退到当前仓库 HEAD。
-        String commitSha = System.getenv("AI_REVIEW_COMMIT_SHA");
-        if (isBlank(commitSha)) {
-            commitSha = readHeadCommitSha();
-        }
-
         // 统一交给发布器处理：生成 markdown、按日期归档并推送到目标仓库。
         return GitCloudRepositoryPublisher.publishReviewMarkdown(
                 remoteUrl,
@@ -338,8 +318,7 @@ public final class GitCloudRepositoryPublisher {
                 sourceBranch,
                 commitRemark,
                 reviewText,
-                diffText,
-                commitSha
+                diffText
         );
     }
 
@@ -388,17 +367,6 @@ public final class GitCloudRepositoryPublisher {
         return result.getStdout().trim();
     }
 
-    /**
-     * 读取当前 HEAD 的完整提交 SHA。
-     */
-    private static String readHeadCommitSha() throws Exception {
-        CommandResult result = CommandExecutor.run(List.of("git", "rev-parse", "HEAD"));
-        if (result.getExitCode() != 0 || isBlank(result.getStdout())) {
-            return "unknown-sha";
-        }
-        return result.getStdout().trim();
-    }
-
     /**
      * 判断字符串是否为 null 或仅包含空白字符。
      *
```
