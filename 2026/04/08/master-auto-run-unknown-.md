# AI 代码评审记录

- 评审工程: `super-d-coder/openai-code-check`
- 来源分支: `master`
- 提交备注: auto-run
- 提交SHA: `unknown-`
- 生成时间: 2026-04-08 01:16:36 CST

## 评审结论

代码评审结果如下：

### 1. 错误提示
**错误位置**：`GitCommand.java` 类中的 `publishReviewIfConfigured` 方法。

**错误原因**：该方法移除了原有的获取 `commitSha` 和 `commitRemark` 的逻辑，直接使用了常量默认值 `DEFAULT_COMMIT_SHA` 和 `DEFAULT_COMMIT_REMARK`。这会导致生成的评审日志文件名失去唯一性和描述性（例如文件名变为 `main-auto-run-unknown-sha.md`），在同一天内多次运行时极易发生文件覆盖，且无法通过文件名追溯具体的提交信息。

**错误代码片段**：
```java
        // 读取来源分支（失败时用兜底值）
        String sourceBranch = DEFAULT_BRANCH;
        CommandResult branchResult = run(List.of("git", "rev-parse", "--abbrev-ref", "HEAD"));
        if (branchResult.getExitCode() == 0 && !isBlank(branchResult.getStdout())) {
            sourceBranch = branchResult.getStdout().trim();
        }
        //生成并发布 markdown
        return publishReviewMarkdown(
                AI_REVIEW_REMOTE_URL,
                DEFAULT_TARGET_BRANCH,
                sourceBranch,
                DEFAULT_COMMIT_REMARK,
                reviewText,
                diffText,
                DEFAULT_COMMIT_SHA
        );
```

**修改建议**：恢复读取环境变量或 Git 命令来获取真实的提交 SHA 和提交备注的逻辑，仅在获取失败时使用默认值。

**修改后的代码片段**：
```java
        // 读取来源分支（失败时用兜底值）
        String sourceBranch = DEFAULT_BRANCH;
        CommandResult branchResult = run(List.of("git", "rev-parse", "--abbrev-ref", "HEAD"));
        if (branchResult.getExitCode() == 0 && !isBlank(branchResult.getStdout())) {
            sourceBranch = branchResult.getStdout().trim();
        }

        // 读取提交 SHA（优先环境变量，失败时用兜底值）
        String commitSha = System.getenv("AI_REVIEW_COMMIT_SHA");
        if (isBlank(commitSha)) {
            CommandResult shaResult = run(List.of("git", "rev-parse", "HEAD"));
            if (shaResult.getExitCode() == 0 && !isBlank(shaResult.getStdout())) {
                commitSha = shaResult.getStdout().trim();
            } else {
                commitSha = DEFAULT_COMMIT_SHA;
            }
        }

        // 读取提交备注（优先环境变量，失败时用兜底值）
        String commitRemark = System.getenv("AI_REVIEW_COMMIT_NOTE");
        if (isBlank(commitRemark)) {
            CommandResult logResult = run(List.of("git", "log", "-1", "--pretty=%s"));
            if (logResult.getExitCode() == 0 && !isBlank(logResult.getStdout())) {
                commitRemark = logResult.getStdout().trim();
            } else {
                commitRemark = DEFAULT_COMMIT_REMARK;
            }
        }

        //生成并发布 markdown
        return publishReviewMarkdown(
                AI_REVIEW_REMOTE_URL,
                DEFAULT_TARGET_BRANCH,
                sourceBranch,
                commitRemark,
                reviewText,
                diffText,
                commitSha
        );
```

### 总结
本次代码重构进行了大量的方法内联（如 `GLM.java` 和 `Wx.java`），逻辑合并正确，无明显问题。但 `GitCommand.java` 中的 `publishReviewIfConfigured` 方法存在严重逻辑缺失，直接使用常量替代了动态获取的 Git 信息，会导致生成的日志文件名失去意义并可能互相覆盖，必须修复。其他文件修改无问题。

## 代码差异

```diff
diff --git a/openai-code-check-sdk/src/main/java/com/superd/domain/service/impl/OpenaiCodeCheckServiceImpl.java b/openai-code-check-sdk/src/main/java/com/superd/domain/service/impl/OpenaiCodeCheckServiceImpl.java
index 4e5966a..f3cd2b7 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/domain/service/impl/OpenaiCodeCheckServiceImpl.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/domain/service/impl/OpenaiCodeCheckServiceImpl.java
@@ -9,27 +9,18 @@ import com.superd.infrastructure.weixin.Wx;
 import java.util.List;
 
 public class OpenaiCodeCheckServiceImpl implements OpenaiCodeCheckService {
-
-    //weixin
     //公众号 AppId，请替换为你自己的值。
     private final String APP_ID;
     //公众号 AppSecret，请替换为你自己的值。
     private final String APP_SECRET;
-
-    //git
     // AI 仓库地址
     private final String AI_REVIEW_REMOTE_URL;
-
-    //openai
-    // API KEY
+    //openai API KEY
     private final String OPENAI_API_KEY;
-
-    //主函数需要
     // 微信模板消息接收人（openid）
     private final String WX_TO_USER;
     // 微信模板 id
     private final String WX_TEMPLATE_ID;
-
     // 当本次未生成 md URL 时的兜底地址
     private final String DEFAULT_REVIEW_LOG_URL;
 
diff --git a/openai-code-check-sdk/src/main/java/com/superd/infrastructure/git/GitCommand.java b/openai-code-check-sdk/src/main/java/com/superd/infrastructure/git/GitCommand.java
index 4ca9087..d9c5c88 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/infrastructure/git/GitCommand.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/infrastructure/git/GitCommand.java
@@ -28,6 +28,10 @@ public class GitCommand {
     private static final long PROCESS_TIMEOUT_SECONDS = 30;
     // 时区 默认为上海
     private static final ZoneId TIME_ZONE = ZoneId.of("Asia/Shanghai");
+    private static final String DEFAULT_TARGET_BRANCH = "main";
+    private static final String DEFAULT_COMMIT_REMARK = "auto-run";
+    private static final String DEFAULT_COMMIT_SHA = "unknown-sha";
+    private static final String DEFAULT_BRANCH = "unknown-branch";
 
 
     /**
@@ -121,18 +125,23 @@ public class GitCommand {
                                          String relativePath,
                                          String content,
                                          String commitMessage) {
-        validateInput(remoteUrl, branch, relativePath, content, commitMessage);
+        // 1) 参数校验
+        requireNotBlank(remoteUrl, "remoteUrl");
+        requireNotBlank(branch, "branch");
+        requireNotBlank(relativePath, "relativePath");
+        Objects.requireNonNull(content, "content 不能为空");
+        requireNotBlank(commitMessage, "commitMessage");
 
         Path tempDir = null;
         try {
-            // 在临时目录中进行所有 git 操作，避免污染当前工作区。
+            // 2) 在临时目录中执行 git，避免污染当前工作区
             tempDir = Files.createTempDirectory("ai-code-publish-");
             File repoDir = tempDir.resolve("repo").toFile();
 
             runOrThrow(Arrays.asList("git", "clone", remoteUrl, repoDir.getAbsolutePath()), tempDir.toFile(), "克隆远端仓库失败");
             runOrThrow(Arrays.asList("git", "checkout", "-B", branch), repoDir, "切换分支失败");
 
-            // 规范化并校验路径，防止通过 ../ 写出仓库目录。
+            // 3) 写入文件并防止路径越界
             Path targetFile = repoDir.toPath().resolve(relativePath).normalize();
             if (!targetFile.startsWith(repoDir.toPath())) {
                 throw new IllegalArgumentException("relativePath 非法，不能跳出仓库目录: " + relativePath);
@@ -146,7 +155,7 @@ public class GitCommand {
 
             runOrThrow(Arrays.asList("git", "add", relativePath), repoDir, "暂存文件失败");
 
-            // 没有变更时跳过提交和推送，避免无意义失败。
+            // 4) 无变更时直接返回，避免空提交
             CommandResult status = runOrThrow(Arrays.asList("git", "status", "--porcelain"), repoDir, "检查仓库状态失败");
             if (status.getStdout().isEmpty()) {
                 System.out.println("没有检测到文件变更，已跳过提交与推送。");
@@ -160,7 +169,7 @@ public class GitCommand {
         } catch (Exception e) {
             throw new RuntimeException("发布代码到云端仓库失败: " + e.getMessage(), e);
         } finally {
-            // 无论成功失败都清理临时目录，避免 runner/本地残留。
+            // 5) 无论成功失败都清理临时目录
             if (tempDir != null) {
                 deleteRecursively(tempDir);
             }
@@ -203,12 +212,14 @@ public class GitCommand {
         String safeReview = reviewText == null ? "(无评审内容)" : reviewText;
         String safeCode = codeText == null || codeText.trim().isEmpty() ? "(没有检测到代码差异)" : codeText;
         String shortSha = toShortSha(commitSha);
-        String projectName = resolveProjectName();
-
+        String projectName = System.getenv("GITHUB_REPOSITORY");
+        if (isBlank(projectName)) {
+            String dirName = Paths.get(System.getProperty("user.dir")).getFileName().toString();
+            projectName = isBlank(dirName) ? "unknown-project" : dirName;
+        }
 
-        ZoneId reviewZone = TIME_ZONE;// reviewZone = "Asia/Shanghai"
 
-        ZonedDateTime now = ZonedDateTime.now(reviewZone);
+        ZonedDateTime now = ZonedDateTime.now(TIME_ZONE);
         String folder = String.format("%04d/%02d/%02d", now.getYear(), now.getMonthValue(), now.getDayOfMonth());
         String fileName = safeBranch + "-" + safeRemark + "-" + shortSha + ".md";
         String relativePath = folder + "/" + fileName;
@@ -280,30 +291,6 @@ public class GitCommand {
         return result;
     }
 
-    /**
-     * 校验发布单文件所需参数。
-     */
-    public void validateInput(String remoteUrl,
-                                      String branch,
-                                      String relativePath,
-                                      String content,
-                                      String commitMessage) {
-        requireNotBlank(remoteUrl, "remoteUrl");
-        requireNotBlank(branch, "branch");
-        requireNotBlank(relativePath, "relativePath");
-        Objects.requireNonNull(content, "content 不能为空");
-        requireNotBlank(commitMessage, "commitMessage");
-    }
-
-    /**
-     * 校验字符串必须为非空白。
-     */
-    private static void requireNotBlank(String value, String fieldName) {
-        if (value == null || value.trim().isEmpty()) {
-            throw new IllegalArgumentException(fieldName + " 不能为空");
-        }
-    }
-
     /**
      * 清理文件名中的非法字符并限制长度，避免跨平台路径问题。
      */
@@ -361,112 +348,35 @@ public class GitCommand {
      * @throws Exception 读取 git 信息或执行发布流程时抛出的异常
      */
     public String publishReviewIfConfigured(String reviewText, String diffText) throws Exception {
-        // PR 阶段只做检查，不向日志仓库写入任何文件。
-        if (isPullRequestEvent()) {
+        //PR 事件仅做自检，不发布
+        String eventName = System.getenv("GITHUB_EVENT_NAME");
+        if ("pull_request".equalsIgnoreCase(eventName)) {
             System.out.println("当前是 pull_request 事件，仅执行代码自检，不发布评审结果。");
             return "";
         }
 
-        // 默认写入 main。
-        String targetBranch = "main";
-
-        // 来源分支用于构建 markdown 文件名，便于回溯评审来源。
-        String sourceBranch = readCurrentBranch();
-
-        // 备注优先使用外部注入（如 workflow 里的提交信息），否则回退到本地 git 最近一次提交标题。
-        String commitRemark = System.getenv("AI_REVIEW_COMMIT_NOTE");
-        if (isBlank(commitRemark)) {
-            commitRemark = readLatestCommitRemark();
-        }
-
-        // 提交 SHA 优先使用 CI 注入，缺省回退到当前仓库 HEAD。
-        String commitSha = System.getenv("AI_REVIEW_COMMIT_SHA");
-        if (isBlank(commitSha)) {
-            commitSha = readHeadCommitSha();
+        // 读取来源分支（失败时用兜底值）
+        String sourceBranch = DEFAULT_BRANCH;
+        CommandResult branchResult = run(List.of("git", "rev-parse", "--abbrev-ref", "HEAD"));
+        if (branchResult.getExitCode() == 0 && !isBlank(branchResult.getStdout())) {
+            sourceBranch = branchResult.getStdout().trim();
         }
-
-        // 统一交给发布器处理：生成 markdown、按日期归档并推送到目标仓库。
+        //生成并发布 markdown
         return publishReviewMarkdown(
                 AI_REVIEW_REMOTE_URL,
-                targetBranch,
+                DEFAULT_TARGET_BRANCH,
                 sourceBranch,
-                commitRemark,
+                DEFAULT_COMMIT_REMARK,
                 reviewText,
                 diffText,
-                commitSha
+                DEFAULT_COMMIT_SHA
         );
     }
 
-    /**
-     * 判断当前运行是否由 GitHub pull_request 事件触发。
-     *
-     * <p>当返回 true 时，上层会跳过发布，仅执行代码自检，避免在 PR 阶段写入日志仓库。</p>
-     *
-     * @return true 表示当前事件为 pull_request
-     */
-    public boolean isPullRequestEvent() {
-        String eventName = System.getenv("GITHUB_EVENT_NAME");
-        return "pull_request".equalsIgnoreCase(eventName);
-    }
-
-    /**
-     * 读取当前仓库分支名。
-     *
-     * <p>内部通过 {@code git rev-parse --abbrev-ref HEAD} 获取，失败时返回兜底值，
-     * 以保证发布流程仍可继续。</p>
-     *
-     * @return 当前分支名，失败时返回 {@code unknown-branch}
-     * @throws Exception git 命令执行异常
-     */
-    public String readCurrentBranch() throws Exception {
-        CommandResult result = run(List.of("git", "rev-parse", "--abbrev-ref", "HEAD"));
-        if (result.getExitCode() != 0 || isBlank(result.getStdout())) {
-            return "unknown-branch";
-        }
-        return result.getStdout().trim();
-    }
-
-    /**
-     * 读取最近一次提交标题，作为评审 markdown 的备注信息兜底值。
-     *
-     * <p>内部通过 {@code git log -1 --pretty=%s} 获取，失败时返回兜底值。</p>
-     *
-     * @return 最近一次提交标题，失败时返回 {@code no-commit-message}
-     * @throws Exception git 命令执行异常
-     */
-    public String readLatestCommitRemark() throws Exception {
-        CommandResult result = run(List.of("git", "log", "-1", "--pretty=%s"));
-        if (result.getExitCode() != 0 || isBlank(result.getStdout())) {
-            return "no-commit-message";
-        }
-        return result.getStdout().trim();
-    }
-
-    /**
-     * 读取当前 HEAD 的完整提交 SHA。
-     */
-    public String readHeadCommitSha() throws Exception {
-        CommandResult result = run(List.of("git", "rev-parse", "HEAD"));
-        if (result.getExitCode() != 0 || isBlank(result.getStdout())) {
-            return "unknown-sha";
-        }
-        return result.getStdout().trim();
-    }
-
-    /**
-     * 解析本次评审对应的工程标识。
-     *
-     * <p>优先读取 GitHub Actions 注入的 {@code GITHUB_REPOSITORY}（owner/repo），
-     * 本地运行时回退为当前工作目录名。</p>
-     */
-    private String resolveProjectName() {
-        String repository = System.getenv("GITHUB_REPOSITORY");
-        if (!isBlank(repository)) {
-            return repository.trim();
+    private static void requireNotBlank(String value, String fieldName) {
+        if (value == null || value.trim().isEmpty()) {
+            throw new IllegalArgumentException(fieldName + " 不能为空");
         }
-
-        String dirName = Paths.get(System.getProperty("user.dir")).getFileName().toString();
-        return isBlank(dirName) ? "unknown-project" : dirName;
     }
 
     /**
diff --git a/openai-code-check-sdk/src/main/java/com/superd/infrastructure/openai/impl/GLM.java b/openai-code-check-sdk/src/main/java/com/superd/infrastructure/openai/impl/GLM.java
index eeb5db2..c6011fa 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/infrastructure/openai/impl/GLM.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/infrastructure/openai/impl/GLM.java
@@ -38,12 +38,21 @@ public class GLM implements OpenAi {
      */
     @Override
     public String openAiChat(String content) {
-        String safeContent = truncateIfNeeded(content);
+        // 1) 先做长度保护，避免超长请求导致服务端报错
+        String safeContent;
+        if (content == null) {
+            safeContent = "";
+        } else if (content.length() <= MAX_CONTENT_LENGTH) {
+            safeContent = content;
+        } else {
+            System.err.println("提示：代码 diff 过长，已截断后再请求模型。原长度=" + content.length());
+            safeContent = content.substring(0, MAX_CONTENT_LENGTH);
+        }
 
         RuntimeException lastError = null;
         for (int attempt = 0; attempt <= MAX_RETRY; attempt++) {
             try {
-                // 初始化客户端
+                // 2) 初始化客户端并组装请求
                 ZhipuAiClient client = ZhipuAiClient.builder().ofZHIPU()
                         .apiKey(API_KEY)
                         .build();
@@ -62,20 +71,70 @@ public class GLM implements OpenAi {
                         .maxTokens(MAX_TOKEN)
                         .build();
 
-                // 发送请求
+                // 3) 调用模型并解析返回
                 ChatCompletionResponse response = client.chat().createChatCompletion(request);
+                if (response == null || response.getData() == null) {
+                    System.out.println("模型返回数据为空。\n");
+                    return "";
+                }
+
+                var choices = response.getData().getChoices();
+                if (choices == null || choices.isEmpty()) {
+                    System.out.println("模型已返回结果，但没有可用的回答选项。\n");
+                    return "";
+                }
+
+                var choice = choices.get(0);
+                String answerContent = null;
+                if (choice.getMessage() != null && choice.getMessage().getContent() != null) {
+                    answerContent = String.valueOf(choice.getMessage().getContent());
+                }
 
-                return printResponse(response);
+                if (answerContent == null || answerContent.trim().isEmpty()) {
+                    System.out.println("模型已返回结果，但正文为空。");
+                } else {
+                    System.out.println(answerContent);
+                }
+
+                Object finishReasonObj = choice.getFinishReason();
+                String finishReason = finishReasonObj == null ? null : String.valueOf(finishReasonObj);
+                if (finishReason != null && !finishReason.trim().isEmpty()) {
+                    System.out.println("finishReason=" + finishReason);
+                    if ("length".equalsIgnoreCase(finishReason.trim())) {
+                        System.out.println("提示：模型输出可能被长度限制截断。");
+                    }
+                }
+                return answerContent;
             } catch (Exception e) {
+                // 4) 失败时按可重试规则处理
                 String errorMessage = e.getMessage() == null ? "(无错误详情)" : e.getMessage();
                 lastError = new RuntimeException("调用大模型失败，第 " + (attempt + 1) + " 次：" + errorMessage, e);
 
-                if (!isRetriable(e) || attempt == MAX_RETRY) {
+                boolean retriable = false;
+                Throwable current = e;
+                while (current != null) {
+                    if (current instanceof SocketTimeoutException) {
+                        retriable = true;
+                        break;
+                    }
+                    current = current.getCause();
+                }
+                if (!retriable) {
+                    String message = e.getMessage();
+                    retriable = message != null
+                            && (message.contains("HTTP 500") || message.contains("Network error") || message.contains("timeout"));
+                }
+
+                if (!retriable || attempt == MAX_RETRY) {
                     throw lastError;
                 }
 
                 System.err.println("模型服务暂时不可用，正在重试...");
-                sleepQuietly(2000L);
+                try {
+                    Thread.sleep(2000L);
+                } catch (InterruptedException ie) {
+                    Thread.currentThread().interrupt();
+                }
             }
         }
 
@@ -84,119 +143,4 @@ public class GLM implements OpenAi {
         }
         return "";
     }
-
-    /**
-     * 解析并打印模型返回内容，同时返回正文字符串供后续流程使用。
-     *
-     * <p>该方法会做基础空值保护，避免响应结构异常导致空指针。</p>
-     *
-     * @param response 模型响应对象
-     * @return 模型正文内容；当正文缺失时返回空字符串
-     */
-    public String printResponse(ChatCompletionResponse response) {
-        if (response == null || response.getData() == null) {
-            System.out.println("模型返回数据为空。\n");
-            return "";
-        }
-
-        var choices = response.getData().getChoices();
-        if (choices == null || choices.isEmpty()) {
-            System.out.println("模型已返回结果，但没有可用的回答选项。\n");
-            return "";
-        }
-
-        var choice = choices.get(0);
-        Object messageObj = choice.getMessage();
-        String answerContent = null;
-        if (messageObj != null) {
-            Object contentObj = choice.getMessage().getContent();
-            answerContent = contentObj == null ? null : String.valueOf(contentObj);
-        }
-
-        if (answerContent == null || answerContent.trim().isEmpty()) {
-            System.out.println("模型已返回结果，但正文为空。");
-        } else {
-            System.out.println(answerContent);
-        }
-
-        // finishReason 在当前 SDK 中可直接读取，无需反射。
-        Object finishReasonObj = choice.getFinishReason();
-        String finishReason = finishReasonObj == null ? null : String.valueOf(finishReasonObj);
-        if (finishReason != null && !finishReason.trim().isEmpty()) {
-            System.out.println("finishReason=" + finishReason);
-            if ("length".equalsIgnoreCase(finishReason.trim())) {
-                System.out.println("提示：模型输出可能被长度限制截断。");
-            }
-        }
-        return answerContent;
-    }
-
-    /**
-     * 对输入文本进行长度截断，避免超长 diff 触发服务端限制。
-     *
-     * @param content 原始请求内容
-     * @return 截断后的安全内容；入参为 null 时返回空字符串
-     */
-    private String truncateIfNeeded(String content) {
-        if (content == null) {
-            return "";
-        }
-        if (content.length() <= MAX_CONTENT_LENGTH) {
-            return content;
-        }
-        System.err.println("提示：代码 diff 过长，已截断后再请求模型。原长度=" + content.length());
-        return content.substring(0, MAX_CONTENT_LENGTH);
-    }
-
-    /**
-     * 判断异常是否属于可重试类型。
-     *
-     * @param e 调用过程中捕获的异常
-     * @return true 表示建议重试；false 表示直接失败
-     */
-    private boolean isRetriable(Exception e) {
-        if (e == null) {
-            return false;
-        }
-        if (hasCause(e, SocketTimeoutException.class)) {
-            return true;
-        }
-
-        String message = e.getMessage();
-        if (message == null) {
-            return false;
-        }
-        return message.contains("HTTP 500") || message.contains("Network error") || message.contains("timeout");
-    }
-
-    /**
-     * 判断异常链中是否包含指定类型的根因或中间 cause。
-     *
-     * @param throwable 待检查异常
-     * @param type 目标异常类型
-     * @return true 表示异常链中存在该类型
-     */
-    private boolean hasCause(Throwable throwable, Class<? extends Throwable> type) {
-        Throwable current = throwable;
-        while (current != null) {
-            if (type.isInstance(current)) {
-                return true;
-            }
-            current = current.getCause();
-        }
-        return false;
-    }
-
-    /**
-     * 安静休眠：中断时仅恢复中断标记，不向外抛异常。
-     *
-     * @param millis 休眠毫秒数
-     */
-    private void sleepQuietly(long millis) {
-        try {
-            Thread.sleep(millis);
-        } catch (InterruptedException ie) {
-            Thread.currentThread().interrupt();
-        }
-    }
 }
diff --git a/openai-code-check-sdk/src/main/java/com/superd/infrastructure/weixin/Wx.java b/openai-code-check-sdk/src/main/java/com/superd/infrastructure/weixin/Wx.java
index 4004736..1613fd4 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/infrastructure/weixin/Wx.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/infrastructure/weixin/Wx.java
@@ -53,12 +53,16 @@ public class Wx {
 	 */
 	public String getAccessToken() throws IOException, InterruptedException {
 		return executeWithRetry(() -> {
-			String requestBody = buildRequestBody();
+			JSONObject requestBody = new JSONObject();
+			requestBody.put("grant_type", GRANT_TYPE);
+			requestBody.put("appid", APP_ID);
+			requestBody.put("secret", APP_SECRET);
+			requestBody.put("force_refresh", FORCE_REFRESH);
 			String responseBody = Url.sendRequest(
 					ACCESS_TOKEN_URL,
 					HTTP_METHOD_POST,
 					Map.of("Content-Type", "application/json;charset=UTF-8"),
-					requestBody
+					requestBody.toJSONString()
 			);
 
 			if (responseBody == null || responseBody.trim().isEmpty()) {
@@ -161,15 +165,19 @@ public class Wx {
 	 * @throws InterruptedException 线程在等待响应时被中断时抛出
 	 */
 	public String sendReviewTemplateMessage(String touser,
-											 String templateId,
-											 String reviewMdUrl,
-											 String defaultReviewLogUrl) throws IOException, InterruptedException {
+										 String templateId,
+										 String reviewMdUrl,
+										 String defaultReviewLogUrl) throws IOException, InterruptedException {
 		String messageUrl = isBlank(reviewMdUrl) ? defaultReviewLogUrl : reviewMdUrl;
 		String reviewValue = isBlank(reviewMdUrl) ? "(本次未生成md文件URL)" : reviewMdUrl;
+		String projectName = Paths.get(System.getProperty("user.dir")).getFileName().toString();
+		if (isBlank(projectName)) {
+			projectName = "unknown-project";
+		}
 
 		JSONObject data = new JSONObject();
 		JSONObject project = new JSONObject();
-		project.put("value", resolveProjectName());
+		project.put("value", projectName);
 		JSONObject review = new JSONObject();
 		review.put("value", reviewValue);
 		data.put("project", project);
@@ -178,30 +186,10 @@ public class Wx {
 		return sendTemplateMessage(touser, templateId, messageUrl, data);
 	}
 
-	/**
-	 * 构建获取 access_token 所需的 JSON 请求体。
-	 */
-	private String buildRequestBody() {
-		JSONObject body = new JSONObject();
-		body.put("grant_type", GRANT_TYPE);
-		body.put("appid", APP_ID);
-		body.put("secret", APP_SECRET);
-		body.put("force_refresh", FORCE_REFRESH);
-		return body.toJSONString();
-	}
-
-	/**
-	 * 解析当前项目名称，默认取当前工作目录名称。
-	 */
-	private String resolveProjectName() {
-		String projectName = Paths.get(System.getProperty("user.dir")).getFileName().toString();
-		return isBlank(projectName) ? "unknown-project" : projectName;
-	}
-
 	/**
 	 * 判断字符串是否为空白。
 	 */
-	public boolean isBlank(String value) {
+	private static boolean isBlank(String value) {
 		return value == null || value.trim().isEmpty();
 	}
```
