# AI 代码评审记录

- 评审工程: `super-d-coder/openai-code-check`
- 来源分支: `master`
- 提交备注: 08-v4（修复显示bug）
- 提交SHA: `aa0741be`
- 生成时间: 2026-04-08 16:59:39 CST

## 评审结论

代码没有问题。

**总结**：
新增代码逻辑清晰，通过 `git log` 和 `git rev-parse` 命令动态获取提交备注和 SHA，替换了原有的默认常量，增强了代码的灵活性。异常处理（回退到默认值）也符合健壮性要求。

## 代码差异

```diff
diff --git a/openai-code-check-sdk/src/main/java/com/superd/infrastructure/git/GitCommand.java b/openai-code-check-sdk/src/main/java/com/superd/infrastructure/git/GitCommand.java
index d9c5c88..18d0aa0 100644
--- a/openai-code-check-sdk/src/main/java/com/superd/infrastructure/git/GitCommand.java
+++ b/openai-code-check-sdk/src/main/java/com/superd/infrastructure/git/GitCommand.java
@@ -338,7 +338,7 @@ public class GitCommand {
      * <ul>
      *     <li>评审仓库地址：优先读取环境变量 {@code AI_REVIEW_REMOTE_URL}，为空时使用默认地址。</li>
      *     <li>目标分支：读取 {@code AI_REVIEW_TARGET_BRANCH}，为空时默认 {@code main}。</li>
-     *     <li>提交备注：读取 {@code AI_REVIEW_COMMIT_NOTE}，为空时回退到当前仓库最近一次提交标题。</li>
+     *     <li>提交备注：通过 git 命令实时读取最近一次提交标题。</li>
      *     <li>来源分支：通过 git 命令实时读取当前分支名。</li>
      * </ul>
      *
@@ -361,15 +361,36 @@ public class GitCommand {
         if (branchResult.getExitCode() == 0 && !isBlank(branchResult.getStdout())) {
             sourceBranch = branchResult.getStdout().trim();
         }
+
+        // 直接通过 git 读取提交备注。
+        String commitRemark = null;
+        CommandResult commitMsgResult = run(List.of("git", "log", "-1", "--pretty=%s"));
+        if (commitMsgResult.getExitCode() == 0 && !isBlank(commitMsgResult.getStdout())) {
+            commitRemark = commitMsgResult.getStdout().trim();
+        }
+        if (isBlank(commitRemark)) {
+            commitRemark = DEFAULT_COMMIT_REMARK;
+        }
+
+        // 直接通过 git 读取提交 SHA。
+        String commitSha = null;
+        CommandResult shaResult = run(List.of("git", "rev-parse", "HEAD"));
+        if (shaResult.getExitCode() == 0 && !isBlank(shaResult.getStdout())) {
+            commitSha = shaResult.getStdout().trim();
+        }
+        if (isBlank(commitSha)) {
+            commitSha = DEFAULT_COMMIT_SHA;
+        }
+
         //生成并发布 markdown
         return publishReviewMarkdown(
                 AI_REVIEW_REMOTE_URL,
                 DEFAULT_TARGET_BRANCH,
                 sourceBranch,
-                DEFAULT_COMMIT_REMARK,
+                commitRemark,
                 reviewText,
                 diffText,
-                DEFAULT_COMMIT_SHA
+                commitSha
         );
     }
```
