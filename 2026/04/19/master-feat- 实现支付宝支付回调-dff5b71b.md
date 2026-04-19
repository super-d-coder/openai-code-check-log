# AI 代码评审记录

- 评审工程: `super-d-coder/small-pay-mvc`
- 来源分支: `master`
- 提交备注: feat: 实现支付宝支付回调
- 提交SHA: `dff5b71b`
- 生成时间: 2026-04-19 21:31:20 CST

## 评审结论

代码评审结果如下：

### 1. 错误提示：SQL更新缺少状态校验
**错误位置**：`small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml`
**错误说明**：`changeOrderPaySuccess` 和 `changeOrderClose` 方法在更新订单状态时，未添加 `status = 'PAY_WAIT'` 作为更新条件。在并发场景下（如支付回调与定时任务同时执行），可能导致订单状态被错误覆盖（例如：已关闭的订单被更新为支付成功）。
**修改建议**：在 `where` 子句中增加状态判断条件。
**修改后的代码片段**：
```xml
    <update id="changeOrderPaySuccess" parameterType="com.superd.domain.po.PayOrder">
        update pay_order set status = #{status}, pay_time = now(), update_time = now()
        where order_id = #{orderId} and status = 'PAY_WAIT'
    </update>

    <update id="changeOrderClose" parameterType="java.lang.String">
        update pay_order set status = 'CLOSE', pay_time = now(), update_time = now()
        where order_id = #{orderId} and status = 'PAY_WAIT'
    </update>
```

### 2. 错误提示：支付状态判断逻辑有误
**错误位置**：`small-pay-controller/src/main/java/com/superd/job/NoPayNotifyOrderJob.java`
**错误说明**：代码仅判断了支付宝接口返回的 `code == "10000"`，这仅表示接口调用成功，并不代表交易支付成功。需进一步判断 `trade_status` 是否为 `TRADE_SUCCESS` 或 `TRADE_FINISHED`。
**修改建议**：增加交易状态的判断逻辑。
**修改后的代码片段**：
```java
                AlipayTradeQueryResponse response = alipayClient.execute(request);
                if (response.isSuccess()) {
                    String tradeStatus = response.getTradeStatus();
                    if ("TRADE_SUCCESS".equals(tradeStatus) || "TRADE_FINISHED".equals(tradeStatus)) {
                        orderService.changeOrderPaySuccess(orderId);
                    }
                }
```

### 3. 错误提示：事件发布逻辑存在隐患
**错误位置**：`small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java` 与 `small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java`
**错误说明**：`changeOrderPaySuccess` 方法在 DAO 层定义为 `void`，Service 层直接调用并发布事件。如果因为并发导致数据库更新失败（行数为0），Service 层无法感知，仍会错误地发布支付成功事件。
**修改建议**：DAO 层方法返回 `boolean` (或 `int`)，Service 层根据更新结果决定是否发布事件。
**修改后的代码片段**：
```java
// IOrderDao.java
boolean changeOrderPaySuccess(PayOrder order);

// OrderServiceImpl.java
    @Override
    public void changeOrderPaySuccess(String orderId) {
        PayOrder payOrderReq = new PayOrder();
        payOrderReq.setOrderId(orderId);
        payOrderReq.setStatus(Constants.OrderStatusEnum.PAY_SUCCESS.getCode());
        boolean isSuccess = orderDao.changeOrderPaySuccess(payOrderReq);
        
        if (isSuccess) {
            eventBus.post(JSON.toJSONString(payOrderReq));
        }
    }
```

### 总结
代码主要存在**并发安全性**和**业务逻辑判断**两方面问题：
1. 数据库更新操作缺少状态前置校验，易引发并发状态覆盖。
2. 支付主动查询任务仅校验了接口通信状态，未校验交易业务状态。
3. 事件驱动逻辑缺乏对数据库操作结果的判断，可能导致虚假消息通知。

建议按照上述意见修改，确保订单状态流转的准确性与一致性。

## 代码差异

```diff
diff --git a/docs/dev-ops/nginx/html/form.html b/docs/dev-ops/nginx/html/form.html
index d43fd9c..460182d 100644
--- a/docs/dev-ops/nginx/html/form.html
+++ b/docs/dev-ops/nginx/html/form.html
@@ -1,5 +1,5 @@
-<form name=\"punchout_form\" method=\"post\" action=\"https://openapi-sandbox.dl.alipaydev.com/gateway.do?charset=utf-8&method=alipay.trade.page.pay&sign=oJdNp9kILTPAiBEBebGZGsEjFpFmCVIFgffwQoV0RNNrJfR6b5sa8xOko85bviTSMKAu8eVWeEH0WeuNTejshUm4ni3tYTk9eB%2F2NdqeRmKGptyR7duhcinxVg1%2Fg%2BJxSWbcCjit2pTohxr4BxDokYqoDi1ZCtgfTy2U%2FFJPKLlE8hNMNDbE3OfBrW1fkbZSNXLlnBMdx1K9q1AdUtQkWo0MGdaCRCKmWahPmnqoe8yWIhk6rQjzPEiC6qB0agT2hBjG9kQRIybYaGmzSvPJ8j9TALDic4EbLykAxsYXl51sRvMvTJq8Xt0w6o3m4xtliqD8gy5BYMxNl5HxqDuJTA%3D%3D&return_url=https%3A%2F%2Fgaga.plus&notify_url=http%3A%2F%2Fxfg-studio.natapp1.cc%2Fapi%2Fv1%2Falipay%2Fpay_notify&version=1.0&app_id=9021000162682405&sign_type=RSA2&timestamp=2026-04-19+20%3A44%3A53&alipay_sdk=alipay-sdk-java-4.38.157.ALL&format=json\">
-    <input type=\"hidden\" name=\"biz_content\" value=\"{&quot;out_trade_no&quot;:&quot;8456416034220272&quot;,&quot;total_amount&quot;:&quot;1.68&quot;,&quot;subject&quot;:&quot;测试商品&quot;,&quot;product_code&quot;:&quot;FAST_INSTANT_TRADE_PAY&quot;}\">
-    <input type=\"submit\" value=\"立即支付\" style=\"display:none\" >
+<form name="punchout_form" method="post" action="https://openapi-sandbox.dl.alipaydev.com/gateway.do?charset=utf-8&method=alipay.trade.page.pay&sign=nxDnIXH%2BFN03dDYdzMBSuYn%2FxT7GUL63%2B8AuQVi8E63Cm5V0V%2FtSu1i8%2FY5KEBSDNn9j9ulyT5ngs6rBs2n8wfAB4aIAS%2FNBtFTOBnz8EsuggW67Z0J1zF2rrsY7g9U%2F3%2F6IZhL4GUsAE7sB%2BbA6mkLGfZQtgARiRvXjbHpFJJvueZIMfCiKrSAJ%2Fkq1XyQqw4luCm6qEB1Oj7jp5rzl86zIk1roNuFFAVAilndIlmQ8KicwgFhcAMxsejPU3TgjSrmMcejPv53iuANwgSRGnTydAqT3TsWH0nFeyk%2FkE4GdxqJIjKnn5WXqTA9M4jTXXCBbSZK%2BsysWI%2FgH5jwN4Q%3D%3D&return_url=https%3A%2F%2Fgaga.plus&notify_url=http%3A%2F%2Fm4e393d6.natappfree.cc%2Fapi%2Fv1%2Falipay%2Fpay_notify&version=1.0&app_id=9021000162682405&sign_type=RSA2&timestamp=2026-04-19+21%3A23%3A41&alipay_sdk=alipay-sdk-java-4.38.157.ALL&format=json">
+    <input type="hidden" name="biz_content" value="{&quot;out_trade_no&quot;:&quot;1488083243378084&quot;,&quot;total_amount&quot;:&quot;1.68&quot;,&quot;subject&quot;:&quot;测试商品&quot;,&quot;product_code&quot;:&quot;FAST_INSTANT_TRADE_PAY&quot;}">
+    <input type="submit" value="立即支付" style="display:none" >
 </form>
 <script>document.forms[0].submit();</script>
\ No newline at end of file
diff --git a/small-pay-controller/src/main/java/com/superd/Application.java b/small-pay-controller/src/main/java/com/superd/Application.java
index 733790b..f91504b 100644
--- a/small-pay-controller/src/main/java/com/superd/Application.java
+++ b/small-pay-controller/src/main/java/com/superd/Application.java
@@ -3,9 +3,11 @@ package com.superd;
 
 import org.springframework.beans.factory.annotation.Configurable;
 import org.springframework.boot.autoconfigure.SpringBootApplication;
+import org.springframework.scheduling.annotation.EnableScheduling;
 
 @SpringBootApplication
 @Configurable
+@EnableScheduling
 public class Application {
     public static void main(String[] args) {
         org.springframework.boot.SpringApplication.run(Application.class, args);
diff --git a/small-pay-controller/src/main/java/com/superd/config/Retrofit2Config.java b/small-pay-controller/src/main/java/com/superd/config/Retrofit2Config.java
index 6f837f4..aa41bec 100644
--- a/small-pay-controller/src/main/java/com/superd/config/Retrofit2Config.java
+++ b/small-pay-controller/src/main/java/com/superd/config/Retrofit2Config.java
@@ -1,5 +1,7 @@
 package com.superd.config;
 
+import com.google.common.eventbus.EventBus;
+import com.superd.listener.OrderPaySuccessListener;
 import com.superd.service.weixin.IWeixinApiService;
 import lombok.extern.slf4j.Slf4j;
 import org.springframework.context.annotation.Bean;
@@ -25,4 +27,10 @@ public class Retrofit2Config {
         return retrofit.create(IWeixinApiService.class);
     }
 
+    @Bean
+    public EventBus eventBusListener(OrderPaySuccessListener listener){
+        EventBus eventBus = new EventBus();
+        eventBus.register(listener);
+        return eventBus;
+    }
 }
diff --git a/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java b/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java
index d0199ea..78d53e7 100644
--- a/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java
+++ b/small-pay-controller/src/main/java/com/superd/controller/AliPayController.java
@@ -66,7 +66,7 @@ public class AliPayController {
     }
 
     /**
-     * http://xfg-studio.natapp1.cc/api/v1/alipay/alipay_notify_url
+     * http://m4e393d6.natappfree.cc/api/v1/alipay/alipay_notify_url
      */
     @RequestMapping(value = "alipay_notify_url", method = RequestMethod.POST)
     public String payNotify(HttpServletRequest request) throws AlipayApiException {
@@ -105,6 +105,8 @@ public class AliPayController {
         log.info("支付回调，买家付款金额: {}", params.get("buyer_pay_amount"));
         log.info("支付回调，支付回调，更新订单 {}", tradeNo);
 
+        orderService.changeOrderPaySuccess(tradeNo);
+
         return "success";
     }
 
diff --git a/small-pay-controller/src/main/java/com/superd/job/NoPayNotifyOrderJob.java b/small-pay-controller/src/main/java/com/superd/job/NoPayNotifyOrderJob.java
new file mode 100644
index 0000000..0f1e2a8
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/job/NoPayNotifyOrderJob.java
@@ -0,0 +1,54 @@
+package com.superd.job;
+
+import com.alipay.api.AlipayClient;
+import com.alipay.api.domain.AlipayTradeQueryModel;
+import com.alipay.api.request.AlipayTradeQueryRequest;
+import com.alipay.api.response.AlipayTradeQueryResponse;
+import com.superd.service.IOrderService;
+import lombok.extern.slf4j.Slf4j;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+import javax.annotation.Resource;
+import java.util.List;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 检测未接收到或未正确处理的支付回调通知
+ * @create 2024-09-30 09:59
+ */
+@Slf4j
+@Component()
+public class NoPayNotifyOrderJob {
+
+    @Resource
+    private IOrderService orderService;
+    @Resource
+    private AlipayClient alipayClient;
+
+    @Scheduled(cron = "0/3 * * * * ?")
+    public void exec() {
+        try {
+            log.info("任务；检测未接收到或未正确处理的支付回调通知");
+            List<String> orderIds = orderService.queryNoPayNotifyOrder();
+            if (null == orderIds || orderIds.isEmpty()) return;
+
+            for (String orderId : orderIds) {
+                AlipayTradeQueryRequest request = new AlipayTradeQueryRequest();
+                AlipayTradeQueryModel bizModel = new AlipayTradeQueryModel();
+                bizModel.setOutTradeNo(orderId);
+                request.setBizModel(bizModel);
+
+                AlipayTradeQueryResponse alipayTradeQueryResponse = alipayClient.execute(request);
+                String code = alipayTradeQueryResponse.getCode();
+                // 判断状态码
+                if ("10000".equals(code)) {
+                    orderService.changeOrderPaySuccess(orderId);
+                }
+            }
+        } catch (Exception e) {
+            log.error("检测未接收到或未正确处理的支付回调通知失败", e);
+        }
+    }
+
+}
diff --git a/small-pay-controller/src/main/java/com/superd/job/TimeoutCloseOrderJob.java b/small-pay-controller/src/main/java/com/superd/job/TimeoutCloseOrderJob.java
new file mode 100644
index 0000000..898bd56
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/job/TimeoutCloseOrderJob.java
@@ -0,0 +1,41 @@
+package com.superd.job;
+
+import com.superd.service.IOrderService;
+import lombok.extern.slf4j.Slf4j;
+import org.springframework.scheduling.annotation.Scheduled;
+import org.springframework.stereotype.Component;
+
+import javax.annotation.Resource;
+import java.util.List;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 超时关单
+ * @create 2024-09-30 09:59
+ */
+@Slf4j
+@Component()
+public class TimeoutCloseOrderJob {
+
+    @Resource
+    private IOrderService orderService;
+
+    @Scheduled(cron = "0 0/10 * * * ?")
+    public void exec() {
+        try {
+            log.info("任务；超时30分钟订单关闭");
+            List<String> orderIds = orderService.queryTimeoutCloseOrderList();
+            if (null == orderIds || orderIds.isEmpty()) {
+                log.info("定时任务，超时30分钟订单关闭，暂无超时未支付订单 orderIds is null");
+                return;
+            }
+            for (String orderId : orderIds) {
+                boolean status = orderService.changeOrderClose(orderId);
+                log.info("定时任务，超时30分钟订单关闭 orderId: {} status：{}", orderId, status);
+            }
+        } catch (Exception e) {
+            log.error("定时任务，超时15分钟订单关闭失败", e);
+        }
+    }
+
+}
diff --git a/small-pay-controller/src/main/java/com/superd/listener/OrderPaySuccessListener.java b/small-pay-controller/src/main/java/com/superd/listener/OrderPaySuccessListener.java
new file mode 100644
index 0000000..4fc53ad
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/listener/OrderPaySuccessListener.java
@@ -0,0 +1,21 @@
+package com.superd.listener;
+
+import com.google.common.eventbus.Subscribe;
+import lombok.extern.slf4j.Slf4j;
+import org.springframework.stereotype.Component;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 支付成功回调消息
+ * @create 2024-09-30 09:52
+ */
+@Slf4j
+@Component
+public class OrderPaySuccessListener {
+
+    @Subscribe
+    public void handleEvent(String paySuccessMessage) {
+        log.info("收到支付成功消息，可以做接下来的事情，如；发货、充值、开户员、返利 {}", paySuccessMessage);
+    }
+
+}
diff --git a/small-pay-controller/src/main/resources/application-dev.yml b/small-pay-controller/src/main/resources/application-dev.yml
index 6777a0c..4434ad7 100644
--- a/small-pay-controller/src/main/resources/application-dev.yml
+++ b/small-pay-controller/src/main/resources/application-dev.yml
@@ -49,7 +49,7 @@ alipay:
   merchant_private_key: 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
   alipay_public_key: MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAnOHAalNHcXIAHsPxpjMcvp8VudUEIml2gWADmzJakBfn3RzSWtqUbRh2u/z1b9xWjG+NzqYUxUM0bANkgE4omcl6Q0T4H6U4P0hsn+Wcsa3ZOLToBk+XwRnW6GJj0IppR2kVsqVMqJjVD+UcRPX8XpUy4U+d2q9FmWvPsGUlC3XmM5/nRUrOzA+CoEvmz2Q/c2f8/tRFHLdvRM6peGk6Y2xtCCPe/8cBm3X97g53rFdyRkjflc2UmqlU6dbzBvJk6u9mX+1mKK2fvEuPlDx3IB5L7JXo2XGjwfNJ41z20/h9TbIEheFYAQ77i+zdw0NcORT8MiR0kZm4aXMMJEmTUQIDAQAB/T3EZTHtQIGwF705Yrq1bd63l70iTfkrS0Ry9f72SDZEBBLllXfFo+otChwRRN+UXDd8X/bplV3/cbRncV5yWRnHHCgzQiwpH3ilS+sOmMfdfac0bi/xB7HIU6nUX04VCjAR7itSr0OmU8HC6p20Ubvjs45R6VuR7FMI+OahCd3LDe/ayelScfQ4zavruk4HGx3TDH4hLDA3N+xid5Cu5erLDPHtFXfnQHI4n/opQaXo5wIDAQAB
 #  todo 需要修改为自己地址
-  notify_url: http://xfg-studio.natapp1.cc/api/v1/alipay/pay_notify
+  notify_url: http://m4e393d6.natappfree.cc/api/v1/alipay/pay_notify
   return_url: https://gaga.plus
   gatewayUrl: https://openapi-sandbox.dl.alipaydev.com/gateway.do
 
diff --git a/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml b/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml
index 46278bb..572f01e 100644
--- a/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml
+++ b/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml
@@ -36,4 +36,28 @@
         limit 1
     </select>
 
+    <update id="changeOrderPaySuccess" parameterType="com.superd.domain.po.PayOrder">
+        update pay_order set status = #{status}, pay_time = now(), update_time = now()
+        where order_id = #{orderId}
+    </update>
+
+    <update id="changeOrderClose" parameterType="java.lang.String">
+        update pay_order set status = 'CLOSE', pay_time = now(), update_time = now()
+        where order_id = #{orderId}
+    </update>
+
+    <select id="queryTimeoutCloseOrderList" parameterType="java.lang.String" resultType="java.lang.String">
+        SELECT order_id as orderId FROM pay_order
+        WHERE status = 'PAY_WAIT' AND NOW() >= order_time + INTERVAL 30 MINUTE
+        ORDER BY id ASC
+        LIMIT 50
+    </select>
+
+    <select id="queryNoPayNotifyOrder" parameterType="java.lang.String" resultType="java.lang.String">
+        SELECT order_id as orderId FROM pay_order
+        WHERE status = 'PAY_WAIT' AND NOW() >= order_time + INTERVAL 1 MINUTE
+        ORDER BY id ASC
+        LIMIT 10
+    </select>
+
 </mapper>
diff --git a/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java b/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java
index 597a4c0..91b70b8 100644
--- a/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java
+++ b/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java
@@ -4,6 +4,8 @@ package com.superd.dao;
 import com.superd.domain.po.PayOrder;
 import org.apache.ibatis.annotations.Mapper;
 
+import java.util.List;
+
 @Mapper
 public interface IOrderDao {
 
@@ -12,4 +14,12 @@ public interface IOrderDao {
     PayOrder queryUnPayOrder(PayOrder payOrder);
 
     void updateOrderPayInfo(PayOrder payOrder);
+
+    void changeOrderPaySuccess(PayOrder order);
+
+    List<String> queryNoPayNotifyOrder();
+
+    List<String> queryTimeoutCloseOrderList();
+
+    boolean changeOrderClose(String orderId);
 }
diff --git a/small-pay-service/src/main/java/com/superd/service/IOrderService.java b/small-pay-service/src/main/java/com/superd/service/IOrderService.java
index 892e3fe..afcf4ba 100644
--- a/small-pay-service/src/main/java/com/superd/service/IOrderService.java
+++ b/small-pay-service/src/main/java/com/superd/service/IOrderService.java
@@ -3,6 +3,8 @@ package com.superd.service;
 import com.superd.domain.req.ShopCartReq;
 import com.superd.domain.res.PayOrderRes;
 
+import java.util.List;
+
 /**
  * @author Fuzhengwei bugstack.cn @小傅哥
  * @description 订单服务接口
@@ -12,4 +14,12 @@ public interface IOrderService {
 
     PayOrderRes createOrder(ShopCartReq shopCartReq) throws Exception;
 
+    void changeOrderPaySuccess(String orderId);
+
+    List<String> queryNoPayNotifyOrder();
+
+    List<String> queryTimeoutCloseOrderList();
+
+    boolean changeOrderClose(String orderId);
+
 }
diff --git a/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java b/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java
index bafa03c..d29415f 100644
--- a/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java
+++ b/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java
@@ -1,9 +1,11 @@
 package com.superd.service.impl;
 
+import com.alibaba.fastjson.JSON;
 import com.alibaba.fastjson.JSONObject;
 import com.alipay.api.AlipayApiException;
 import com.alipay.api.AlipayClient;
 import com.alipay.api.request.AlipayTradePagePayRequest;
+import com.google.common.eventbus.EventBus;
 import com.superd.common.constants.Constants;
 import com.superd.dao.IOrderDao;
 import com.superd.domain.po.PayOrder;
@@ -20,7 +22,9 @@ import org.springframework.stereotype.Service;
 
 import javax.annotation.Resource;
 import java.math.BigDecimal;
+import java.util.Collections;
 import java.util.Date;
+import java.util.List;
 
 /**
  * @author Fuzhengwei bugstack.cn @小傅哥
@@ -41,6 +45,8 @@ public class OrderServiceImpl implements IOrderService {
     private String notifyUrl;
     @Value("${alipay.return_url}")
     private String returnUrl;
+    @Autowired
+    private EventBus eventBus;
 
     @Override
     public PayOrderRes createOrder(ShopCartReq shopCartReq) throws Exception {
@@ -87,6 +93,31 @@ public class OrderServiceImpl implements IOrderService {
                 .build();
     }
 
+    @Override
+    public void changeOrderPaySuccess(String orderId) {
+        PayOrder payOrderReq = new PayOrder();
+        payOrderReq.setOrderId(orderId);
+        payOrderReq.setStatus(Constants.OrderStatusEnum.PAY_SUCCESS.getCode());
+        orderDao.changeOrderPaySuccess(payOrderReq);
+
+        eventBus.post(JSON.toJSONString(payOrderReq));
+    }
+
+    @Override
+    public List<String> queryNoPayNotifyOrder() {
+        return orderDao.queryNoPayNotifyOrder();
+    }
+
+    @Override
+    public List<String> queryTimeoutCloseOrderList() {
+        return orderDao.queryTimeoutCloseOrderList();
+    }
+
+    @Override
+    public boolean changeOrderClose(String orderId) {
+        return orderDao.changeOrderClose(orderId);
+    }
+
     private PayOrder doPrepayOrder(String productId, String productName, String orderId, BigDecimal totalAmount) throws AlipayApiException {
         AlipayTradePagePayRequest request = new AlipayTradePagePayRequest();
         request.setNotifyUrl(notifyUrl);
```
