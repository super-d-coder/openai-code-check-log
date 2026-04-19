# AI 代码评审记录

- 评审工程: `super-d-coder/small-pay-mvc`
- 来源分支: `master`
- 提交备注: feat: 接入支付功能
- 提交SHA: `452e36f8`
- 生成时间: 2026-04-19 20:51:48 CST

## 评审结论

### 代码评审意见

**1. 错误提示：日志参数传递错误**
*   **错误位置**：`AliPayController.java` 第 43 行
*   **错误代码片段**：
    ```java
    log.info("商品下单，根据商品ID创建支付单开始 userId:{} productId:{}", createPayRequestDTO.getUserId(), createPayRequestDTO.getUserId());
    ```
*   **修改建议**：第二个参数应传入 `productId`，当前重复传入了 `userId`。
*   **修改后代码片段**：
    ```java
    log.info("商品下单，根据商品ID创建支付单开始 userId:{} productId:{}", createPayRequestDTO.getUserId(), createPayRequestDTO.getProductId());
    ```

**2. 错误提示：回调地址映射不一致**
*   **错误位置**：`AliPayController.java` 第 68 行
*   **错误代码片段**：
    ```java
    @RequestMapping(value = "alipay_notify_url", method = RequestMethod.POST)
    ```
*   **修改建议**：配置文件 `application-dev.yml` 和 `form.html` 中的回调地址均为 `pay_notify`，与 Controller 定义不一致，会导致回调无法接收。建议修改 Controller 映射值。
*   **修改后代码片段**：
    ```java
    @RequestMapping(value = "pay_notify", method = RequestMethod.POST)
    ```

**3. 错误提示：空指针异常风险**
*   **错误位置**：`AliPayController.java` 第 72 行
*   **错误代码片段**：
    ```java
    if (!request.getParameter("trade_status").equals("TRADE_SUCCESS")) {
    ```
*   **修改建议**：`request.getParameter` 可能返回 null，直接调用 `equals` 方法会导致空指针异常，建议使用常量调用 equals。
*   **修改后代码片段**：
    ```java
    if (!"TRADE_SUCCESS".equals(request.getParameter("trade_status"))) {
    ```

**4. 错误提示：缺少业务逻辑实现**
*   **错误位置**：`AliPayController.java` 第 100 行
*   **错误代码片段**：
    ```java
    log.info("支付回调，支付回调，更新订单 {}", tradeNo);

    return "success";
    ```
*   **修改建议**：代码中仅打印了更新日志，缺少实际更新订单状态的代码逻辑（如调用 `orderService` 更新数据库），需补充。
*   **修改后代码片段**：
    ```java
    log.info("支付回调，支付回调，更新订单 {}", tradeNo);
    // 补充更新订单逻辑，例如：orderService.updateOrderPayStatus(tradeNo, ...);
    return "success";
    ```

**5. 错误提示：配置注入方式不统一**
*   **错误位置**：`AliPayController.java` 第 34-35 行
*   **错误代码片段**：
    ```java
    @Value("${alipay.alipay_public_key}")
    private String alipayPublicKey;
    ```
*   **修改建议**：项目已定义 `AliPayConfigProperties` 配置类，建议统一注入配置类以保持一致性，避免分散配置。
*   **修改后代码片段**：
    ```java
    @Resource
    private AliPayConfigProperties aliPayConfigProperties;
    // 方法中使用 aliPayConfigProperties.getAlipay_public_key() 替换 alipayPublicKey
    ```

**6. 错误提示：敏感信息泄露风险**
*   **错误位置**：`application-dev.yml` 第 49 行
*   **错误代码片段**：
    ```yaml
    merchant_private_key: MIIEvwIBADANBgkqhki...
    ```
*   **修改建议**：禁止在代码库中提交私钥等敏感信息，建议使用环境变量或配置中心管理，或者使用占位符替代。

### 总结
代码主要存在日志参数错误、回调地址配置不一致导致的功能失效、潜在的空指针异常以及关键业务逻辑缺失等问题。同时存在敏感信息硬编码的安全风险。建议修复以上问题后再合并代码。

## 代码差异

```diff
diff --git a/.gitignore b/.gitignore
index a0a0591..8e5a622 100644
--- a/.gitignore
+++ b/.gitignore
@@ -35,7 +35,7 @@ build/
 ### VS Code ###
 .vscode/
 
-### Mac OS ###
 .DS_Store
 /.idea/
 /data/log/
+/*/data/log/
diff --git a/docs/dev-ops/nginx/html/form.html b/docs/dev-ops/nginx/html/form.html
index 6bfd20b..d43fd9c 100644
--- a/docs/dev-ops/nginx/html/form.html
+++ b/docs/dev-ops/nginx/html/form.html
@@ -1,5 +1,5 @@
-<form name="punchout_form" method="post" action="https://openapi-sandbox.dl.alipaydev.com/gateway.do?charset=utf-8&method=alipay.trade.page.pay&sign=BpHfZxfqi937dmLh06o2LwRGzYPOEIz%2Fwu1y82krprjGZiboAv0CnveTfllt9GOZG7L8gcuEE%2FIgZc8iK4XR09rshco5rs3rYgLimqtZSih2XoEaORmdnFmVSJl5f9N4OEDlPEYFMYI8%2FR5vo18a2DdGU0Ij5OgB3OV8y6UHhmAPwTXS2o7UQ7Jz8tWWmoIO7ge4RUiEHF%2FE8UuT63nhtjjU%2BF2Y16flWftDGAYoZppsnLnd5ruF7I8PkRkoAxUl%2BBqJJkKHhkcuj%2BxLI9a9wBsH1u3MwMEo%2B9o3pSdmHXFMi3ANH5aUywZ3y4zwijJ9EsWW50FKQrTEiQ9SB6ZqHQ%3D%3D&return_url=https%3A%2F%2Fgaga.plus&notify_url=https%3A%2F%2Fxfg.natapp.cn%2Fapi%2Fv1%2Falipay%2Falipay_notify_url&version=1.0&app_id=9021000132689924&sign_type=RSA2&timestamp=2024-07-25+07%3A51%3A33&alipay_sdk=alipay-sdk-java-4.38.157.ALL&format=json">
-    <input type="hidden" name="biz_content" value="{&quot;out_trade_no&quot;:&quot;daniel82AAAA000032333361Y001&quot;,&quot;total_amount&quot;:&quot;0.01&quot;,&quot;subject&quot;:&quot;测试商品&quot;,&quot;product_code&quot;:&quot;FAST_INSTANT_TRADE_PAY&quot;}">
-    <input type="submit" value="立即支付" style="display:none" >
+<form name=\"punchout_form\" method=\"post\" action=\"https://openapi-sandbox.dl.alipaydev.com/gateway.do?charset=utf-8&method=alipay.trade.page.pay&sign=oJdNp9kILTPAiBEBebGZGsEjFpFmCVIFgffwQoV0RNNrJfR6b5sa8xOko85bviTSMKAu8eVWeEH0WeuNTejshUm4ni3tYTk9eB%2F2NdqeRmKGptyR7duhcinxVg1%2Fg%2BJxSWbcCjit2pTohxr4BxDokYqoDi1ZCtgfTy2U%2FFJPKLlE8hNMNDbE3OfBrW1fkbZSNXLlnBMdx1K9q1AdUtQkWo0MGdaCRCKmWahPmnqoe8yWIhk6rQjzPEiC6qB0agT2hBjG9kQRIybYaGmzSvPJ8j9TALDic4EbLykAxsYXl51sRvMvTJq8Xt0w6o3m4xtliqD8gy5BYMxNl5HxqDuJTA%3D%3D&return_url=https%3A%2F%2Fgaga.plus&notify_url=http%3A%2F%2Fxfg-studio.natapp1.cc%2Fapi%2Fv1%2Falipay%2Fpay_notify&version=1.0&app_id=9021000162682405&sign_type=RSA2&timestamp=2026-04-19+20%3A44%3A53&alipay_sdk=alipay-sdk-java-4.38.157.ALL&format=json\">
+    <input type=\"hidden\" name=\"biz_content\" value=\"{&quot;out_trade_no&quot;:&quot;8456416034220272&quot;,&quot;total_amount&quot;:&quot;1.68&quot;,&quot;subject&quot;:&quot;测试商品&quot;,&quot;product_code&quot;:&quot;FAST_INSTANT_TRADE_PAY&quot;}\">
+    <input type=\"submit\" value=\"立即支付\" style=\"display:none\" >
 </form>
 <script>document.forms[0].submit();</script>
\ No newline at end of file
diff --git a/small-pay-controller/src/main/java/com/superd/config/AliPayConfigProperties.java b/small-pay-controller/src/main/java/com/superd/config/AliPayConfigProperties.java
new file mode 100644
index 0000000..4ffec43
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/config/AliPayConfigProperties.java
@@ -0,0 +1,29 @@
+package com.superd.config;
+
+import lombok.Data;
+import org.springframework.boot.context.properties.ConfigurationProperties;
+
+@Data
+@ConfigurationProperties(prefix = "alipay", ignoreInvalidFields = true)
+public class AliPayConfigProperties {
+
+    // 「沙箱环境」应用ID - 您的APPID，收款账号既是你的APPID对应支付宝账号。获取地址；https://open.alipay.com/develop/sandbox/app
+    private String app_id;
+    // 「沙箱环境」商户私钥，你的PKCS8格式RSA2私钥
+    private String merchant_private_key;
+    // 「沙箱环境」支付宝公钥
+    private String alipay_public_key;
+    // 「沙箱环境」服务器异步通知页面路径
+    private String notify_url;
+    // 「沙箱环境」页面跳转同步通知页面路径 需http://格式的完整路径，不能加?id=123这类自定义参数，必须外网可以正常访问
+    private String return_url;
+    // 「沙箱环境」
+    private String gatewayUrl;
+    // 签名方式
+    private String sign_type = "RSA2";
+    // 字符编码格式
+    private String charset = "utf-8";
+    // 传输格式
+    private String format = "json";
+
+}
diff --git a/small-pay-controller/src/main/java/com/superd/config/AlipayConfig.java b/small-pay-controller/src/main/java/com/superd/config/AlipayConfig.java
new file mode 100644
index 0000000..b7ad6bc
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/config/AlipayConfig.java
@@ -0,0 +1,23 @@
+package com.superd.config;
+
+import com.alipay.api.AlipayClient;
+import com.alipay.api.DefaultAlipayClient;
+import org.springframework.boot.context.properties.EnableConfigurationProperties;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+
+@Configuration
+@EnableConfigurationProperties(AliPayConfigProperties.class)
+public class AlipayConfig {
+
+    @Bean
+    public AlipayClient alipayClient(AliPayConfigProperties properties) {
+        return new DefaultAlipayClient(properties.getGatewayUrl(),
+                properties.getApp_id(),
+                properties.getMerchant_private_key(),
+                properties.getFormat(),
+                properties.getCharset(),
+                properties.getAlipay_public_key(),
+                properties.getSign_type()); // 替换为实际的 AlipayClient 实例
+    }
+}
diff --git a/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java b/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java
new file mode 100644
index 0000000..d0199ea
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java
@@ -0,0 +1,111 @@
+package com.superd.controller;
+
+
+import com.alipay.api.AlipayApiException;
+import com.alipay.api.internal.util.AlipaySignature;
+import com.superd.common.constants.Constants;
+import com.superd.common.response.Response;
+import com.superd.controller.dto.CreatePayRequestDTO;
+import com.superd.domain.req.ShopCartReq;
+import com.superd.domain.res.PayOrderRes;
+import com.superd.service.IOrderService;
+import lombok.extern.slf4j.Slf4j;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.web.bind.annotation.*;
+
+import javax.annotation.Resource;
+import javax.servlet.http.HttpServletRequest;
+import java.util.HashMap;
+import java.util.Map;
+
+@Slf4j
+@RestController()
+@CrossOrigin("*")
+@RequestMapping("/api/v1/alipay/")
+public class AliPayController {
+
+    @Value("${alipay.alipay_public_key}")
+    private String alipayPublicKey;
+
+    @Resource
+    private IOrderService orderService;
+
+    /**
+     * http://localhost:8080/api/v1/alipay/create_pay_order
+     *
+     * {
+     *     "userId": "10001",
+     *     "productId": "100001"
+     * }
+     */
+    @RequestMapping(value = "create_pay_order", method =  RequestMethod.POST)
+    public Response<String> createPayOrder(@RequestBody CreatePayRequestDTO createPayRequestDTO){
+        try {
+            log.info("商品下单，根据商品ID创建支付单开始 userId:{} productId:{}", createPayRequestDTO.getUserId(), createPayRequestDTO.getUserId());
+            String userId = createPayRequestDTO.getUserId();
+            String productId = createPayRequestDTO.getProductId();
+            // 下单
+            PayOrderRes payOrderRes = orderService.createOrder(ShopCartReq.builder()
+                    .userId(userId)
+                    .productId(productId)
+                    .build());
+
+            log.info("商品下单，根据商品ID创建支付单完成 userId:{} productId:{} orderId:{}", userId, productId, payOrderRes.getOrderId());
+            return Response.<String>builder()
+                    .code(Constants.ResponseCode.SUCCESS.getCode())
+                    .info(Constants.ResponseCode.SUCCESS.getInfo())
+                    .data(payOrderRes.getPayUrl())
+                    .build();
+        } catch (Exception e) {
+            log.error("商品下单，根据商品ID创建支付单失败 userId:{} productId:{}", createPayRequestDTO.getUserId(), createPayRequestDTO.getUserId(), e);
+            return Response.<String>builder()
+                    .code(Constants.ResponseCode.UN_ERROR.getCode())
+                    .info(Constants.ResponseCode.UN_ERROR.getInfo())
+                    .build();
+        }
+    }
+
+    /**
+     * http://xfg-studio.natapp1.cc/api/v1/alipay/alipay_notify_url
+     */
+    @RequestMapping(value = "alipay_notify_url", method = RequestMethod.POST)
+    public String payNotify(HttpServletRequest request) throws AlipayApiException {
+        log.info("支付回调，消息接收 {}", request.getParameter("trade_status"));
+
+        if (!request.getParameter("trade_status").equals("TRADE_SUCCESS")) {
+            return "false";
+        }
+
+        Map<String, String> params = new HashMap<>();
+        Map<String, String[]> requestParams = request.getParameterMap();
+        for (String name : requestParams.keySet()) {
+            params.put(name, request.getParameter(name));
+        }
+
+        String tradeNo = params.get("out_trade_no");
+        String gmtPayment = params.get("gmt_payment");
+        String alipayTradeNo = params.get("trade_no");
+
+        String sign = params.get("sign");
+        String content = AlipaySignature.getSignCheckContentV1(params);
+        boolean checkSignature = AlipaySignature.rsa256CheckContent(content, sign, alipayPublicKey, "UTF-8"); // 验证签名
+        // 支付宝验签
+        if (!checkSignature) {
+            return "false";
+        }
+
+        // 验签通过
+        log.info("支付回调，交易名称: {}", params.get("subject"));
+        log.info("支付回调，交易状态: {}", params.get("trade_status"));
+        log.info("支付回调，支付宝交易凭证号: {}", params.get("trade_no"));
+        log.info("支付回调，商户订单号: {}", params.get("out_trade_no"));
+        log.info("支付回调，交易金额: {}", params.get("total_amount"));
+        log.info("支付回调，买家在支付宝唯一id: {}", params.get("buyer_id"));
+        log.info("支付回调，买家付款时间: {}", params.get("gmt_payment"));
+        log.info("支付回调，买家付款金额: {}", params.get("buyer_pay_amount"));
+        log.info("支付回调，支付回调，更新订单 {}", tradeNo);
+
+        return "success";
+    }
+
+}
diff --git a/small-pay-controller/src/main/java/com/superd/controller/dto/CreatePayRequestDTO.java b/small-pay-controller/src/main/java/com/superd/controller/dto/CreatePayRequestDTO.java
new file mode 100644
index 0000000..d67c42a
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/controller/dto/CreatePayRequestDTO.java
@@ -0,0 +1,13 @@
+package com.superd.controller.dto;
+
+import lombok.Data;
+
+@Data
+public class CreatePayRequestDTO {
+
+    // 用户ID 【实际产生中会通过登录模块获取，不需要透彻】
+    private String userId;
+    // 产品编号
+    private String productId;
+
+}
diff --git a/small-pay-controller/src/main/resources/application-dev.yml b/small-pay-controller/src/main/resources/application-dev.yml
index cdb11f7..6777a0c 100644
--- a/small-pay-controller/src/main/resources/application-dev.yml
+++ b/small-pay-controller/src/main/resources/application-dev.yml
@@ -42,6 +42,16 @@ mybatis:
   mapper-locations: classpath*:mybatis/mapper/*.xml
   config-location: classpath:mybatis/config/mybatis-config.xml
 
+# 支付宝支付 - 沙箱 https://opendocs.alipay.com/common/02kkv7
+alipay:
+  enabled: true
+  app_id: 9021000162682405
+  merchant_private_key: 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
+  alipay_public_key: MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAnOHAalNHcXIAHsPxpjMcvp8VudUEIml2gWADmzJakBfn3RzSWtqUbRh2u/z1b9xWjG+NzqYUxUM0bANkgE4omcl6Q0T4H6U4P0hsn+Wcsa3ZOLToBk+XwRnW6GJj0IppR2kVsqVMqJjVD+UcRPX8XpUy4U+d2q9FmWvPsGUlC3XmM5/nRUrOzA+CoEvmz2Q/c2f8/tRFHLdvRM6peGk6Y2xtCCPe/8cBm3X97g53rFdyRkjflc2UmqlU6dbzBvJk6u9mX+1mKK2fvEuPlDx3IB5L7JXo2XGjwfNJ41z20/h9TbIEheFYAQ77i+zdw0NcORT8MiR0kZm4aXMMJEmTUQIDAQAB/T3EZTHtQIGwF705Yrq1bd63l70iTfkrS0Ry9f72SDZEBBLllXfFo+otChwRRN+UXDd8X/bplV3/cbRncV5yWRnHHCgzQiwpH3ilS+sOmMfdfac0bi/xB7HIU6nUX04VCjAR7itSr0OmU8HC6p20Ubvjs45R6VuR7FMI+OahCd3LDe/ayelScfQ4zavruk4HGx3TDH4hLDA3N+xid5Cu5erLDPHtFXfnQHI4n/opQaXo5wIDAQAB
+#  todo 需要修改为自己地址
+  notify_url: http://xfg-studio.natapp1.cc/api/v1/alipay/pay_notify
+  return_url: https://gaga.plus
+  gatewayUrl: https://openapi-sandbox.dl.alipaydev.com/gateway.do
 
 
 # 日志
diff --git a/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml b/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml
index 628976c..46278bb 100644
--- a/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml
+++ b/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml
@@ -23,6 +23,10 @@
         values(#{userId}, #{productId}, #{productName}, #{orderId}, #{orderTime},
         #{totalAmount}, #{status}, now(), now())
     </insert>
+    <update id="updateOrderPayInfo">
+        update pay_order set pay_url = #{payUrl}, status = #{status}, update_time = now()
+        where order_id = #{orderId}
+    </update>
 
     <select id="queryUnPayOrder" parameterType="com.superd.domain.po.PayOrder" resultMap="dataMap">
         select product_id, product_name, order_id, order_time, total_amount, status, pay_url
diff --git a/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java b/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java
index 4ad4364..597a4c0 100644
--- a/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java
+++ b/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java
@@ -11,4 +11,5 @@ public interface IOrderDao {
 
     PayOrder queryUnPayOrder(PayOrder payOrder);
 
+    void updateOrderPayInfo(PayOrder payOrder);
 }
diff --git a/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java b/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java
index 04adbe1..bafa03c 100644
--- a/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java
+++ b/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java
@@ -1,5 +1,9 @@
 package com.superd.service.impl;
 
+import com.alibaba.fastjson.JSONObject;
+import com.alipay.api.AlipayApiException;
+import com.alipay.api.AlipayClient;
+import com.alipay.api.request.AlipayTradePagePayRequest;
 import com.superd.common.constants.Constants;
 import com.superd.dao.IOrderDao;
 import com.superd.domain.po.PayOrder;
@@ -10,9 +14,12 @@ import com.superd.service.IOrderService;
 import com.superd.service.rpc.ProductRPC;
 import lombok.extern.slf4j.Slf4j;
 import org.apache.commons.lang3.RandomStringUtils;
+import org.springframework.beans.factory.annotation.Autowired;
+import org.springframework.beans.factory.annotation.Value;
 import org.springframework.stereotype.Service;
 
 import javax.annotation.Resource;
+import java.math.BigDecimal;
 import java.util.Date;
 
 /**
@@ -28,6 +35,12 @@ public class OrderServiceImpl implements IOrderService {
     private IOrderDao orderDao;
     @Resource
     private ProductRPC productRPC;
+    @Autowired
+    private AlipayClient alipayClient;
+    @Value("${alipay.notify_url}")
+    private String notifyUrl;
+    @Value("${alipay.return_url}")
+    private String returnUrl;
 
     @Override
     public PayOrderRes createOrder(ShopCartReq shopCartReq) throws Exception {
@@ -45,7 +58,12 @@ public class OrderServiceImpl implements IOrderService {
                     .payUrl(unpaidOrder.getPayUrl())
                     .build();
         } else if (null != unpaidOrder && Constants.OrderStatusEnum.CREATE.getCode().equals(unpaidOrder.getStatus())){
-            // todo 调用支付宝沙箱创建订单
+            log.info("创建订单-存在，存在未创建支付单订单，创建支付单开始 userId:{} productId:{} orderId:{}", shopCartReq.getUserId(), shopCartReq.getProductId(), unpaidOrder.getOrderId());
+            PayOrder payOrder = doPrepayOrder(unpaidOrder.getProductId(), unpaidOrder.getProductName(), unpaidOrder.getOrderId(), unpaidOrder.getTotalAmount());
+            return PayOrderRes.builder()
+                    .orderId(payOrder.getOrderId())
+                    .payUrl(payOrder.getPayUrl())
+                    .build();
         }
 
         // 2. 查询商品 & 创建订单
@@ -61,14 +79,37 @@ public class OrderServiceImpl implements IOrderService {
                         .status(Constants.OrderStatusEnum.CREATE.getCode())
                 .build());
 
-        // 3. 创建支付单 todo
-
+        PayOrder payOrder = doPrepayOrder(shopCartReq.getProductId(), productVO.getProductName(), orderId, productVO.getPrice());
 
         return PayOrderRes.builder()
                 .orderId(orderId)
-                .payUrl("暂无")
+                .payUrl(payOrder.getPayUrl())
                 .build();
     }
 
+    private PayOrder doPrepayOrder(String productId, String productName, String orderId, BigDecimal totalAmount) throws AlipayApiException {
+        AlipayTradePagePayRequest request = new AlipayTradePagePayRequest();
+        request.setNotifyUrl(notifyUrl);
+        request.setReturnUrl(returnUrl);
+
+        JSONObject bizContent = new JSONObject();
+        bizContent.put("out_trade_no", orderId);
+        bizContent.put("total_amount", totalAmount.toString());
+        bizContent.put("subject", productName);
+        bizContent.put("product_code", "FAST_INSTANT_TRADE_PAY");
+        request.setBizContent(bizContent.toString());
+
+        String form = alipayClient.pageExecute(request).getBody();
+
+        PayOrder payOrder = new PayOrder();
+        payOrder.setOrderId(orderId);
+        payOrder.setPayUrl(form);
+        payOrder.setStatus(Constants.OrderStatusEnum.PAY_WAIT.getCode());
+
+        orderDao.updateOrderPayInfo(payOrder);
+
+        return payOrder;
+    }
+
 
 }
```
