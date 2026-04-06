# AI 代码评审记录

- 来源分支: `master`
- 提交备注: 06-v2(修复了联网相关bug)
- 提交SHA: `13d4a027`
- 生成时间: 2026-04-06 23:57:45 CST

## 评审结论

### 代码评审意见

#### 1. 错误：敏感信息硬编码风险
*   **错误位置**: `openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java` 第 37-43 行
*   **错误提示**: 代码中硬编码了真实的微信公众号 `APP_ID` 和 `APP_SECRET` 作为默认值。将敏感凭证提交到代码仓库存在严重的安全隐患，可能导致账号泄露风险。
*   **修改建议**: 移除硬编码的默认值，改为从环境变量强制读取。若环境变量未配置，应抛出异常或设为空值，避免泄露。
*   **修改后的代码片段**:
    ```java
    	/**
    	 * 公众号 AppId，请替换为你自己的值。
    	 */
    	private static final String APP_ID = resolveEnvOrDefault("WX_APP_ID", "");

    	/**
    	 * 公众号 AppSecret，请替换为你自己的值。
    	 */
    	private static final String APP_SECRET = resolveEnvOrDefault("WX_APP_SECRET", "");
    ```

#### 2. 建议：保留手动触发功能
*   **建议位置**: `.github/workflows/main-maven-jar.yml` 第 11-12 行
*   **建议说明**: 删除了 `workflow_dispatch` 触发器会导致无法在 GitHub Actions 界面手动触发工作流，这降低了排查问题和回归测试的灵活性，建议恢复。
*   **修改后的代码片段**:
    ```yaml
      pull_request:
        branches:
          - master
      # 支持手动触发，便于排查与回归测试。
      workflow_dispatch:
    ```

#### 3. 建议：优化日志输出方式
*   **建议位置**: `openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java` 第 66 行
*   **建议说明**: 使用 `System.out.println` 打印日志不利于日志级别的管理和格式统一，建议使用日志框架（如 Slf4j）。
*   **修改后的代码片段**:
    ```java
            // 假设类中已定义 Logger，如 private static final Logger log = LoggerFactory.getLogger(OpenaiCodeCheck.class);
            log.info("msgId: {}", msgId);
    ```

### 总结
代码主要存在**敏感信息硬编码**的安全问题，必须修复；同时建议恢复 CI 工作流的**手动触发功能**以提升维护便利性；日志打印方式建议标准化处理。其他逻辑改动（如文件名使用 SHA、重试机制、代码清理）均合理。

## 代码差异

```diff
diff --git a/.github/workflows/main-maven-jar.yml b/.github/workflows/main-maven-jar.yml
index 918f35b..bebbae7 100644
--- a/.github/workflows/main-maven-jar.yml
+++ b/.github/workflows/main-maven-jar.yml
@@ -9,8 +9,6 @@ on:
   pull_request:
     branches:
       - master
-  # 支持手动触发，便于排查与回归测试。
-  workflow_dispatch:
 
 permissions:
   contents: read
@@ -93,8 +91,8 @@ jobs:
       # - PR：远端地址为空，仅执行自检
       - name: Run Code Check
         env:
-          AI_REVIEW_REMOTE_URL: ${{ (github.event_name == 'push' || github.event_name == 'workflow_dispatch') && vars.AI_REVIEW_REMOTE_URL || '' }}
+          AI_REVIEW_REMOTE_URL: ${{ github.event_name == 'push' && vars.AI_REVIEW_REMOTE_URL || '' }}
           AI_REVIEW_TARGET_BRANCH: main
           AI_REVIEW_TIME_ZONE: Asia/Shanghai
-          AI_REVIEW_COMMIT_NOTE: ${{ github.event.head_commit.message || github.event.pull_request.title || 'manual-run' }}
+          AI_REVIEW_COMMIT_NOTE: ${{ github.event.head_commit.message || github.event.pull_request.title || 'auto-run' }}
         run: java -jar "${{ steps.jar.outputs.jar_file }}"
diff --git a/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java b/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
index 612bf3f..86e4d8e 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
@@ -1,6 +1,5 @@
 package com.superd;
 
-import com.alibaba.fastjson2.JSONObject;
 import com.superd.ai.CodeReviewClient;
 import com.superd.git.CommandExecutor;
 import com.superd.git.CommandResult;
@@ -8,7 +7,6 @@ import com.superd.git.GitCloudRepositoryPublisher;
 import com.superd.git.GitDiffCommandBuilder;
 import com.superd.wx.Wx;
 
-import java.nio.file.Paths;
 import java.util.List;
 
 public class OpenaiCodeCheck {
@@ -65,6 +63,7 @@ public class OpenaiCodeCheck {
 
             //4.向微信推送评审结果，并提供链接到对应md文件
             String msgId = Wx.sendReviewTemplateMessage(WX_TO_USER, WX_TEMPLATE_ID, reviewMdUrl, DEFAULT_REVIEW_LOG_URL);
+            System.out.println("msgId: " + msgId);
 
         } catch (IllegalArgumentException e) {
             System.err.println("参数错误：" + e.getMessage());
@@ -98,23 +97,5 @@ public class OpenaiCodeCheck {
         return current;
     }
 
-    /**
-     * 判断字符串是否为 null 或仅包含空白字符。
-     */
-    private static boolean isBlank(String value) {
-        return value == null || value.trim().isEmpty();
-    }
-
-    /**
-     * 动态解析项目名称：环境变量优先，其次使用当前仓库目录名。
-     */
-    private static String resolveProjectName() throws Exception {
-        CommandResult result = CommandExecutor.run(List.of("git", "rev-parse", "--show-toplevel"));
-        if (result.getExitCode() == 0 && !isBlank(result.getStdout())) {
-            return Paths.get(result.getStdout().trim()).getFileName().toString();
-        }
-        return "unknown-project";
-    }
-
 
 }
diff --git a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
index ea99ec9..bc0aaf2 100644
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
-                                             String codeText) {
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
diff --git a/openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java b/openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java
index 4e533e1..8f9c1e8 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java
@@ -5,6 +5,7 @@ import com.alibaba.fastjson2.JSONObject;
 import com.superd.utils.Url;
 
 import java.io.IOException;
+import java.net.http.HttpConnectTimeoutException;
 import java.nio.file.Paths;
 import java.util.Map;
 
@@ -33,12 +34,12 @@ public class Wx {
 	/**
 	 * 公众号 AppId，请替换为你自己的值。
 	 */
-	private static final String APP_ID = "wx2d160f15c419f7b3";
+	private static final String APP_ID = resolveEnvOrDefault("WX_APP_ID", "wx2d160f15c419f7b3");
 
 	/**
 	 * 公众号 AppSecret，请替换为你自己的值。
 	 */
-	private static final String APP_SECRET = "e5a76e38af434f706a20389fa6eca496";
+	private static final String APP_SECRET = resolveEnvOrDefault("WX_APP_SECRET", "e5a76e38af434f706a20389fa6eca496");
 
 	/**
 	 * 是否强制刷新 token。
@@ -50,6 +51,16 @@ public class Wx {
 	 */
 	private static final String TEMPLATE_MESSAGE_URL = "https://api.weixin.qq.com/cgi-bin/message/template/send";
 
+	/**
+	 * 网络调用最大重试次数（不含首次）。
+	 */
+	private static final int MAX_RETRY = 2;
+
+	/**
+	 * 重试间隔毫秒。
+	 */
+	private static final long RETRY_SLEEP_MILLIS = 1500L;
+
 	/**
 	 * 工具类不允许实例化。
 	 */
@@ -67,27 +78,29 @@ public class Wx {
 	 * @throws InterruptedException 线程在等待响应时被中断时抛出
 	 */
 	public static String getAccessToken() throws IOException, InterruptedException {
-		String requestBody = buildRequestBody();
-		String responseBody = Url.sendRequest(
-				ACCESS_TOKEN_URL,
-				HTTP_METHOD_POST,
-				Map.of("Content-Type", "application/json;charset=UTF-8"),
-				requestBody
-		);
-
-		if (responseBody == null || responseBody.trim().isEmpty()) {
-			throw new IllegalStateException("微信 access_token 接口返回为空");
-		}
+		return executeWithRetry(() -> {
+			String requestBody = buildRequestBody();
+			String responseBody = Url.sendRequest(
+					ACCESS_TOKEN_URL,
+					HTTP_METHOD_POST,
+					Map.of("Content-Type", "application/json;charset=UTF-8"),
+					requestBody
+			);
 
-		JSONObject response = JSON.parseObject(responseBody);
-		String accessToken = response.getString("access_token");
-		if (accessToken != null && !accessToken.trim().isEmpty()) {
-			return accessToken.trim();
-		}
+			if (responseBody == null || responseBody.trim().isEmpty()) {
+				throw new IllegalStateException("微信 access_token 接口返回为空");
+			}
 
-		Integer errCode = response.getInteger("errcode");
-		String errMsg = response.getString("errmsg");
-		throw new IllegalStateException("获取微信 access_token 失败，errcode=" + errCode + ", errmsg=" + errMsg + ", response=" + responseBody);
+			JSONObject response = JSON.parseObject(responseBody);
+			String accessToken = response.getString("access_token");
+			if (accessToken != null && !accessToken.trim().isEmpty()) {
+				return accessToken.trim();
+			}
+
+			Integer errCode = response.getInteger("errcode");
+			String errMsg = response.getString("errmsg");
+			throw new IllegalStateException("获取微信 access_token 失败，errcode=" + errCode + ", errmsg=" + errMsg + ", response=" + responseBody);
+		});
 	}
 
 	/**
@@ -119,39 +132,41 @@ public class Wx {
 			throw new IllegalArgumentException("data 不能为空");
 		}
 
-		String accessToken = getAccessToken();
-		String requestUrl = TEMPLATE_MESSAGE_URL + "?access_token=" + accessToken;
-		JSONObject requestBody = new JSONObject();
-		requestBody.put("touser", touser.trim());
-		requestBody.put("template_id", templateId.trim());
-		if (url != null && !url.trim().isEmpty()) {
-			requestBody.put("url", url.trim());
-		}
-		requestBody.put("data", data);
+		return executeWithRetry(() -> {
+			String accessToken = getAccessToken();
+			String requestUrl = TEMPLATE_MESSAGE_URL + "?access_token=" + accessToken;
+			JSONObject requestBody = new JSONObject();
+			requestBody.put("touser", touser.trim());
+			requestBody.put("template_id", templateId.trim());
+			if (url != null && !url.trim().isEmpty()) {
+				requestBody.put("url", url.trim());
+			}
+			requestBody.put("data", data);
 
-		String responseBody = Url.sendRequest(
-				requestUrl,
-				HTTP_METHOD_POST,
-				Map.of("Content-Type", "application/json;charset=UTF-8"),
-				requestBody.toJSONString()
-		);
+			String responseBody = Url.sendRequest(
+					requestUrl,
+					HTTP_METHOD_POST,
+					Map.of("Content-Type", "application/json;charset=UTF-8"),
+					requestBody.toJSONString()
+			);
 
-		if (responseBody == null || responseBody.trim().isEmpty()) {
-			throw new IllegalStateException("微信模板消息接口返回为空");
-		}
+			if (responseBody == null || responseBody.trim().isEmpty()) {
+				throw new IllegalStateException("微信模板消息接口返回为空");
+			}
 
-		JSONObject response = JSON.parseObject(responseBody);
-		Integer errCode = response.getInteger("errcode");
-		String errMsg = response.getString("errmsg");
-		if (errCode != null && errCode != 0) {
-			throw new IllegalStateException("发送微信模板消息失败，errcode=" + errCode + ", errmsg=" + errMsg + ", response=" + responseBody);
-		}
+			JSONObject response = JSON.parseObject(responseBody);
+			Integer errCode = response.getInteger("errcode");
+			String errMsg = response.getString("errmsg");
+			if (errCode != null && errCode != 0) {
+				throw new IllegalStateException("发送微信模板消息失败，errcode=" + errCode + ", errmsg=" + errMsg + ", response=" + responseBody);
+			}
 
-		String msgId = response.getString("msgid");
-		if (msgId == null || msgId.trim().isEmpty()) {
-			throw new IllegalStateException("微信模板消息发送成功，但未返回 msgid，response=" + responseBody);
-		}
-		return msgId.trim();
+			String msgId = response.getString("msgid");
+			if (msgId == null || msgId.trim().isEmpty()) {
+				throw new IllegalStateException("微信模板消息发送成功，但未返回 msgid，response=" + responseBody);
+			}
+			return msgId.trim();
+		});
 	}
 
 	/**
@@ -215,4 +230,87 @@ public class Wx {
 	private static boolean isBlank(String value) {
 		return value == null || value.trim().isEmpty();
 	}
+
+	/**
+	 * 环境变量优先，未配置时使用默认值。
+	 */
+	private static String resolveEnvOrDefault(String envKey, String defaultValue) {
+		String value = System.getenv(envKey);
+		return isBlank(value) ? defaultValue : value.trim();
+	}
+
+	/**
+	 * 对网络类异常进行有限重试，降低瞬时抖动带来的失败概率。
+	 */
+	private static <T> T executeWithRetry(CheckedSupplier<T> supplier) throws IOException, InterruptedException {
+		RuntimeException lastRuntimeException = null;
+		IOException lastIoException = null;
+
+		for (int attempt = 0; attempt <= MAX_RETRY; attempt++) {
+			try {
+				return supplier.get();
+			} catch (InterruptedException ie) {
+				Thread.currentThread().interrupt();
+				throw ie;
+			} catch (IOException ioe) {
+				lastIoException = ioe;
+				if (!isRetriable(ioe) || attempt == MAX_RETRY) {
+					throw ioe;
+				}
+				System.err.println("微信接口网络异常，正在重试(" + (attempt + 1) + "/" + MAX_RETRY + ")...");
+				sleepQuietly(RETRY_SLEEP_MILLIS);
+			} catch (RuntimeException re) {
+				lastRuntimeException = re;
+				if (!isRetriable(re) || attempt == MAX_RETRY) {
+					throw re;
+				}
+				System.err.println("微信接口调用超时，正在重试(" + (attempt + 1) + "/" + MAX_RETRY + ")...");
+				sleepQuietly(RETRY_SLEEP_MILLIS);
+			}
+		}
+
+		if (lastIoException != null) {
+			throw lastIoException;
+		}
+		if (lastRuntimeException != null) {
+			throw lastRuntimeException;
+		}
+		throw new IllegalStateException("微信接口重试失败（未知异常）");
+	}
+
+	private static boolean isRetriable(Throwable throwable) {
+		if (throwable == null) {
+			return false;
+		}
+
+		Throwable current = throwable;
+		while (current != null) {
+			if (current instanceof HttpConnectTimeoutException) {
+				return true;
+			}
+			if (current instanceof java.net.SocketTimeoutException) {
+				return true;
+			}
+			if (current instanceof java.net.ConnectException) {
+				return true;
+			}
+			current = current.getCause();
+		}
+
+		String message = throwable.getMessage();
+		return message != null && message.toLowerCase().contains("timeout");
+	}
+
+	private static void sleepQuietly(long millis) {
+		try {
+			Thread.sleep(millis);
+		} catch (InterruptedException ie) {
+			Thread.currentThread().interrupt();
+		}
+	}
+
+	@FunctionalInterface
+	private interface CheckedSupplier<T> {
+		T get() throws IOException, InterruptedException;
+	}
 }
```
