# AI 代码评审记录

- 评审工程: `super-d-coder/small-pay-mvc`
- 来源分支: `master`
- 提交备注: feat: 接入前端部分
- 提交SHA: `53084050`
- 生成时间: 2026-04-20 22:26:26 CST

## 评审结论

### 代码评审意见

**1. 日志级别使用不当**
*   **错误位置：** `small-pay-controller/src/main/java/com/superd/controller/AliPayController.java`
*   **错误说明：** 支付回调验签失败属于安全风险或异常情况，使用 `info` 级别日志容易被淹没，不利于问题排查和监控报警。
*   **修改建议：** 将日志级别改为 `error` 或 `warn`。
*   **修改后代码片段：**
    ```java
            log.error("支付回调，验签失败，交易名称: {}, 交易状态: {}, 支付宝交易凭证号: {}, 商户订单号: {}, 交易金额: {}, 买家在支付宝唯一id: {}, 买家付款时间: {}, 买家付款金额: {}", params.get("subject"), params.get("trade_status"), params.get("trade_no"), params.get("out_trade_no"), params.get("total_amount"), params.get("buyer_id"), params.get("gmt_payment"), params.get("buyer_pay_amount"));
    ```

**2. 前端硬编码接口地址**
*   **错误位置：** `docs/dev-ops/nginx/html/index.html` 及 `docs/dev-ops/nginx/html/login.html`
*   **错误说明：** 前端代码中硬编码了 `127.0.0.1:8080` 的地址，这在生产环境或通过 Nginx 代理访问时会导致跨域问题或接口无法访问。
*   **修改建议：** 建议使用相对路径，由 Nginx 配置反向代理来转发请求。
*   **修改后代码片段:**
    ```javascript
    // index.html
    var url = '/api/v1/alipay/create_pay_order';
    ```
*   **修改后代码片段:**
    ```javascript
    // login.html
    fetch('/api/v1/login/weixin_qrcode_ticket')
    // ...
    fetch(`/api/v1/login/check_login?ticket=${ticket}`)
    ```

### 总结
代码主要存在两个问题：一是后端验签失败日志级别过低，不利于安全监控；二是前端硬编码了本地 IP 和端口，破坏了部署的灵活性。建议修正日志级别并使用相对路径代替硬编码地址。

## 代码差异

```diff
diff --git a/docs/dev-ops/nginx/html/index.html b/docs/dev-ops/nginx/html/index.html
index 92d2d4d..296e4a2 100644
--- a/docs/dev-ops/nginx/html/index.html
+++ b/docs/dev-ops/nginx/html/index.html
@@ -68,7 +68,7 @@
         <p>价格：¥1.68</p>
     </div>
     <button id="orderButton" class="order-button">立即下单「沙箱支付」</button>
-    <span class="account-info">测试账号：kvhmoj3832@sandbox.com 密码：111111 支付：111111</span>
+    <span class="account-info">测试账号：yoonwg0159@sandbox.com 密码：111111 支付：111111</span>
 </div>
 
 <script>
@@ -90,8 +90,8 @@ document.getElementById('orderButton').addEventListener('click', function() {
         return;
     }
 
-    var productId = "100010090091";
-    var url = 'http://127.0.0.1:8091/api/v1/alipay/create_pay_order';
+    var productId = "donkQuWanL";
+    var url = 'http://127.0.0.1:8080/api/v1/alipay/create_pay_order';
 
     var requestBody = {
         userId: userId,
diff --git a/docs/dev-ops/nginx/html/login.html b/docs/dev-ops/nginx/html/login.html
index 8ac191e..dd447a0 100644
--- a/docs/dev-ops/nginx/html/login.html
+++ b/docs/dev-ops/nginx/html/login.html
@@ -52,7 +52,7 @@
 <script>
     document.addEventListener("DOMContentLoaded", function() {
         // 获取二维码 ticket
-        fetch('http://localhost:8091/api/v1/login/weixin_qrcode_ticket')
+        fetch('http://localhost:8080/api/v1/login/weixin_qrcode_ticket')
             .then(response => response.json())
             .then(data => {
                 if (data.code === "0000") {
@@ -73,7 +73,7 @@
             });
 
         function checkLoginStatus(ticket, intervalId) {
-            fetch(`http://localhost:8091/api/v1/login/check_login?ticket=${ticket}`)
+            fetch(`http://localhost:8080/api/v1/login/check_login?ticket=${ticket}`)
                 .then(response => response.json())
                 .then(data => {
                     if (data.code === "0000") {
diff --git a/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java b/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java
index 78d53e7..f72e931 100644
--- a/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java
+++ b/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java
@@ -91,6 +91,7 @@ public class AliPayController {
         boolean checkSignature = AlipaySignature.rsa256CheckContent(content, sign, alipayPublicKey, "UTF-8"); // 验证签名
         // 支付宝验签
         if (!checkSignature) {
+            log.info("支付回调，验签失败，交易名称: {}, 交易状态: {}, 支付宝交易凭证号: {}, 商户订单号: {}, 交易金额: {}, 买家在支付宝唯一id: {}, 买家付款时间: {}, 买家付款金额: {}", params.get("subject"), params.get("trade_status"), params.get("trade_no"), params.get("out_trade_no"), params.get("total_amount"), params.get("buyer_id"), params.get("gmt_payment"), params.get("buyer_pay_amount"));
             return "false";
         }
```
