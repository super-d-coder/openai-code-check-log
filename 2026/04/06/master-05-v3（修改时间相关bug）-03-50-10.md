# AI 代码评审记录

- 来源分支: `master`
- 提交备注: 05-v3（修改时间相关bug）
- 生成时间: 2026-04-06 03:50:10 CST

## 评审结论

### 代码评审意见

代码整体逻辑正确，实现了时区配置化及日期目录格式的修正。有一处异常处理建议优化。

### 修改建议

**建议**：在 `resolveReviewZoneId` 方法中，建议捕获具体的 `DateTimeException` 而不是通用的 `Exception`，遵循精准异常处理原则，避免掩盖其他非预期错误。

**错误/建议位置**：`openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java` 第 152 行

**修改后的代码片段**：
```java
        try {
            return ZoneId.of(zoneText.trim());
        } catch (DateTimeException e) {
            System.err.println("无效时区配置 " + REVIEW_TIME_ZONE_ENV + "=" + zoneText + "，已回退为 " + DEFAULT_REVIEW_ZONE);
            return DEFAULT_REVIEW_ZONE;
        }
```

### 总结
代码功能实现无误，已针对异常处理的健壮性提出优化建议，修改后代码更加规范。

## 代码差异

```diff
diff --git a/.github/workflows/main-maven-jar.yml b/.github/workflows/main-maven-jar.yml
index ccfb6e2..918f35b 100644
--- a/.github/workflows/main-maven-jar.yml
+++ b/.github/workflows/main-maven-jar.yml
@@ -93,7 +93,8 @@ jobs:
       # - PR：远端地址为空，仅执行自检
       - name: Run Code Check
         env:
-          AI_REVIEW_REMOTE_URL: ${{ github.event_name == 'push' && vars.AI_REVIEW_REMOTE_URL || '' }}
+          AI_REVIEW_REMOTE_URL: ${{ (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && vars.AI_REVIEW_REMOTE_URL || '' }}
           AI_REVIEW_TARGET_BRANCH: main
+          AI_REVIEW_TIME_ZONE: Asia/Shanghai
           AI_REVIEW_COMMIT_NOTE: ${{ github.event.head_commit.message || github.event.pull_request.title || 'manual-run' }}
         run: java -jar "${{ steps.jar.outputs.jar_file }}"
diff --git a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
index 65f9fd1..cd78a71 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
@@ -5,7 +5,8 @@ import java.io.IOException;
 import java.nio.charset.StandardCharsets;
 import java.nio.file.Files;
 import java.nio.file.Path;
-import java.time.LocalDateTime;
+import java.time.ZoneId;
+import java.time.ZonedDateTime;
 import java.time.format.DateTimeFormatter;
 import java.util.Arrays;
 import java.util.List;
@@ -23,7 +24,10 @@ import java.util.stream.Stream;
  */
 public final class GitCloudRepositoryPublisher {
 
-    private static final DateTimeFormatter TIME_FORMATTER = DateTimeFormatter.ofPattern("HHmmss");
+    private static final String REVIEW_TIME_ZONE_ENV = "AI_REVIEW_TIME_ZONE";
+    private static final ZoneId DEFAULT_REVIEW_ZONE = ZoneId.of("Asia/Shanghai");
+    private static final DateTimeFormatter TIME_FORMATTER = DateTimeFormatter.ofPattern("HH-mm-ss");
+    private static final DateTimeFormatter DISPLAY_TIME_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss z");
 
     private GitCloudRepositoryPublisher() {
     }
@@ -95,8 +99,8 @@ public final class GitCloudRepositoryPublisher {
      *
      * <p>生成规则：</p>
      * <ul>
-     *     <li>目录：yyyy/dd/MM</li>
-     *     <li>文件名：分支名-备注-HHmmss.md</li>
+     *     <li>目录：yyyy/MM/dd</li>
+     *     <li>文件名：分支名-备注-HH-mm-ss.md</li>
      * </ul>
      *
      * @param remoteUrl    远端仓库地址
@@ -120,15 +124,16 @@ public final class GitCloudRepositoryPublisher {
         String safeReview = reviewText == null ? "(无评审内容)" : reviewText;
         String safeCode = codeText == null || codeText.trim().isEmpty() ? "(没有检测到代码差异)" : codeText;
 
-        LocalDateTime now = LocalDateTime.now();
-        String folder = String.format("%04d/%02d/%02d", now.getYear(), now.getDayOfMonth(), now.getMonthValue());
+        ZoneId reviewZone = resolveReviewZoneId();
+        ZonedDateTime now = ZonedDateTime.now(reviewZone);
+        String folder = String.format("%04d/%02d/%02d", now.getYear(), now.getMonthValue(), now.getDayOfMonth());
         String fileName = safeBranch + "-" + safeRemark + "-" + TIME_FORMATTER.format(now) + ".md";
         String relativePath = folder + "/" + fileName;
 
         String markdown = "# AI 代码评审记录\n\n"
                 + "- 来源分支: `" + sourceBranch + "`\n"
                 + "- 提交备注: " + commitRemark + "\n"
-                + "- 生成时间: " + now + "\n\n"
+                + "- 生成时间: " + DISPLAY_TIME_FORMATTER.format(now) + "\n\n"
                 + "## 评审结论\n\n"
                 + safeReview + "\n\n"
                 + "## 代码差异\n\n"
@@ -140,6 +145,19 @@ public final class GitCloudRepositoryPublisher {
         publishSingleFile(remoteUrl, targetBranch, relativePath, markdown, pushCommitMessage);
     }
 
+    private static ZoneId resolveReviewZoneId() {
+        String zoneText = System.getenv(REVIEW_TIME_ZONE_ENV);
+        if (zoneText == null || zoneText.trim().isEmpty()) {
+            return DEFAULT_REVIEW_ZONE;
+        }
+        try {
+            return ZoneId.of(zoneText.trim());
+        } catch (Exception ignored) {
+            System.err.println("无效时区配置 " + REVIEW_TIME_ZONE_ENV + "=" + zoneText + "，已回退为 " + DEFAULT_REVIEW_ZONE);
+            return DEFAULT_REVIEW_ZONE;
+        }
+    }
+
     /**
      * 执行命令并在失败时抛出带上下文的异常。
      *
```
