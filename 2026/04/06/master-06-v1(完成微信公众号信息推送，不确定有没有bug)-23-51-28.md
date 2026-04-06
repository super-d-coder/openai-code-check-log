# AI 代码评审记录

- 来源分支: `master`
- 提交备注: 06-v1(完成微信公众号信息推送，不确定有没有bug)
- 生成时间: 2026-04-06 23:51:28 CST

## 评审结论

### 代码评审建议

#### 1. 敏感信息硬编码风险
*   **错误位置**: `openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java` 第 42-46 行
*   **问题描述**: 微信公众号 `APP_ID` 和 `APP_SECRET` 直接硬编码在代码中，存在严重的安全泄露风险。
*   **修改建议**: 移除硬编码常量，改为从环境变量中读取。
*   **修改后代码片段**:
    ```java
    // 删除原有常量定义，改为方法读取
    private static String getAppId() {
        String appId = System.getenv("WX_APP_ID");
        if (appId == null || appId.trim().isEmpty()) {
            throw new IllegalStateException("未配置环境变量 WX_APP_ID");
        }
        return appId.trim();
    }

    private static String getAppSecret() {
        String secret = System.getenv("WX_APP_SECRET");
        if (secret == null || secret.trim().isEmpty()) {
            throw new IllegalStateException("未配置环境变量 WX_APP_SECRET");
        }
        return secret.trim();
    }

    // 修改 buildRequestBody 方法调用
    private static String buildRequestBody() {
        JSONObject body = new JSONObject();
        body.put("grant_type", GRANT_TYPE);
        body.put("appid", getAppId());
        body.put("secret", getAppSecret());
        body.put("force_refresh", FORCE_REFRESH);
        return body.toJSONString();
    }
    ```

#### 2. 代码冗余与未使用代码
*   **错误位置**: `openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java` 第 101-120 行
*   **问题描述**: 新增的 `isBlank` 和 `resolveProjectName` 方法在 `OpenaiCodeCheck` 类中并未被调用，且 `Wx.java` 中已有类似实现，属于冗余代码。
*   **修改建议**: 删除 `OpenaiCodeCheck` 类中未使用的 `isBlank` 和 `resolveProjectName` 方法。
*   **修改后代码片段**:
    ```java
    // 直接删除 OpenaiCodeCheck.java 中第 101-120 行新增的两个方法
    ```

#### 3. 配置硬编码问题
*   **错误位置**: `openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java` 第 20-24 行
*   **问题描述**: 微信接收人 (`WX_TO_USER`) 和模板ID (`WX_TEMPLATE_ID`) 硬编码，不利于多环境部署或配置更改。
*   **修改建议**: 建议改为从环境变量或配置文件读取，若为演示目的可暂时保留但需添加注释说明。
*   **修改后代码片段**:
    ```java
    // 建议修改为从环境变量获取，示例如下：
    private static final String WX_TO_USER = System.getenv().getOrDefault("WX_TO_USER", "o7Oox3ArEQmsNnVUiWi8irX9Uqww");
    private static final String WX_TEMPLATE_ID = System.getenv().getOrDefault("WX_TEMPLATE_ID", "JywlRGjUcSj2b23wQtSF-9pI2D0T786TrsJbtHbdHco");
    ```

### 总结
代码主要存在**敏感信息泄露风险**，需立即移除硬编码的微信密钥；同时存在**冗余代码**，建议清理未使用的方法以保持代码整洁。

## 代码差异

```diff
diff --git a/.gitignore b/.gitignore
index 85e7c1d..4eedddc 100644
--- a/.gitignore
+++ b/.gitignore
@@ -1 +1,2 @@
 /.idea/
+**/*.class
diff --git a/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java b/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
index bc50b5b..612bf3f 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/OpenaiCodeCheck.java
@@ -1,15 +1,25 @@
 package com.superd;
 
+import com.alibaba.fastjson2.JSONObject;
 import com.superd.ai.CodeReviewClient;
 import com.superd.git.CommandExecutor;
 import com.superd.git.CommandResult;
 import com.superd.git.GitCloudRepositoryPublisher;
 import com.superd.git.GitDiffCommandBuilder;
+import com.superd.wx.Wx;
 
+import java.nio.file.Paths;
 import java.util.List;
 
 public class OpenaiCodeCheck {
 
+    // 微信模板消息接收人（openid）
+    private static final String WX_TO_USER = "o7Oox3ArEQmsNnVUiWi8irX9Uqww";
+    // 微信模板 id
+    private static final String WX_TEMPLATE_ID = "JywlRGjUcSj2b23wQtSF-9pI2D0T786TrsJbtHbdHco";
+    // 当本次未生成 md URL 时的兜底地址
+    private static final String DEFAULT_REVIEW_LOG_URL = "https://github.com/super-d-coder/openai-code-check-log/tree/main";
+
 
 
     /**
@@ -50,9 +60,11 @@ public class OpenaiCodeCheck {
             System.out.println("开始进行代码检查...");
             String reviewText = CodeReviewClient.chatClm(rules + result.getStdout());
 
-            //3. 发布代码评审结果到日志仓库（如果当前环境允许发布）
-            GitCloudRepositoryPublisher.publishReviewIfConfigured(reviewText, result.getStdout());
+            //3. 发布代码评审结果到日志仓库（如果当前环境允许发布），并获取可访问的 md 文件链接
+            String reviewMdUrl = GitCloudRepositoryPublisher.publishReviewIfConfigured(reviewText, result.getStdout());
 
+            //4.向微信推送评审结果，并提供链接到对应md文件
+            String msgId = Wx.sendReviewTemplateMessage(WX_TO_USER, WX_TEMPLATE_ID, reviewMdUrl, DEFAULT_REVIEW_LOG_URL);
 
         } catch (IllegalArgumentException e) {
             System.err.println("参数错误：" + e.getMessage());
@@ -86,5 +98,23 @@ public class OpenaiCodeCheck {
         return current;
     }
 
+    /**
+     * 判断字符串是否为 null 或仅包含空白字符。
+     */
+    private static boolean isBlank(String value) {
+        return value == null || value.trim().isEmpty();
+    }
+
+    /**
+     * 动态解析项目名称：环境变量优先，其次使用当前仓库目录名。
+     */
+    private static String resolveProjectName() throws Exception {
+        CommandResult result = CommandExecutor.run(List.of("git", "rev-parse", "--show-toplevel"));
+        if (result.getExitCode() == 0 && !isBlank(result.getStdout())) {
+            return Paths.get(result.getStdout().trim()).getFileName().toString();
+        }
+        return "unknown-project";
+    }
+
 
 }
diff --git a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
index ee0444c..ea99ec9 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/git/GitCloudRepositoryPublisher.java
@@ -97,7 +97,7 @@ public final class GitCloudRepositoryPublisher {
     }
 
     /**
-     * 将 AI 评审结果写入 markdown，并按 年/日/月 分类后推送到远端仓库。
+     * 将 AI 评审结果写入 markdown，并按 年/月/日 分类后推送到远端仓库。
      *
      * <p>生成规则：</p>
      * <ul>
@@ -111,8 +111,9 @@ public final class GitCloudRepositoryPublisher {
      * @param commitRemark 提交备注（写入 markdown 与文件名）
      * @param reviewText   AI 评审正文
      * @param codeText     被评审的代码差异
+     * @return 本次评审 markdown 的可访问 URL
      */
-    public static void publishReviewMarkdown(String remoteUrl,
+    public static String publishReviewMarkdown(String remoteUrl,
                                              String targetBranch,
                                              String sourceBranch,
                                              String commitRemark,
@@ -145,6 +146,27 @@ public final class GitCloudRepositoryPublisher {
 
         String pushCommitMessage = "docs: add ai review " + safeBranch + " " + TIME_FORMATTER.format(now);
         publishSingleFile(remoteUrl, targetBranch, relativePath, markdown, pushCommitMessage);
+        return buildGithubFileUrl(remoteUrl, targetBranch, relativePath);
+    }
+
+    /**
+     * 将仓库地址转换为 GitHub 页面地址，并拼接指定分支与文件路径。
+     */
+    private static String buildGithubFileUrl(String remoteUrl, String branch, String relativePath) {
+        String normalized = remoteUrl == null ? "" : remoteUrl.trim();
+        if (normalized.endsWith(".git")) {
+            normalized = normalized.substring(0, normalized.length() - 4);
+        }
+
+        if (normalized.startsWith("git@github.com:")) {
+            normalized = "https://github.com/" + normalized.substring("git@github.com:".length());
+        }
+
+        if (!normalized.startsWith("https://github.com/")) {
+            return "";
+        }
+
+        return normalized + "/blob/" + branch + "/" + relativePath;
     }
 
     private static ZoneId resolveReviewZoneId() {
@@ -255,13 +277,14 @@ public final class GitCloudRepositoryPublisher {
      *
      * @param reviewText AI 返回的评审正文
      * @param diffText   本次参与评审的 diff 文本
+     * @return 本次评审 markdown 的可访问 URL；若未发布则返回空字符串
      * @throws Exception 读取 git 信息或执行发布流程时抛出的异常
      */
-    public static void publishReviewIfConfigured(String reviewText, String diffText) throws Exception {
+    public static String publishReviewIfConfigured(String reviewText, String diffText) throws Exception {
         // PR 阶段只做检查，不向日志仓库写入任何文件。
         if (isPullRequestEvent()) {
             System.out.println("当前是 pull_request 事件，仅执行代码自检，不发布评审结果。");
-            return;
+            return "";
         }
 
         // 先读环境变量，便于在 CI 中按仓库变量动态切换目标仓库。
@@ -270,7 +293,7 @@ public final class GitCloudRepositoryPublisher {
         // 双重保护：仓库地址仍为空时直接跳过发布，避免后续 git 命令报错。
         if (isBlank(remoteUrl)) {
             System.out.println("未配置评审结果仓库地址，已跳过评审结果入库。");
-            return;
+            return "";
         }
 
         // 允许通过环境变量覆盖目标分支，默认写入 main。
@@ -289,7 +312,7 @@ public final class GitCloudRepositoryPublisher {
         }
 
         // 统一交给发布器处理：生成 markdown、按日期归档并推送到目标仓库。
-        GitCloudRepositoryPublisher.publishReviewMarkdown(
+        return GitCloudRepositoryPublisher.publishReviewMarkdown(
                 remoteUrl,
                 targetBranch,
                 sourceBranch,
diff --git a/openai-code-check-sdk/src/main/java/com/superd/utils/Url.java b/openai-code-check-sdk/src/main/java/com/superd/utils/Url.java
new file mode 100644
index 0000000..929f03b
--- /dev/null
+++ b/openai-code-check-sdk/src/main/java/com/superd/utils/Url.java
@@ -0,0 +1,74 @@
+package com.superd.utils;
+
+import java.io.IOException;
+import java.net.URI;
+import java.net.http.HttpClient;
+import java.net.http.HttpRequest;
+import java.net.http.HttpResponse;
+import java.nio.charset.StandardCharsets;
+import java.time.Duration;
+import java.util.Map;
+
+public class Url {
+
+	/**
+	 * 向指定 URL 发送 HTTP 请求。
+	 *
+	 * <p>该方法支持自定义请求方式（如 GET、POST、PUT、DELETE、PATCH）、请求头和请求体。
+	 * 当请求方式为 GET 时，会自动忽略请求体。</p>
+	 *
+	 * @param url 目标地址
+	 * @param method HTTP 方法，建议使用 GET、POST、PUT、DELETE、PATCH 等标准值
+	 * @param headers 请求头，可为 null
+	 * @param body 请求体，可为 null；GET 请求会忽略该参数
+	 * @return 响应正文
+	 * @throws IOException 网络请求失败时抛出
+	 * @throws InterruptedException 线程在等待响应时被中断时抛出
+	 */
+	public static String sendRequest(String url,
+									 String method,
+									 Map<String, String> headers,
+									 String body) throws IOException, InterruptedException {
+		if (url == null || url.trim().isEmpty()) {
+			throw new IllegalArgumentException("url 不能为空");
+		}
+		if (method == null || method.trim().isEmpty()) {
+			throw new IllegalArgumentException("method 不能为空");
+		}
+
+		String normalizedMethod = method.trim().toUpperCase();
+		HttpClient client = HttpClient.newBuilder()
+				.connectTimeout(Duration.ofSeconds(15))
+				.build();
+
+		HttpRequest.Builder builder = HttpRequest.newBuilder()
+				.uri(URI.create(url.trim()))
+				.timeout(Duration.ofSeconds(30));
+
+		if (headers != null) {
+			headers.forEach((key, value) -> {
+				if (key != null && value != null) {
+					builder.header(key, value);
+				}
+			});
+		}
+
+		HttpRequest.BodyPublisher bodyPublisher = buildBodyPublisher(normalizedMethod, body);
+		builder.method(normalizedMethod, bodyPublisher);
+
+		HttpResponse<String> response = client.send(builder.build(), HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8));
+		return response.body();
+	}
+
+	/**
+	 * 根据请求方式和请求体构建请求内容。
+	 */
+	private static HttpRequest.BodyPublisher buildBodyPublisher(String method, String body) {
+		if ("GET".equals(method)) {
+			return HttpRequest.BodyPublishers.noBody();
+		}
+
+		String content = body == null ? "" : body;
+		return HttpRequest.BodyPublishers.ofString(content, StandardCharsets.UTF_8);
+	}
+}
diff --git a/openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java b/openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java
new file mode 100644
index 0000000..4e533e1
--- /dev/null
+++ b/openai-code-check-sdk/src/main/java/com/superd/wx/Wx.java
@@ -0,0 +1,218 @@
+package com.superd.wx;
+
+import com.alibaba.fastjson2.JSON;
+import com.alibaba.fastjson2.JSONObject;
+import com.superd.utils.Url;
+
+import java.io.IOException;
+import java.nio.file.Paths;
+import java.util.Map;
+
+/**
+ * 微信接口工具类。
+ *
+ * <p>当前仅提供获取 access_token 的能力，后续可以继续扩展其他微信 API。</p>
+ */
+public class Wx {
+
+	/**
+	 * 微信稳定版 token 接口地址。
+	 */
+	private static final String ACCESS_TOKEN_URL = "https://api.weixin.qq.com/cgi-bin/stable_token";
+
+	/**
+	 * 固定请求方式。
+	 */
+	private static final String HTTP_METHOD_POST = "POST";
+
+	/**
+	 * 固定请求参数：授权类型。
+	 */
+	private static final String GRANT_TYPE = "client_credential";
+
+	/**
+	 * 公众号 AppId，请替换为你自己的值。
+	 */
+	private static final String APP_ID = "wx2d160f15c419f7b3";
+
+	/**
+	 * 公众号 AppSecret，请替换为你自己的值。
+	 */
+	private static final String APP_SECRET = "e5a76e38af434f706a20389fa6eca496";
+
+	/**
+	 * 是否强制刷新 token。
+	 */
+	private static final boolean FORCE_REFRESH = false;
+
+	/**
+	 * 微信模板消息接口地址。
+	 */
+	private static final String TEMPLATE_MESSAGE_URL = "https://api.weixin.qq.com/cgi-bin/message/template/send";
+
+	/**
+	 * 工具类不允许实例化。
+	 */
+	private Wx() {
+	}
+
+	/**
+	 * 获取微信 access_token。
+	 *
+	 * <p>请求会按照微信官方接口要求，向 stable_token 接口发送 POST JSON 请求，
+	 * 成功时返回 access_token；如果接口返回错误，则抛出异常并携带原始响应内容。</p>
+	 *
+	 * @return 微信 access_token
+	 * @throws IOException 网络请求失败时抛出
+	 * @throws InterruptedException 线程在等待响应时被中断时抛出
+	 */
+	public static String getAccessToken() throws IOException, InterruptedException {
+		String requestBody = buildRequestBody();
+		String responseBody = Url.sendRequest(
+				ACCESS_TOKEN_URL,
+				HTTP_METHOD_POST,
+				Map.of("Content-Type", "application/json;charset=UTF-8"),
+				requestBody
+		);
+
+		if (responseBody == null || responseBody.trim().isEmpty()) {
+			throw new IllegalStateException("微信 access_token 接口返回为空");
+		}
+
+		JSONObject response = JSON.parseObject(responseBody);
+		String accessToken = response.getString("access_token");
+		if (accessToken != null && !accessToken.trim().isEmpty()) {
+			return accessToken.trim();
+		}
+
+		Integer errCode = response.getInteger("errcode");
+		String errMsg = response.getString("errmsg");
+		throw new IllegalStateException("获取微信 access_token 失败，errcode=" + errCode + ", errmsg=" + errMsg + ", response=" + responseBody);
+	}
+
+	/**
+	 * 发送微信模板消息。
+	 *
+	 * <p>该方法会先调用 {@link #getAccessToken()} 获取 access_token，
+	 * 再向模板消息接口发送 POST JSON 请求。</p>
+	 *
+	 * <p>请求体只包含微信模板消息所需的字段：{@code touser}、{@code template_id}、
+	 * 可选的 {@code url}、以及 {@code data}；不包含小程序相关字段 {@code miniprogram}。</p>
+	 *
+	 * @param touser 接收者 openid
+	 * @param templateId 模板 id
+	 * @param url 模板跳转链接，可为 null 或空字符串；为空时不会写入请求体
+	 * @param data 模板消息内容，格式需符合微信要求，例如 {@code {"keyword1":{"value":"xxx"}}}
+	 * @return 微信返回的消息 id
+	 * @throws IOException 网络请求失败时抛出
+	 * @throws InterruptedException 线程在等待响应时被中断时抛出
+	 */
+	public static String sendTemplateMessage(String touser, String templateId, String url, JSONObject data)
+			throws IOException, InterruptedException {
+		if (touser == null || touser.trim().isEmpty()) {
+			throw new IllegalArgumentException("touser 不能为空");
+		}
+		if (templateId == null || templateId.trim().isEmpty()) {
+			throw new IllegalArgumentException("templateId 不能为空");
+		}
+		if (data == null || data.isEmpty()) {
+			throw new IllegalArgumentException("data 不能为空");
+		}
+
+		String accessToken = getAccessToken();
+		String requestUrl = TEMPLATE_MESSAGE_URL + "?access_token=" + accessToken;
+		JSONObject requestBody = new JSONObject();
+		requestBody.put("touser", touser.trim());
+		requestBody.put("template_id", templateId.trim());
+		if (url != null && !url.trim().isEmpty()) {
+			requestBody.put("url", url.trim());
+		}
+		requestBody.put("data", data);
+
+		String responseBody = Url.sendRequest(
+				requestUrl,
+				HTTP_METHOD_POST,
+				Map.of("Content-Type", "application/json;charset=UTF-8"),
+				requestBody.toJSONString()
+		);
+
+		if (responseBody == null || responseBody.trim().isEmpty()) {
+			throw new IllegalStateException("微信模板消息接口返回为空");
+		}
+
+		JSONObject response = JSON.parseObject(responseBody);
+		Integer errCode = response.getInteger("errcode");
+		String errMsg = response.getString("errmsg");
+		if (errCode != null && errCode != 0) {
+			throw new IllegalStateException("发送微信模板消息失败，errcode=" + errCode + ", errmsg=" + errMsg + ", response=" + responseBody);
+		}
+
+		String msgId = response.getString("msgid");
+		if (msgId == null || msgId.trim().isEmpty()) {
+			throw new IllegalStateException("微信模板消息发送成功，但未返回 msgid，response=" + responseBody);
+		}
+		return msgId.trim();
+	}
+
+	/**
+	 * 发送“代码评审结果”模板消息。
+	 *
+	 * <p>该方法会自动构建模板消息 data：</p>
+	 * <ul>
+	 * 	<li>project.value：动态项目名（当前目录名）</li>
+	 * 	<li>review.value：本次发布生成的 md URL；若为空则写提示文案</li>
+	 * </ul>
+	 *
+	 * @param touser 接收者 openid
+	 * @param templateId 模板 id
+	 * @param reviewMdUrl 本次发布生成的 md URL
+	 * @param defaultReviewLogUrl 当 reviewMdUrl 为空时的模板跳转兜底地址
+	 * @return 微信返回的消息 id
+	 * @throws IOException 网络请求失败时抛出
+	 * @throws InterruptedException 线程在等待响应时被中断时抛出
+	 */
+	public static String sendReviewTemplateMessage(String touser,
+											 String templateId,
+											 String reviewMdUrl,
+											 String defaultReviewLogUrl) throws IOException, InterruptedException {
+		String messageUrl = isBlank(reviewMdUrl) ? defaultReviewLogUrl : reviewMdUrl;
+		String reviewValue = isBlank(reviewMdUrl) ? "(本次未生成md文件URL)" : reviewMdUrl;
+
+		JSONObject data = new JSONObject();
+		JSONObject project = new JSONObject();
+		project.put("value", resolveProjectName());
+		JSONObject review = new JSONObject();
+		review.put("value", reviewValue);
+		data.put("project", project);
+		data.put("review", review);
+
+		return sendTemplateMessage(touser, templateId, messageUrl, data);
+	}
+
+	/**
+	 * 构建获取 access_token 所需的 JSON 请求体。
+	 */
+	private static String buildRequestBody() {
+		JSONObject body = new JSONObject();
+		body.put("grant_type", GRANT_TYPE);
+		body.put("appid", APP_ID);
+		body.put("secret", APP_SECRET);
+		body.put("force_refresh", FORCE_REFRESH);
+		return body.toJSONString();
+	}
+
+	/**
+	 * 解析当前项目名称，默认取当前工作目录名称。
+	 */
+	private static String resolveProjectName() {
+		String projectName = Paths.get(System.getProperty("user.dir")).getFileName().toString();
+		return isBlank(projectName) ? "unknown-project" : projectName;
+	}
+
+	/**
+	 * 判断字符串是否为空白。
+	 */
+	private static boolean isBlank(String value) {
+		return value == null || value.trim().isEmpty();
+	}
+}
diff --git a/openai-code-check-sdk/src/test/java/com/superd/WxTest.java b/openai-code-check-sdk/src/test/java/com/superd/WxTest.java
index c602fd7..f76365c 100644
--- a/openai-code-check-sdk/src/test/java/com/superd/WxTest.java
+++ b/openai-code-check-sdk/src/test/java/com/superd/WxTest.java
@@ -1,8 +1,34 @@
 package com.superd;
 
+import com.alibaba.fastjson2.JSONObject;
+import com.superd.wx.Wx;
+
+import java.io.IOException;
+
 public class WxTest {
 
-    public static void main(String[] args) {
-        System.out.println("hello world");
+    public static void main(String[] args) throws IOException, InterruptedException {
+        /*// 获取微信 access_token
+        String accessToken = Wx.getAccessToken();
+        System.out.println("access_token: " + accessToken);*/
+
+        //测试模板消息 发送
+        JSONObject data = new JSONObject();
+        JSONObject project = new JSONObject();
+        project.put("value", "openai-code-check");
+        JSONObject review = new JSONObject();
+        review.put("value", "通过");
+        data.put("project", project);
+        data.put("review", review);
+
+        String msgId = Wx.sendTemplateMessage(
+                "o7Oox3ArEQmsNnVUiWi8irX9Uqww",
+                "JywlRGjUcSj2b23wQtSF-9pI2D0T786TrsJbtHbdHco",
+                "https://github.com/super-d-coder/openai-code-check-log/tree/main",
+                data
+        );
+        System.out.println("msgId: " + msgId);
+
     }
+
 }
```
