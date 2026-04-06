# AI 代码评审记录

- 来源分支: `master`
- 提交备注: 05-v7（修复超时bug）
- 生成时间: 2026-04-06 21:24:22 CST

## 评审结论

### 代码评审结果

#### 1. 错误指出
**错误提示**：测试类 `WxTest.java` 使用了 `main` 方法而非标准测试框架（如 JUnit），且包含无意义的临时代码，不应提交到项目中。

**错误位置**：`openai-code-check-sdk/src/test/java/com/superd/WxTest.java`

**错误代码片段**：
```java
public class WxTest {

    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

**建议**：删除该文件。如果需要进行单元测试，应使用 JUnit 等测试框架编写标准的测试用例。

**修改后的代码片段**：
(建议直接删除该文件)

---

#### 2. 代码建议 (CodeReviewClient.java)
**建议**：`CodeReviewClient.java` 的修改逻辑正确，增强了重试机制的健壮性（支持 `SocketTimeoutException` 和异常链检查），参数调整（重试次数和间隔）属于合理的优化策略，代码无明显错误。

---

### 总结
1. **CodeReviewClient.java**：修改无误，增强了异常处理和重试策略，建议保留。
2. **WxTest.java**：属于无效的测试代码，建议删除。

## 代码差异

```diff
diff --git a/openai-code-check-sdk/src/main/java/com/superd/ai/CodeReviewClient.java b/openai-code-check-sdk/src/main/java/com/superd/ai/CodeReviewClient.java
index 9f2ec3e..c1aa52e 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/ai/CodeReviewClient.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/ai/CodeReviewClient.java
@@ -6,6 +6,7 @@ import ai.z.openapi.service.model.ChatCompletionResponse;
 import ai.z.openapi.service.model.ChatMessage;
 import ai.z.openapi.service.model.ChatMessageRole;
 
+import java.net.SocketTimeoutException;
 import java.util.Arrays;
 
 public final class CodeReviewClient {
@@ -13,13 +14,13 @@ public final class CodeReviewClient {
     // API KEY
     private static final String API_KEY = "8b4e64f09197476781534cc3f39296de.kjYh9O6M4VLZxmhU";
 
-    // 默认最大token数
+    // 默认最大 token 数
     private static final int MAX_TOKEN = 1024 * 32;
 
     // 控制传入模型的文本长度，避免过大 diff 导致服务端报错。
     private static final int MAX_CONTENT_LENGTH = 24000;
 
-    private static final int MAX_RETRY = 1;
+    private static final int MAX_RETRY = 2;
 
     /**
      * 工具类不允许实例化。
@@ -75,7 +76,7 @@ public final class CodeReviewClient {
                 }
 
                 System.err.println("模型服务暂时不可用，正在重试...");
-                sleepQuietly(1200L);
+                sleepQuietly(2000L);
             }
         }
 
@@ -155,11 +156,36 @@ public final class CodeReviewClient {
      * @return true 表示建议重试；false 表示直接失败
      */
     private static boolean isRetriable(Exception e) {
+        if (e == null) {
+            return false;
+        }
+        if (hasCause(e, SocketTimeoutException.class)) {
+            return true;
+        }
+
         String message = e.getMessage();
         if (message == null) {
             return false;
         }
-        return message.contains("HTTP 500") || message.contains("Network error");
+        return message.contains("HTTP 500") || message.contains("Network error") || message.contains("timeout");
+    }
+
+    /**
+     * 判断异常链中是否包含指定类型的根因或中间 cause。
+     *
+     * @param throwable 待检查异常
+     * @param type 目标异常类型
+     * @return true 表示异常链中存在该类型
+     */
+    private static boolean hasCause(Throwable throwable, Class<? extends Throwable> type) {
+        Throwable current = throwable;
+        while (current != null) {
+            if (type.isInstance(current)) {
+                return true;
+            }
+            current = current.getCause();
+        }
+        return false;
     }
 
     /**
diff --git a/openai-code-check-sdk/src/test/java/com/superd/WxTest.java b/openai-code-check-sdk/src/test/java/com/superd/WxTest.java
new file mode 100644
index 0000000..c602fd7
--- /dev/null
+++ b/openai-code-check-sdk/src/test/java/com/superd/WxTest.java
@@ -0,0 +1,8 @@
+package com.superd;
+
+public class WxTest {
+
+    public static void main(String[] args) {
+        System.out.println("hello world");
+    }
+}
```
