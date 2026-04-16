# AI 代码评审记录

- 评审工程: `super-d-coder/small-pay-mvc`
- 来源分支: `master`
- 提交备注: feat: 实现订单功能
- 提交SHA: `60bce177`
- 生成时间: 2026-04-16 19:45:44 CST

## 评审结论

代码评审发现以下问题：

### 1. 安全隐患：明文密码
**错误位置**：`small-pay-controller/src/main/resources/application-dev.yml`
**错误描述**：配置文件中数据库密码直接明文编写，存在安全风险。
**修改建议**：建议使用环境变量或配置加密方式管理敏感信息。
**修改后代码**：
```yaml
  datasource:
    username: root
    password: ${DB_PASSWORD:1190670265} # 建议使用环境变量
```

### 2. 配置无效：连接池属性不匹配
**错误位置**：`small-pay-controller/src/main/resources/application-dev.yml`
**错误描述**：`initialSize`、`maxActive` 等配置是 Druid 连接池特有的属性。Spring Boot 默认使用 HikariCP，这些配置将不会生效，会导致连接池配置失效。
**修改建议**：若未引入 Druid 依赖，需将配置改为 HikariCP 标准格式。
**修改后代码**：
```yaml
  datasource:
    username: root
    password: ${DB_PASSWORD:1190670265}
    url: jdbc:mysql://127.0.0.1:3306/s-pay-mall?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&serverTimezone=UTC&useSSL=true
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 60000
      max-lifetime: 60000
      connection-timeout: 60000
```

### 3. 业务逻辑错误：缺少 return 语句
**错误位置**：`small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java`
**错误描述**：在 `else if` 分支判断订单状态为 `CREATE` 时，代码块内没有 `return` 语句。程序会继续向下执行第 2 步“查询商品 & 创建订单”，导致重复下单或数据错误。
**修改建议**：在处理 `CREATE` 状态的逻辑分支中添加返回值，阻止后续执行。
**修改后代码**：
```java
        } else if (null != unpaidOrder && Constants.OrderStatusEnum.CREATE.getCode().equals(unpaidOrder.getStatus())){
            // todo 调用支付宝沙箱创建订单
            // 修复：必须返回，否则会继续执行创建新订单逻辑
            return PayOrderRes.builder()
                    .orderId(unpaidOrder.getOrderId())
                    .payUrl(unpaidOrder.getPayUrl())
                    .build();
        }
```

### 总结
代码存在安全隐患（明文密码）、配置兼容性问题（连接池配置无效）以及严重的业务逻辑漏洞（缺少 return 导致逻辑穿透）。建议修复后再进行合并。

## 代码差异

```diff
diff --git a/small-pay-controller/pom.xml b/small-pay-controller/pom.xml
index c4ea52c..a9e1a94 100644
--- a/small-pay-controller/pom.xml
+++ b/small-pay-controller/pom.xml
@@ -33,17 +33,17 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-configuration-processor</artifactId>
         </dependency>
-        <!--        <dependency>-->
-        <!--            <groupId>org.mybatis.spring.boot</groupId>-->
-        <!--            <artifactId>mybatis-spring-boot-starter</artifactId>-->
-        <!--        </dependency>-->
-        <!-- # 多数据源路由配置
-             # mysql 5.x driver-class-name: com.mysql.jdbc.Driver    mysql-connector-java 5.1.34
-             # mysql 8.x driver-class-name: com.mysql.cj.jdbc.Driver mysql-connector-java 8.0.22-->
-        <!--        <dependency>-->
-        <!--            <groupId>mysql</groupId>-->
-        <!--            <artifactId>mysql-connector-java</artifactId>-->
-        <!--        </dependency>-->
+        <dependency>
+            <groupId>org.mybatis.spring.boot</groupId>
+            <artifactId>mybatis-spring-boot-starter</artifactId>
+        </dependency>
+        <!--# 多数据源路由配置
+        # mysql 5.x driver-class-name: com.mysql.jdbc.Driver mysql-connector-java 5.1.34
+        # mysql 8.x driver-class-name: com.mysql.cj.jdbc.Driver mysql-connector-java 8.0.22-->
+        <dependency>
+            <groupId>mysql</groupId>
+            <artifactId>mysql-connector-java</artifactId>
+        </dependency>
         <dependency>
             <groupId>com.alibaba</groupId>
             <artifactId>fastjson</artifactId>
diff --git a/small-pay-controller/src/main/resources/application-dev.yml b/small-pay-controller/src/main/resources/application-dev.yml
index 01d873e..cdb11f7 100644
--- a/small-pay-controller/src/main/resources/application-dev.yml
+++ b/small-pay-controller/src/main/resources/application-dev.yml
@@ -14,32 +14,35 @@ spring:
     database: 1
     password: 1190670265
 
+  datasource:
+    username: root
+    password: 1190670265
+    url: jdbc:mysql://127.0.0.1:3306/s-pay-mall?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&serverTimezone=UTC&useSSL=true
+    driver-class-name: com.mysql.cj.jdbc.Driver
+    initialSize: 5
+    minIdle: 5
+    maxActive: 20
+    maxWait: 60000
+    timeBetweenEvictionRunsMillis: 60000
+    validationQuery: SELECT 1
+    testWhileIdle: true
+    testOnBorrow: false
+    testOnReturn: false
+    poolPreparedStatements: true
+    maxPoolPreparedStatementPerConnectionSize: 20
+    filters: stat
+
 weixin:
   config:
     app-id: wx2d160f15c419f7b3
     app-secret: e5a76e38af434f706a20389fa6eca496
     template_id: Lv7fSJ7sYuvyEMx5lF_o_VsfOKyYDVm0TJe-s85JVho
     openid: o7Oox3ArEQmsNnVUiWi8irX9Uqww
+mybatis:
+  mapper-locations: classpath*:mybatis/mapper/*.xml
+  config-location: classpath:mybatis/config/mybatis-config.xml
 
 
-#spring:
-#  datasource:
-#    username: root
-#    password: 123456
-#    url: jdbc:mysql://127.0.0.1:3306/s-pay-mall?useUnicode=true&characterEncoding=utf8&autoReconnect=true&zeroDateTimeBehavior=convertToNull&serverTimezone=UTC&useSSL=true
-#    driver-class-name: com.mysql.cj.jdbc.Driver
-#    initialSize: 5
-#    minIdle: 5
-#    maxActive: 20
-#    maxWait: 60000
-#    timeBetweenEvictionRunsMillis: 60000
-#    validationQuery: SELECT 1
-#    testWhileIdle: true
-#    testOnBorrow: false
-#    testOnReturn: false
-#    poolPreparedStatements: true
-#    maxPoolPreparedStatementPerConnectionSize: 20
-#    filters: stat
 
 # 日志
 logging:
diff --git a/small-pay-controller/src/main/resources/application.yml b/small-pay-controller/src/main/resources/application.yml
index 802f390..863b420 100644
--- a/small-pay-controller/src/main/resources/application.yml
+++ b/small-pay-controller/src/main/resources/application.yml
@@ -1,6 +1,6 @@
 spring:
   config:
-    name: s-pay-mall-mvc
+    name: small-pay-mvc
   profiles:
     active: dev
 
diff --git a/small-pay-controller/src/main/resources/mybatis/config/mybatis-config.xml b/small-pay-controller/src/main/resources/mybatis/config/mybatis-config.xml
new file mode 100644
index 0000000..8374e01
--- /dev/null
+++ b/small-pay-controller/src/main/resources/mybatis/config/mybatis-config.xml
@@ -0,0 +1,10 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<!DOCTYPE configuration
+        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
+        "http://mybatis.org/dtd/mybatis-3-config.dtd">
+<configuration>
+    <!-- 暂时未使用 文档：https://mybatis.org/mybatis-3/zh/configuration.html#typeAliases -->
+    <typeAliases>
+
+    </typeAliases>
+</configuration>
\ No newline at end of file
diff --git a/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml b/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml
new file mode 100644
index 0000000..628976c
--- /dev/null
+++ b/small-pay-controller/src/main/resources/mybatis/mapper/pay_order_mapper.xml
@@ -0,0 +1,35 @@
+<?xml version="1.0" encoding="UTF-8"?>
+<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
+<mapper namespace="com.superd.dao.IOrderDao">
+
+    <resultMap id="dataMap" type="com.superd.domain.po.PayOrder">
+        <id column="id" property="id"/>
+        <result column="user_id" property="userId"/>
+        <result column="product_id" property="productId"/>
+        <result column="product_name" property="productName"/>
+        <result column="order_id" property="orderId"/>
+        <result column="order_time" property="orderTime"/>
+        <result column="total_amount" property="totalAmount"/>
+        <result column="status" property="status"/>
+        <result column="pay_url" property="payUrl"/>
+        <result column="pay_time" property="payTime"/>
+        <result column="create_time" property="createTime"/>
+        <result column="update_time" property="updateTime"/>
+    </resultMap>
+
+    <insert id="insert" parameterType="com.superd.domain.po.PayOrder">
+        insert into pay_order(user_id, product_id, product_name, order_id, order_time,
+        total_amount, status, create_time, update_time)
+        values(#{userId}, #{productId}, #{productName}, #{orderId}, #{orderTime},
+        #{totalAmount}, #{status}, now(), now())
+    </insert>
+
+    <select id="queryUnPayOrder" parameterType="com.superd.domain.po.PayOrder" resultMap="dataMap">
+        select product_id, product_name, order_id, order_time, total_amount, status, pay_url
+        from pay_order
+        where user_id = #{userId} and product_id = #{productId}
+        order by id desc
+        limit 1
+    </select>
+
+</mapper>
diff --git a/small-pay-controller/src/test/java/com/superd/test/service/OrderServiceTest.java b/small-pay-controller/src/test/java/com/superd/test/service/OrderServiceTest.java
new file mode 100644
index 0000000..8bb49d1
--- /dev/null
+++ b/small-pay-controller/src/test/java/com/superd/test/service/OrderServiceTest.java
@@ -0,0 +1,33 @@
+package com.superd.test.service;
+
+import com.alibaba.fastjson.JSON;
+import com.superd.domain.req.ShopCartReq;
+import com.superd.domain.res.PayOrderRes;
+import com.superd.service.IOrderService;
+import lombok.extern.slf4j.Slf4j;
+import org.junit.Test;
+import org.junit.runner.RunWith;
+import org.springframework.boot.test.context.SpringBootTest;
+import org.springframework.test.context.junit4.SpringRunner;
+
+import javax.annotation.Resource;
+
+@Slf4j
+@RunWith(SpringRunner.class)
+@SpringBootTest
+public class OrderServiceTest {
+
+    @Resource
+    private IOrderService orderService;
+
+    @Test
+    public void test_createOrder() throws Exception {
+        ShopCartReq shopCartReq = new ShopCartReq();
+        shopCartReq.setUserId("xiaofuge");
+        shopCartReq.setProductId("10001");
+        PayOrderRes payOrderRes = orderService.createOrder(shopCartReq);
+        log.info("请求参数:{}", JSON.toJSONString(shopCartReq));
+        log.info("测试结果:{}", JSON.toJSONString(payOrderRes));
+    }
+
+}
diff --git a/small-pay-dao/pom.xml b/small-pay-dao/pom.xml
index 8c6010b..51ce42d 100644
--- a/small-pay-dao/pom.xml
+++ b/small-pay-dao/pom.xml
@@ -18,10 +18,10 @@
     </properties>
 
     <dependencies>
-        <!--        <dependency>-->
-        <!--            <groupId>org.mybatis.spring.boot</groupId>-->
-        <!--            <artifactId>mybatis-spring-boot-starter</artifactId>-->
-        <!--        </dependency>-->
+        <dependency>
+            <groupId>org.mybatis.spring.boot</groupId>
+            <artifactId>mybatis-spring-boot-starter</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.projectlombok</groupId>
             <artifactId>lombok</artifactId>
diff --git a/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java b/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java
new file mode 100644
index 0000000..4ad4364
--- /dev/null
+++ b/small-pay-dao/src/main/java/com/superd/dao/IOrderDao.java
@@ -0,0 +1,14 @@
+package com.superd.dao;
+
+
+import com.superd.domain.po.PayOrder;
+import org.apache.ibatis.annotations.Mapper;
+
+@Mapper
+public interface IOrderDao {
+
+    void insert(PayOrder payOrder);
+
+    PayOrder queryUnPayOrder(PayOrder payOrder);
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/po/PayOrder.java b/small-pay-domain/src/main/java/com/superd/domain/po/PayOrder.java
new file mode 100644
index 0000000..2215d90
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/po/PayOrder.java
@@ -0,0 +1,30 @@
+package com.superd.domain.po;
+
+import lombok.AllArgsConstructor;
+import lombok.Builder;
+import lombok.Data;
+import lombok.NoArgsConstructor;
+
+import java.math.BigDecimal;
+import java.util.Date;
+
+@Data
+@Builder
+@AllArgsConstructor
+@NoArgsConstructor
+public class PayOrder {
+
+    private Long id;
+    private String userId;
+    private String productId;
+    private String productName;
+    private String orderId;
+    private Date orderTime;
+    private BigDecimal totalAmount;
+    private String status;
+    private String payUrl;
+    private Date payTime;
+    private Date createTime;
+    private Date updateTime;
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/req/ShopCartReq.java b/small-pay-domain/src/main/java/com/superd/domain/req/ShopCartReq.java
new file mode 100644
index 0000000..5646159
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/req/ShopCartReq.java
@@ -0,0 +1,18 @@
+package com.superd.domain.req;
+
+import lombok.AllArgsConstructor;
+import lombok.Builder;
+import lombok.Data;
+import lombok.NoArgsConstructor;
+
+@Data
+@Builder
+@AllArgsConstructor
+@NoArgsConstructor
+public class ShopCartReq {
+
+    private String userId;
+
+    private String productId;
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/res/PayOrderRes.java b/small-pay-domain/src/main/java/com/superd/domain/res/PayOrderRes.java
new file mode 100644
index 0000000..8a1e75f
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/res/PayOrderRes.java
@@ -0,0 +1,20 @@
+package com.superd.domain.res;
+
+import com.superd.common.constants.Constants;
+import lombok.AllArgsConstructor;
+import lombok.Builder;
+import lombok.Data;
+import lombok.NoArgsConstructor;
+
+@Data
+@Builder
+@AllArgsConstructor
+@NoArgsConstructor
+public class PayOrderRes {
+
+    private String userId;
+    private String orderId;
+    private String payUrl;
+    private Constants.OrderStatusEnum orderStatusEnum;
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/vo/ProductVO.java b/small-pay-domain/src/main/java/com/superd/domain/vo/ProductVO.java
new file mode 100644
index 0000000..d9ca25e
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/vo/ProductVO.java
@@ -0,0 +1,19 @@
+package com.superd.domain.vo;
+
+import lombok.Data;
+
+import java.math.BigDecimal;
+
+@Data
+public class ProductVO {
+
+    /** 商品ID */
+    private String productId;
+    /** 商品名称 */
+    private String productName;
+    /** 商品描述 */
+    private String productDesc;
+    /** 商品价格 */
+    private BigDecimal price;
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/po/WeixinTemplateMessageVO.java b/small-pay-domain/src/main/java/com/superd/domain/vo/WeixinTemplateMessageVO.java
similarity index 98%
rename from small-pay-domain/src/main/java/com/superd/domain/po/WeixinTemplateMessageVO.java
rename to small-pay-domain/src/main/java/com/superd/domain/vo/WeixinTemplateMessageVO.java
index dacf8b0..8033124 100644
--- a/small-pay-domain/src/main/java/com/superd/domain/po/WeixinTemplateMessageVO.java
+++ b/small-pay-domain/src/main/java/com/superd/domain/vo/WeixinTemplateMessageVO.java
@@ -1,4 +1,4 @@
-package com.superd.domain.po;
+package com.superd.domain.vo;
 
 import lombok.Getter;
 import lombok.Setter;
diff --git a/small-pay-service/src/main/java/com/superd/service/IOrderService.java b/small-pay-service/src/main/java/com/superd/service/IOrderService.java
new file mode 100644
index 0000000..892e3fe
--- /dev/null
+++ b/small-pay-service/src/main/java/com/superd/service/IOrderService.java
@@ -0,0 +1,15 @@
+package com.superd.service;
+
+import com.superd.domain.req.ShopCartReq;
+import com.superd.domain.res.PayOrderRes;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 订单服务接口
+ * @create 2024-09-29 09:43
+ */
+public interface IOrderService {
+
+    PayOrderRes createOrder(ShopCartReq shopCartReq) throws Exception;
+
+}
diff --git a/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java b/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java
new file mode 100644
index 0000000..04adbe1
--- /dev/null
+++ b/small-pay-service/src/main/java/com/superd/service/impl/OrderServiceImpl.java
@@ -0,0 +1,74 @@
+package com.superd.service.impl;
+
+import com.superd.common.constants.Constants;
+import com.superd.dao.IOrderDao;
+import com.superd.domain.po.PayOrder;
+import com.superd.domain.req.ShopCartReq;
+import com.superd.domain.res.PayOrderRes;
+import com.superd.domain.vo.ProductVO;
+import com.superd.service.IOrderService;
+import com.superd.service.rpc.ProductRPC;
+import lombok.extern.slf4j.Slf4j;
+import org.apache.commons.lang3.RandomStringUtils;
+import org.springframework.stereotype.Service;
+
+import javax.annotation.Resource;
+import java.util.Date;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 订单服务
+ * @create 2024-09-29 09:47
+ */
+@Slf4j
+@Service
+public class OrderServiceImpl implements IOrderService {
+
+    @Resource
+    private IOrderDao orderDao;
+    @Resource
+    private ProductRPC productRPC;
+
+    @Override
+    public PayOrderRes createOrder(ShopCartReq shopCartReq) throws Exception {
+        // 1. 查询当前用户是否存在未支付订单或掉单订单
+        PayOrder payOrderReq = new PayOrder();
+        payOrderReq.setUserId(shopCartReq.getUserId());
+        payOrderReq.setProductId(shopCartReq.getProductId());
+
+        PayOrder unpaidOrder = orderDao.queryUnPayOrder(payOrderReq);
+
+        if (null != unpaidOrder && Constants.OrderStatusEnum.PAY_WAIT.getCode().equals(unpaidOrder.getStatus())) {
+            log.info("创建订单-存在，已存在未支付订单。userId:{} productId:{} orderId:{}", shopCartReq.getUserId(), shopCartReq.getProductId(), unpaidOrder.getOrderId());
+            return PayOrderRes.builder()
+                    .orderId(unpaidOrder.getOrderId())
+                    .payUrl(unpaidOrder.getPayUrl())
+                    .build();
+        } else if (null != unpaidOrder && Constants.OrderStatusEnum.CREATE.getCode().equals(unpaidOrder.getStatus())){
+            // todo 调用支付宝沙箱创建订单
+        }
+
+        // 2. 查询商品 & 创建订单
+        ProductVO productVO = productRPC.queryProductByProductId(shopCartReq.getProductId());
+        String orderId = RandomStringUtils.randomNumeric(16);
+        orderDao.insert(PayOrder.builder()
+                        .userId(shopCartReq.getUserId())
+                        .productId(shopCartReq.getProductId())
+                        .productName(productVO.getProductName())
+                        .orderId(orderId)
+                        .totalAmount(productVO.getPrice())
+                        .orderTime(new Date())
+                        .status(Constants.OrderStatusEnum.CREATE.getCode())
+                .build());
+
+        // 3. 创建支付单 todo
+
+
+        return PayOrderRes.builder()
+                .orderId(orderId)
+                .payUrl("暂无")
+                .build();
+    }
+
+
+}
diff --git a/small-pay-service/src/main/java/com/superd/service/impl/WeixinLoginServiceImpl.java b/small-pay-service/src/main/java/com/superd/service/impl/WeixinLoginServiceImpl.java
index 4b2b96a..4524d1a 100644
--- a/small-pay-service/src/main/java/com/superd/service/impl/WeixinLoginServiceImpl.java
+++ b/small-pay-service/src/main/java/com/superd/service/impl/WeixinLoginServiceImpl.java
@@ -1,6 +1,6 @@
 package com.superd.service.impl;
 
-import com.superd.domain.po.WeixinTemplateMessageVO;
+import com.superd.domain.vo.WeixinTemplateMessageVO;
 import com.superd.domain.req.WeixinAccessTokenReq;
 import com.superd.domain.req.WeixinQrCodeReq;
 import com.superd.domain.res.WeixinQrCodeRes;
diff --git a/small-pay-service/src/main/java/com/superd/service/rpc/ProductRPC.java b/small-pay-service/src/main/java/com/superd/service/rpc/ProductRPC.java
new file mode 100644
index 0000000..90d55a9
--- /dev/null
+++ b/small-pay-service/src/main/java/com/superd/service/rpc/ProductRPC.java
@@ -0,0 +1,20 @@
+package com.superd.service.rpc;
+
+import com.superd.domain.vo.ProductVO;
+import org.springframework.stereotype.Service;
+
+import java.math.BigDecimal;
+
+@Service
+public class ProductRPC {
+
+    public ProductVO queryProductByProductId(String productId){
+        ProductVO productVO = new ProductVO();
+        productVO.setProductId(productId);
+        productVO.setProductName("测试商品");
+        productVO.setProductDesc("这是一个测试商品");
+        productVO.setPrice(new BigDecimal("1.68"));
+        return productVO;
+    }
+
+}
diff --git a/small-pay-service/src/main/java/com/superd/service/weixin/IWeixinApiService.java b/small-pay-service/src/main/java/com/superd/service/weixin/IWeixinApiService.java
index cccec21..f3eb61c 100644
--- a/small-pay-service/src/main/java/com/superd/service/weixin/IWeixinApiService.java
+++ b/small-pay-service/src/main/java/com/superd/service/weixin/IWeixinApiService.java
@@ -1,6 +1,6 @@
 package com.superd.service.weixin;
 
-import com.superd.domain.po.WeixinTemplateMessageVO;
+import com.superd.domain.vo.WeixinTemplateMessageVO;
 import com.superd.domain.req.WeixinAccessTokenReq;
 import com.superd.domain.req.WeixinQrCodeReq;
 import com.superd.domain.res.WeixinQrCodeRes;
```
