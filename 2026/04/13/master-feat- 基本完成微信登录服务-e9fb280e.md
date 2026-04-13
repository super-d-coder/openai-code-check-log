# AI 代码评审记录

- 评审工程: `super-d-coder/small-pay-mvc`
- 来源分支: `master`
- 提交备注: feat: 基本完成微信登录服务
- 提交SHA: `e9fb280e`
- 生成时间: 2026-04-13 19:21:58 CST

## 评审结论

根据提供的代码差异，以下是代码评审结果：

### 1. 错误提示：包名引用错误
**错误位置**：`small-pay-common/src/main/java/com/superd/common/exception/AppException.java` 第 50 行
**错误代码片段**：
```java
return "cn.bugstack.common.exception.AppException{" +
```
**修改建议**：`toString` 方法中的包名应与当前项目包名 `com.superd` 保持一致，而不是复制来的 `cn.bugstack`。
**修改后的代码片段**：
```java
return "com.superd.common.exception.AppException{" +
```

### 2. 错误提示：枚举类设计缺陷
**错误位置**：`small-pay-common/src/main/java/com/superd/common/constants/Constants.java` 第 18-32 行
**错误代码片段**：
```java
@AllArgsConstructor
@NoArgsConstructor
@Getter
public enum ResponseCode {
    SUCCESS("0000", "调用成功"),
    // ...
    private String code;
    private String info;
}
```
**修改建议**：枚举类不应有 `@NoArgsConstructor` 且字段不应可变，这会导致枚举实例状态被非法修改。应移除该注解并将字段改为 `final`。
**修改后的代码片段**：
```java
@AllArgsConstructor
@Getter
public enum ResponseCode {
    SUCCESS("0000", "调用成功"),
    UN_ERROR("0001", "调用失败"),
    ILLEGAL_PARAMETER("0002", "非法参数"),
    NO_LOGIN("0003", "未登录"),
    ;

    private final String code;
    private final String info;

}
```

### 3. 错误提示：枚举类包含 Setter 方法
**错误位置**：`small-pay-domain/src/main/java/com/superd/domain/po/WeixinTemplateMessageVO.java` 第 64-70 行
**错误代码片段**：
```java
public void setCode(String code) {
    this.code = code;
}

public void setDesc(String desc) {
    this.desc = desc;
}
```
**修改建议**：枚举实例是单例，不应包含 Setter 方法，防止运行时修改枚举常量属性。应删除 Setter 方法。
**修改后的代码片段**：
```java
// 删除 setCode 和 setDesc 方法
```

### 4. 错误提示：XXE 安全漏洞风险
**错误位置**：`small-pay-common/src/main/java/com/superd/common/weixin/XmlUtil.java` 第 31 行
**错误代码片段**：
```java
SAXReader reader = new SAXReader();
```
**修改建议**：`SAXReader` 默认配置可能导致 XML 外部实体注入（XXE）攻击，需禁用外部实体解析。
**修改后的代码片段**：
```java
SAXReader reader = new SAXReader();
reader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
reader.setFeature("http://xml.org/sax/features/external-general-entities", false);
reader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
```

### 5. 错误提示：敏感信息硬编码
**错误位置**：`small-pay-controller/src/main/resources/application-dev.yml` 第 15-16 行
**错误代码片段**：
```yaml
password: 1190670265
# ...
app-secret: e5a76e38af434f706a20389fa6eca496
```
**修改建议**：禁止在代码库中明文提交数据库密码、AppSecret 等敏感信息。建议使用环境变量或配置中心管理。
**修改后的代码片段**：
```yaml
password: ${REDIS_PASSWORD:}
# ...
app-secret: ${WEIXIN_APP_SECRET:}
```

### 6. 错误提示：类定义截断
**错误位置**：`small-pay-domain/src/main/java/com/superd/domain/req/WeixinQrCodeReq.java` 最后一行
**错误代码片段**：
```java
public class WeixinQrCodeRe
```
**修改建议**：文件内容不完整，需补全类名。
**修改后的代码片段**：
```java
public class WeixinQrCodeReq {
    // 类内容
}
```

### 总结
代码存在包名引用错误、枚举设计缺陷（可变性与Setter）、XXE安全漏洞风险、敏感信息泄露及文件内容截断等问题。建议立即修复安全漏洞与代码逻辑错误，并清理敏感配置。

## 代码差异

```diff
diff --git a/small-pay-common/src/main/java/com/superd/common/constants/Constants.java b/small-pay-common/src/main/java/com/superd/common/constants/Constants.java
new file mode 100644
index 0000000..9af7aeb
--- /dev/null
+++ b/small-pay-common/src/main/java/com/superd/common/constants/Constants.java
@@ -0,0 +1,47 @@
+package com.superd.common.constants;
+
+import lombok.AllArgsConstructor;
+import lombok.Getter;
+import lombok.NoArgsConstructor;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 通用的枚举值
+ * @create 2023-07-22 21:24
+ */
+public class Constants {
+
+    public final static String SPLIT = ",";
+
+    @AllArgsConstructor
+    @NoArgsConstructor
+    @Getter
+    public enum ResponseCode {
+        SUCCESS("0000", "调用成功"),
+        UN_ERROR("0001", "调用失败"),
+        ILLEGAL_PARAMETER("0002", "非法参数"),
+        NO_LOGIN("0003", "未登录"),
+        ;
+
+        private String code;
+        private String info;
+
+    }
+
+    @Getter
+    @AllArgsConstructor
+    public enum OrderStatusEnum {
+
+        CREATE("CREATE", "创建完成 - 如果调单了，也会从创建记录重新发起创建支付单"),
+        PAY_WAIT("PAY_WAIT", "等待支付 - 订单创建完成后，创建支付单"),
+        PAY_SUCCESS("PAY_SUCCESS", "支付成功 - 接收到支付回调消息"),
+        DEAL_DONE("DEAL_DONE", "交易完成 - 商品发货完成"),
+        CLOSE("CLOSE", "超时关单 - 超市未支付"),
+        ;
+
+        private final String code;
+        private final String desc;
+
+    }
+
+}
diff --git a/small-pay-common/src/main/java/com/superd/common/exception/AppException.java b/small-pay-common/src/main/java/com/superd/common/exception/AppException.java
new file mode 100644
index 0000000..a84d1e5
--- /dev/null
+++ b/small-pay-common/src/main/java/com/superd/common/exception/AppException.java
@@ -0,0 +1,55 @@
+package com.superd.common.exception;
+
+import lombok.Data;
+import lombok.EqualsAndHashCode;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 应用自定义异常
+ * @create 2024-02-25 12:17
+ */
+@EqualsAndHashCode(callSuper = true)
+@Data
+public class AppException extends RuntimeException {
+
+    private static final long serialVersionUID = 8653090271840061986L;
+
+    /**
+     * 异常码
+     */
+    private String code;
+
+    /**
+     * 异常信息
+     */
+    private String info;
+
+    public AppException(String code) {
+        this.code = code;
+    }
+
+    public AppException(String code, Throwable cause) {
+        this.code = code;
+        super.initCause(cause);
+    }
+
+    public AppException(String code, String message) {
+        this.code = code;
+        this.info = message;
+    }
+
+    public AppException(String code, String message, Throwable cause) {
+        this.code = code;
+        this.info = message;
+        super.initCause(cause);
+    }
+
+    @Override
+    public String toString() {
+        return "cn.bugstack.common.exception.AppException{" +
+                "code='" + code + '\'' +
+                ", info='" + info + '\'' +
+                '}';
+    }
+
+}
\ No newline at end of file
diff --git a/small-pay-common/src/main/java/com/superd/common/response/Response.java b/small-pay-common/src/main/java/com/superd/common/response/Response.java
new file mode 100644
index 0000000..4916763
--- /dev/null
+++ b/small-pay-common/src/main/java/com/superd/common/response/Response.java
@@ -0,0 +1,22 @@
+package com.superd.common.response;
+
+import lombok.AllArgsConstructor;
+import lombok.Builder;
+import lombok.Data;
+import lombok.NoArgsConstructor;
+
+import java.io.Serializable;
+
+@Data
+@Builder
+@NoArgsConstructor
+@AllArgsConstructor
+public class Response<T> implements Serializable {
+
+    private static final long serialVersionUID = 7000723935764546321L;
+
+    private String code;
+    private String info;
+    private T data;
+
+}
diff --git a/small-pay-common/src/main/java/com/superd/common/weixin/MessageTextEntity.java b/small-pay-common/src/main/java/com/superd/common/weixin/MessageTextEntity.java
new file mode 100644
index 0000000..d0ab560
--- /dev/null
+++ b/small-pay-common/src/main/java/com/superd/common/weixin/MessageTextEntity.java
@@ -0,0 +1,129 @@
+package com.superd.common.weixin;
+
+import com.thoughtworks.xstream.annotations.XStreamAlias;
+
+@XStreamAlias("xml")
+public class MessageTextEntity {
+
+    @XStreamAlias("ToUserName")
+    private String toUserName;
+
+    @XStreamAlias("FromUserName")
+    private String fromUserName;
+
+    @XStreamAlias("CreateTime")
+    private String createTime;
+
+    @XStreamAlias("MsgType")
+    private String msgType;
+
+    @XStreamAlias("Event")
+    private String event;
+
+    @XStreamAlias("EventKey")
+    private String eventKey;
+
+    @XStreamAlias("MsgId")
+    private String msgId;
+
+    @XStreamAlias("MsgID")
+    private String msgID;
+
+    @XStreamAlias("Status")
+    private String status;
+
+    @XStreamAlias("Ticket")
+    private String ticket;
+
+    @XStreamAlias("Content")
+    private String content;
+
+    // Getters and Setters
+    public String getToUserName() {
+        return toUserName;
+    }
+
+    public void setToUserName(String toUserName) {
+        this.toUserName = toUserName;
+    }
+
+    public String getFromUserName() {
+        return fromUserName;
+    }
+
+    public void setFromUserName(String fromUserName) {
+        this.fromUserName = fromUserName;
+    }
+
+    public String getCreateTime() {
+        return createTime;
+    }
+
+    public void setCreateTime(String createTime) {
+        this.createTime = createTime;
+    }
+
+    public String getMsgType() {
+        return msgType;
+    }
+
+    public void setMsgType(String msgType) {
+        this.msgType = msgType;
+    }
+
+    public String getEvent() {
+        return event;
+    }
+
+    public void setEvent(String event) {
+        this.event = event;
+    }
+
+    public String getMsgId() {
+        return msgId;
+    }
+
+    public void setMsgId(String msgId) {
+        this.msgId = msgId;
+    }
+
+    public String getStatus() {
+        return status;
+    }
+
+    public void setStatus(String status) {
+        this.status = status;
+    }
+
+    public String getEventKey() {
+        return eventKey;
+    }
+
+    public void setEventKey(String eventKey) {
+        this.eventKey = eventKey;
+    }
+
+    public String getTicket() {
+        return ticket;
+    }
+
+    public void setTicket(String ticket) {
+        this.ticket = ticket;
+    }
+
+    public String getContent() {
+        return content;
+    }
+
+    public void setContent(String content) {
+        this.content = content;
+    }
+
+    public String getMsgID() {
+        return msgID;
+    }
+
+    public void setMsgID(String msgID) {
+        this.msgID = msgID;
+    }
+}
\ No newline at end of file
diff --git a/small-pay-common/src/main/java/com/superd/common/weixin/SignatureUtil.java b/small-pay-common/src/main/java/com/superd/common/weixin/SignatureUtil.java
new file mode 100644
index 0000000..f78b64e
--- /dev/null
+++ b/small-pay-common/src/main/java/com/superd/common/weixin/SignatureUtil.java
@@ -0,0 +1,68 @@
+package com.superd.common.weixin;
+
+import java.security.MessageDigest;
+import java.security.NoSuchAlgorithmException;
+
+public class SignatureUtil {
+    /**
+     * 验证签名
+     */
+    public static boolean check(String token, String signature, String timestamp, String nonce) {
+        String[] arr = new String[]{token, timestamp, nonce};
+        // 将token、timestamp、nonce三个参数进行字典序排序
+        sort(arr);
+        StringBuilder content = new StringBuilder();
+        for (String s : arr) {
+            content.append(s);
+        }
+        MessageDigest md;
+        String tmpStr = null;
+        try {
+            md = MessageDigest.getInstance("SHA-1");
+            // 将三个参数字符串拼接成一个字符串进行sha1加密
+            byte[] digest = md.digest(content.toString().getBytes());
+            tmpStr = byteToStr(digest);
+        } catch (NoSuchAlgorithmException e) {
+            e.printStackTrace();
+        }
+        // 将sha1加密后的字符串可与signature对比，标识该请求来源于微信
+        return tmpStr != null && tmpStr.equals(signature.toUpperCase());
+    }
+
+    /**
+     * 将字节数组转换为十六进制字符串
+     */
+    private static String byteToStr(byte[] byteArray) {
+        StringBuilder strDigest = new StringBuilder();
+        for (byte b : byteArray) {
+            strDigest.append(byteToHexStr(b));
+        }
+        return strDigest.toString();
+    }
+
+    /**
+     * 将字节转换为十六进制字符串
+     */
+    private static String byteToHexStr(byte mByte) {
+        char[] Digit = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'};
+        char[] tempArr = new char[2];
+        tempArr[0] = Digit[(mByte >>> 4) & 0X0F];
+        tempArr[1] = Digit[mByte & 0X0F];
+        return new String(tempArr);
+    }
+
+    /**
+     * 进行字典排序
+     */
+    private static void sort(String[] str) {
+        for (int i = 0; i < str.length - 1; i++) {
+            for (int j = i + 1; j < str.length; j++) {
+                if (str[j].compareTo(str[i]) < 0) {
+                    String temp = str[i];
+                    str[i] = str[j];
+                    str[j] = temp;
+                }
+            }
+        }
+    }
+}
diff --git a/small-pay-common/src/main/java/com/superd/common/weixin/XmlUtil.java b/small-pay-common/src/main/java/com/superd/common/weixin/XmlUtil.java
new file mode 100644
index 0000000..4a263c6
--- /dev/null
+++ b/small-pay-common/src/main/java/com/superd/common/weixin/XmlUtil.java
@@ -0,0 +1,151 @@
+package com.superd.common.weixin;
+
+import com.thoughtworks.xstream.XStream;
+import com.thoughtworks.xstream.core.util.QuickWriter;
+import com.thoughtworks.xstream.io.HierarchicalStreamWriter;
+import com.thoughtworks.xstream.io.xml.DomDriver;
+import com.thoughtworks.xstream.io.xml.PrettyPrintWriter;
+import com.thoughtworks.xstream.io.xml.XppDriver;
+import com.thoughtworks.xstream.security.AnyTypePermission;
+import org.apache.commons.lang3.StringUtils;
+import org.dom4j.Document;
+import org.dom4j.Element;
+import org.dom4j.io.SAXReader;
+
+import javax.servlet.http.HttpServletRequest;
+import java.io.InputStream;
+import java.io.Writer;
+import java.util.*;
+
+public class XmlUtil {
+
+    /**
+     * 解析微信发来的请求(xml)
+     */
+    @SuppressWarnings("unchecked")
+    public static Map<String, String> xmlToMap(HttpServletRequest request) throws Exception {
+        // 从request中取得输入流
+        try (InputStream inputStream = request.getInputStream()) {
+            // 将解析结果存储在HashMap中
+            Map<String, String> map = new HashMap<>();
+            // 读取输入流
+            SAXReader reader = new SAXReader();
+            // 得到xml文档
+            Document document = reader.read(inputStream);
+            // 得到xml根元素
+            Element root = document.getRootElement();
+            // 得到根元素的所有子节点
+            List<Element> elementList = root.elements();
+            // 遍历所有子节点
+            for (Element e : elementList)
+                map.put(e.getName(), e.getText());
+            // 释放资源
+            inputStream.close();
+            return map;
+        }
+    }
+
+    /**
+     * 将map转化成xml响应给微信服务器
+     */
+    static String mapToXML(Map map) {
+        StringBuffer sb = new StringBuffer();
+        sb.append("<xml>");
+        mapToXML2(map, sb);
+        sb.append("</xml>");
+        try {
+            return sb.toString();
+        } catch (Exception e) {
+            return null;
+        }
+    }
+
+    private static void mapToXML2(Map map, StringBuffer sb) {
+        Set set = map.keySet();
+        for (Object o : set) {
+            String key = (String) o;
+            Object value = map.get(key);
+            if (null == value)
+                value = "";
+            if (value.getClass().getName().equals("java.util.ArrayList")) {
+                ArrayList list = (ArrayList) map.get(key);
+                sb.append("<").append(key).append(">");
+                for (Object o1 : list) {
+                    HashMap hm = (HashMap) o1;
+                    mapToXML2(hm, sb);
+                }
+                sb.append("</").append(key).append(">");
+
+            } else {
+                if (value instanceof HashMap) {
+                    sb.append("<").append(key).append(">");
+                    mapToXML2((HashMap) value, sb);
+                    sb.append("</").append(key).append(">");
+                } else {
+                    sb.append("<").append(key).append("><![CDATA[").append(value).append("]]></").append(key).append(">");
+                }
+
+            }
+
+        }
+    }
+
+    public static XStream getMyXStream() {
+        return new XStream(new XppDriver() {
+            @Override
+            public HierarchicalStreamWriter createWriter(Writer out) {
+                return new PrettyPrintWriter(out) {
+                    // 对所有xml节点都增加CDATA标记
+                    boolean cdata = true;
+
+                    @Override
+                    public void startNode(String name, Class clazz) {
+                        super.startNode(name, clazz);
+                    }
+
+                    @Override
+                    protected void writeText(QuickWriter writer, String text) {
+                        if (cdata && !StringUtils.isNumeric(text)) {
+                            writer.write("<![CDATA[");
+                            writer.write(text);
+                            writer.write("]]>");
+                        } else {
+                            writer.write(text);
+                        }
+                    }
+                };
+            }
+        });
+    }
+
+    /**
+     * bean转成微信的xml消息格式
+     */
+    public static String beanToXml(Object object) {
+        XStream xStream = getMyXStream();
+        xStream.alias("xml", object.getClass());
+        xStream.processAnnotations(object.getClass());
+        String xml = xStream.toXML(object);
+        if (!StringUtils.isEmpty(xml)) {
+            return xml;
+        } else {
+            return null;
+        }
+    }
+
+    /**
+     * xml转成bean泛型方法
+     */
+    public static <T> T xmlToBean(String resultXml, Class clazz) {
+        // XStream对象设置默认安全防护，同时设置允许的类
+        XStream stream = new XStream(new DomDriver());
+        stream.addPermission(AnyTypePermission.ANY);
+        XStream.setupDefaultSecurity(stream);
+        stream.allowTypes(new Class[]{clazz});
+        stream.processAnnotations(new Class[]{clazz});
+        stream.setMode(XStream.NO_REFERENCES);
+        stream.alias("xml", clazz);
+        return (T) stream.fromXML(resultXml);
+    }
+
+}
\ No newline at end of file
diff --git a/small-pay-controller/src/main/java/com/superd/config/Retrofit2Config.java b/small-pay-controller/src/main/java/com/superd/config/Retrofit2Config.java
new file mode 100644
index 0000000..6f837f4
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/config/Retrofit2Config.java
@@ -0,0 +1,28 @@
+package com.superd.config;
+
+import com.superd.service.weixin.IWeixinApiService;
+import lombok.extern.slf4j.Slf4j;
+import org.springframework.context.annotation.Bean;
+import org.springframework.context.annotation.Configuration;
+import retrofit2.Retrofit;
+import retrofit2.converter.jackson.JacksonConverterFactory;
+
+@Slf4j
+@Configuration
+public class Retrofit2Config {
+
+    private static final String BASE_URL = "https://api.weixin.qq.com/";
+
+    @Bean
+    public Retrofit retrofit() {
+        return new Retrofit.Builder()
+                .baseUrl(BASE_URL)
+                .addConverterFactory(JacksonConverterFactory.create()).build();
+    }
+
+    @Bean
+    public IWeixinApiService weixinApiService(Retrofit retrofit) {
+        return retrofit.create(IWeixinApiService.class);
+    }
+
+}
diff --git a/small-pay-controller/src/main/java/com/superd/controller/LoginController.java b/small-pay-controller/src/main/java/com/superd/controller/LoginController.java
new file mode 100644
index 0000000..74d314a
--- /dev/null
+++ b/small-pay-controller/src/main/java/com/superd/controller/LoginController.java
@@ -0,0 +1,84 @@
+package com.superd.controller;
+
+import com.superd.common.constants.Constants;
+import com.superd.common.response.Response;
+import com.superd.service.ILoginService;
+import org.springframework.beans.factory.annotation.Value;
+import lombok.extern.slf4j.Slf4j;
+import org.apache.commons.lang3.StringUtils;
+import org.springframework.web.bind.annotation.*;
+
+import javax.annotation.Resource;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 登录服务
+ * @create 2024-09-28 13:58
+ */
+@Slf4j
+@RestController()
+@CrossOrigin("*")
+@RequestMapping("/api/v1/login/")
+public class LoginController {
+
+    @Resource
+    private ILoginService loginService;
+
+    @Value("${weixin.config.openid}")
+    private String openid;
+
+    /**
+     * http://xfg-studio.natapp1.cc/api/v1/login/weixin_qrcode_ticket
+     * @return
+     */
+    @RequestMapping(value = "weixin_qrcode_ticket", method = RequestMethod.GET)
+    public Response<String> weixinQrCodeTicket() {
+        try {
+            String qrCodeTicket = loginService.createQrCodeTicket();
+            log.info("生成微信扫码登录 ticket:{}", qrCodeTicket);
+            //todo 这里因为没有私人服务器，先不接入微信扫码登录功能，默认生成二维码后直接算登录
+            loginService.saveLoginState(qrCodeTicket,openid);
+            return Response.<String>builder()
+                    .code(Constants.ResponseCode.SUCCESS.getCode())
+                    .info(Constants.ResponseCode.SUCCESS.getInfo())
+                    .data(qrCodeTicket)
+                    .build();
+        } catch (Exception e) {
+            log.error("生成微信扫码登录 ticket 失败", e);
+            return Response.<String>builder()
+                    .code(Constants.ResponseCode.UN_ERROR.getCode())
+                    .info(Constants.ResponseCode.UN_ERROR.getInfo())
+                    .build();
+        }
+    }
+
+    /**
+     * http://xfg-studio.natapp1.cc/api/v1/login/check_login
+     */
+    @RequestMapping(value = "check_login", method = RequestMethod.GET)
+    public Response<String> checkLogin(@RequestParam String ticket) {
+        try {
+            String openidToken = loginService.checkLogin(ticket);
+            log.info("扫码检测登录结果 ticket:{} openidToken:{}", ticket, openidToken);
+            if (StringUtils.isNotBlank(openidToken)) {
+                return Response.<String>builder()
+                        .code(Constants.ResponseCode.SUCCESS.getCode())
+                        .info(Constants.ResponseCode.SUCCESS.getInfo())
+                        .data(openidToken)
+                        .build();
+            } else {
+                return Response.<String>builder()
+                        .code(Constants.ResponseCode.NO_LOGIN.getCode())
+                        .info(Constants.ResponseCode.NO_LOGIN.getInfo())
+                        .build();
+            }
+        } catch (Exception e) {
+            log.error("扫码检测登录结果失败 ticket:{}", ticket, e);
+            return Response.<String>builder()
+                    .code(Constants.ResponseCode.UN_ERROR.getCode())
+                    .info(Constants.ResponseCode.UN_ERROR.getInfo())
+                    .build();
+        }
+    }
+
+}
diff --git a/small-pay-controller/src/main/resources/application-dev.yml b/small-pay-controller/src/main/resources/application-dev.yml
index b12ab97..01d873e 100644
--- a/small-pay-controller/src/main/resources/application-dev.yml
+++ b/small-pay-controller/src/main/resources/application-dev.yml
@@ -7,6 +7,20 @@ server:
       max: 20
       min-spare: 10
     accept-count: 10
+spring:
+  redis:
+    host: 127.0.0.1
+    port: 6379
+    database: 1
+    password: 1190670265
+
+weixin:
+  config:
+    app-id: wx2d160f15c419f7b3
+    app-secret: e5a76e38af434f706a20389fa6eca496
+    template_id: Lv7fSJ7sYuvyEMx5lF_o_VsfOKyYDVm0TJe-s85JVho
+    openid: o7Oox3ArEQmsNnVUiWi8irX9Uqww
+
 
 #spring:
 #  datasource:
diff --git a/small-pay-controller/src/main/resources/application.yml b/small-pay-controller/src/main/resources/application.yml
index b6a1c79..802f390 100644
--- a/small-pay-controller/src/main/resources/application.yml
+++ b/small-pay-controller/src/main/resources/application.yml
@@ -3,3 +3,5 @@ spring:
     name: s-pay-mall-mvc
   profiles:
     active: dev
+
+
diff --git a/small-pay-domain/src/main/java/com/superd/domain/po/WeixinTemplateMessageVO.java b/small-pay-domain/src/main/java/com/superd/domain/po/WeixinTemplateMessageVO.java
new file mode 100644
index 0000000..69e60b5
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/po/WeixinTemplateMessageVO.java
@@ -0,0 +1,72 @@
+package com.superd.domain.po;
+
+import lombok.Getter;
+import lombok.Setter;
+
+import java.util.HashMap;
+import java.util.Map;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 微信模板消息
+ * @create 2024-09-28 13:38
+ */
+@Setter
+@Getter
+public class WeixinTemplateMessageVO {//todo 记得更换为自己的信息
+
+    private String touser = "or0Ab6ivwmypESVp_bYuk92T6SvU";
+    private String template_id = "GLlAM-Q4jdgsktdNd35hnEbHVam2mwsW2YWuxDhpQkU";
+    private String url = "https://weixin.qq.com";
+    private Map<String, Map<String, String>> data = new HashMap<>();
+
+    public WeixinTemplateMessageVO(String touser, String template_id) {
+        this.touser = touser;
+        this.template_id = template_id;
+    }
+
+    public void put(TemplateKey key, String value) {
+        data.put(key.getCode(), new HashMap<String, String>() {
+            private static final long serialVersionUID = 7092338402387318563L;
+
+            {
+                put("value", value);
+            }
+        });
+    }
+
+    public static void put(Map<String, Map<String, String>> data, TemplateKey key, String value) {
+        data.put(key.getCode(), new HashMap<String, String>() {
+            private static final long serialVersionUID = 7092338402387318563L;
+
+            {
+                put("value", value);
+            }
+        });
+    }
+
+
+    @Getter
+    public enum TemplateKey {
+        USER("user","用户ID")
+        ;
+
+        private String code;
+        private String desc;
+
+        TemplateKey(String code, String desc) {
+            this.code = code;
+            this.desc = desc;
+        }
+
+        public void setCode(String code) {
+            this.code = code;
+        }
+
+        public void setDesc(String desc) {
+            this.desc = desc;
+        }
+    }
+
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/req/WeixinAccessTokenReq.java b/small-pay-domain/src/main/java/com/superd/domain/req/WeixinAccessTokenReq.java
new file mode 100644
index 0000000..0c7d240
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/req/WeixinAccessTokenReq.java
@@ -0,0 +1,22 @@
+package com.superd.domain.req;
+
+import lombok.AllArgsConstructor;
+import lombok.Builder;
+import lombok.Data;
+import lombok.NoArgsConstructor;
+
+/**
+ * 微信获取 access_token 请求对象
+ */
+@Data
+@Builder
+@NoArgsConstructor
+@AllArgsConstructor
+public class WeixinAccessTokenReq {
+
+    private String grant_type;
+    private String appid;
+    private String secret;
+    private Boolean force_refresh;
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/req/WeixinQrCodeReq.java b/small-pay-domain/src/main/java/com/superd/domain/req/WeixinQrCodeReq.java
new file mode 100644
index 0000000..be8cdf6
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/req/WeixinQrCodeReq.java
@@ -0,0 +1,50 @@
+package com.superd.domain.req;
+
+import lombok.*;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 获取微信登录二维码请求对象
+ * @create 2024-09-28 13:36
+ */
+@Data
+@Builder
+@AllArgsConstructor
+@NoArgsConstructor
+public class WeixinQrCodeReq {
+
+    private int expire_seconds;
+    private String action_name;
+    private ActionInfo action_info;
+
+    @Data
+    @Builder
+    @AllArgsConstructor
+    @NoArgsConstructor
+    public static class ActionInfo {
+        Scene scene;
+
+        @Data
+        @Builder
+        @AllArgsConstructor
+        @NoArgsConstructor
+        public static class Scene {
+            int scene_id;
+            String scene_str;
+        }
+    }
+
+    @Getter
+    @AllArgsConstructor
+    @NoArgsConstructor
+    public enum ActionNameTypeVO {
+        QR_SCENE("QR_SCENE", "临时的整型参数值"),
+        QR_STR_SCENE("QR_STR_SCENE", "临时的字符串参数值"),
+        QR_LIMIT_SCENE("QR_LIMIT_SCENE", "永久的整型参数值"),
+        QR_LIMIT_STR_SCENE("QR_LIMIT_STR_SCENE", "永久的字符串参数值");
+
+        private String code;
+        private String info;
+    }
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/res/WeixinQrCodeRes.java b/small-pay-domain/src/main/java/com/superd/domain/res/WeixinQrCodeRes.java
new file mode 100644
index 0000000..849d769
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/res/WeixinQrCodeRes.java
@@ -0,0 +1,17 @@
+package com.superd.domain.res;
+
+import lombok.Data;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 获取微信登录二维码响应对象
+ * @create 2024-09-28 13:36
+ */
+@Data
+public class WeixinQrCodeRes {
+
+    private String ticket;
+    private Long expire_seconds;
+    private String url;
+
+}
diff --git a/small-pay-domain/src/main/java/com/superd/domain/res/WeixinTokenRes.java b/small-pay-domain/src/main/java/com/superd/domain/res/WeixinTokenRes.java
new file mode 100644
index 0000000..f298e67
--- /dev/null
+++ b/small-pay-domain/src/main/java/com/superd/domain/res/WeixinTokenRes.java
@@ -0,0 +1,18 @@
+package com.superd.domain.res;
+
+import lombok.Data;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 获取 Access token DTO 对象
+ * @create 2024-09-28 13:32
+ */
+@Data
+public class WeixinTokenRes {
+
+    private String access_token;
+    private int expires_in;
+    private String errcode;
+    private String errmsg;
+
+}
diff --git a/small-pay-service/pom.xml b/small-pay-service/pom.xml
index d04f0af..d0a83d9 100644
--- a/small-pay-service/pom.xml
+++ b/small-pay-service/pom.xml
@@ -22,6 +22,10 @@
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-web</artifactId>
         </dependency>
+        <dependency>
+            <groupId>org.springframework.boot</groupId>
+            <artifactId>spring-boot-starter-data-redis</artifactId>
+        </dependency>
         <dependency>
             <groupId>org.projectlombok</groupId>
             <artifactId>lombok</artifactId>
diff --git a/small-pay-service/src/main/java/com/superd/service/ILoginService.java b/small-pay-service/src/main/java/com/superd/service/ILoginService.java
new file mode 100644
index 0000000..49797bb
--- /dev/null
+++ b/small-pay-service/src/main/java/com/superd/service/ILoginService.java
@@ -0,0 +1,18 @@
+package com.superd.service;
+
+import java.io.IOException;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 微信服务
+ * @create 2024-09-28 13:44
+ */
+public interface ILoginService {
+
+    String createQrCodeTicket() throws Exception;
+
+    String checkLogin(String ticket);
+
+    void saveLoginState(String ticket, String openid) throws IOException;
+
+}
diff --git a/small-pay-service/src/main/java/com/superd/service/impl/WeixinLoginServiceImpl.java b/small-pay-service/src/main/java/com/superd/service/impl/WeixinLoginServiceImpl.java
new file mode 100644
index 0000000..4b2b96a
--- /dev/null
+++ b/small-pay-service/src/main/java/com/superd/service/impl/WeixinLoginServiceImpl.java
@@ -0,0 +1,145 @@
+package com.superd.service.impl;
+
+import com.superd.domain.po.WeixinTemplateMessageVO;
+import com.superd.domain.req.WeixinAccessTokenReq;
+import com.superd.domain.req.WeixinQrCodeReq;
+import com.superd.domain.res.WeixinQrCodeRes;
+import com.superd.domain.res.WeixinTokenRes;
+import com.superd.service.ILoginService;
+import com.superd.service.weixin.IWeixinApiService;
+import org.apache.commons.lang3.StringUtils;
+import org.springframework.beans.factory.annotation.Value;
+import org.springframework.data.redis.core.StringRedisTemplate;
+import org.springframework.stereotype.Service;
+import retrofit2.Call;
+
+import javax.annotation.Resource;
+import java.io.IOException;
+import java.util.HashMap;
+import java.util.Map;
+import java.util.concurrent.TimeUnit;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * 微信服务
+ * 2024-09-28 13:46
+ */
+@Service
+public class WeixinLoginServiceImpl implements ILoginService {
+
+    private static final String ACCESS_TOKEN_KEY_PREFIX = "smallpay:wx:access_token:";
+    private static final String OPENID_TOKEN_KEY_PREFIX = "smallpay:wx:ticket_openid:";
+    private static final long LOGIN_STATE_TTL_SECONDS = 2592000L;
+    private static final long ACCESS_TOKEN_TTL_FALLBACK_SECONDS = 7000L;
+    private static final long ACCESS_TOKEN_TTL_BUFFER_SECONDS = 200L;
+
+    @Value("${weixin.config.app-id}")
+    private String appid;
+    @Value("${weixin.config.app-secret}")
+    private String appSecret;
+    @Value("${weixin.config.template_id}")
+    private String template_id;
+
+
+    @Resource
+    private IWeixinApiService weixinApiService;
+    @Resource
+    private StringRedisTemplate stringRedisTemplate;
+
+    @Override
+    public String createQrCodeTicket() throws Exception {
+        // 1. 获取 accessToken
+        String accessToken = stringRedisTemplate.opsForValue().get(accessTokenKey());
+        if (null == accessToken) {
+            accessToken = fetchAndCacheAccessToken();
+        }
+
+        // 2. 生成 ticket
+        WeixinQrCodeReq weixinQrCodeReq = WeixinQrCodeReq.builder()
+                .expire_seconds(2592000)
+                .action_name(WeixinQrCodeReq.ActionNameTypeVO.QR_SCENE.getCode())
+                .action_info(WeixinQrCodeReq.ActionInfo.builder()
+                        .scene(WeixinQrCodeReq.ActionInfo.Scene.builder()
+                                .scene_id(123)
+                                .build())
+                        .build())
+                .build();
+
+        Call<WeixinQrCodeRes> call = weixinApiService.createQrCode(accessToken, weixinQrCodeReq);
+        WeixinQrCodeRes weixinQrCodeRes = call.execute().body();
+        assert null != weixinQrCodeRes;
+        return weixinQrCodeRes.getTicket();
+    }
+
+    @Override
+    public String checkLogin(String ticket) {
+        return stringRedisTemplate.opsForValue().get(openidTokenKey(ticket));
+    }
+
+    @Override
+    public void saveLoginState(String ticket, String openid) throws IOException {
+        stringRedisTemplate.opsForValue().set(openidTokenKey(ticket), openid, LOGIN_STATE_TTL_SECONDS, TimeUnit.SECONDS);
+
+        // 1. 获取 accessToken 【实际业务场景，按需处理下异常】
+        String accessToken = stringRedisTemplate.opsForValue().get(accessTokenKey());
+        if (null == accessToken){
+            accessToken = fetchAndCacheAccessToken();
+        }
+
+        // 2. 发送模板消息
+        Map<String, Map<String, String>> data = new HashMap<>();
+        WeixinTemplateMessageVO.put(data, WeixinTemplateMessageVO.TemplateKey.USER, openid);
+
+        WeixinTemplateMessageVO templateMessageDTO = new WeixinTemplateMessageVO(openid, template_id);
+        templateMessageDTO.setUrl("https://gaga.plus");//todo 后续需要动态设置
+        templateMessageDTO.setData(data);
+
+        Call<Void> call = weixinApiService.sendMessage(accessToken, templateMessageDTO);
+        call.execute();
+
+    }
+
+    private String fetchAndCacheAccessToken() throws IOException {
+        if (StringUtils.isBlank(appid) || StringUtils.isBlank(appSecret)) {
+            throw new IllegalStateException("微信配置未正确加载：appid 或 appSecret 为空");
+        }
+
+        WeixinAccessTokenReq weixinAccessTokenReq = WeixinAccessTokenReq.builder()
+                .grant_type("client_credential")
+                .appid(appid)
+                .secret(appSecret)
+                .force_refresh(false)
+                .build();
+
+        Call<WeixinTokenRes> call = weixinApiService.getToken(weixinAccessTokenReq);
+        WeixinTokenRes weixinTokenRes = call.execute().body();
+        if (null == weixinTokenRes) {
+            throw new IllegalStateException("获取微信 access_token 失败：响应体为空");
+        }
+
+        if (StringUtils.isNotBlank(weixinTokenRes.getErrcode())) {
+            throw new IllegalStateException("获取微信 access_token 失败：errcode=" + weixinTokenRes.getErrcode() + ", errmsg=" + weixinTokenRes.getErrmsg());
+        }
+
+        String accessToken = weixinTokenRes.getAccess_token();
+        if (StringUtils.isBlank(accessToken)) {
+            throw new IllegalStateException("获取微信 access_token 失败：access_token 为空");
+        }
+
+        long ttlSeconds = weixinTokenRes.getExpires_in() - ACCESS_TOKEN_TTL_BUFFER_SECONDS;
+        if (ttlSeconds <= 0) {
+            ttlSeconds = ACCESS_TOKEN_TTL_FALLBACK_SECONDS;
+        }
+        stringRedisTemplate.opsForValue().set(accessTokenKey(), accessToken, ttlSeconds, TimeUnit.SECONDS);
+        return accessToken;
+    }
+
+    private String accessTokenKey() {
+        return ACCESS_TOKEN_KEY_PREFIX + appid;
+    }
+
+    private String openidTokenKey(String ticket) {
+        return OPENID_TOKEN_KEY_PREFIX + ticket;
+    }
+
+}
diff --git a/small-pay-service/src/main/java/com/superd/service/weixin/IWeixinApiService.java b/small-pay-service/src/main/java/com/superd/service/weixin/IWeixinApiService.java
new file mode 100644
index 0000000..cccec21
--- /dev/null
+++ b/small-pay-service/src/main/java/com/superd/service/weixin/IWeixinApiService.java
@@ -0,0 +1,53 @@
+package com.superd.service.weixin;
+
+import com.superd.domain.po.WeixinTemplateMessageVO;
+import com.superd.domain.req.WeixinAccessTokenReq;
+import com.superd.domain.req.WeixinQrCodeReq;
+import com.superd.domain.res.WeixinQrCodeRes;
+import com.superd.domain.res.WeixinTokenRes;
+import retrofit2.Call;
+import retrofit2.http.Body;
+import retrofit2.http.POST;
+import retrofit2.http.Query;
+
+/**
+ * @author Fuzhengwei bugstack.cn @小傅哥
+ * @description 微信API服务 retrofit2
+ * @create 2024-09-28 13:30
+ */
+public interface IWeixinApiService {
+
+    /**
+     * 获取 Access token
+     * 文档：<a href="https://developers.weixin.qq.com/doc/offiaccount/Basic_Information/Get_access_token.html">Get_access_token</a>
+     *
+     * @param weixinAccessTokenReq 获取access_token请求体
+     * @return 响应结果
+     */
+    @POST("cgi-bin/stable_token")
+    Call<WeixinTokenRes> getToken(@Body WeixinAccessTokenReq weixinAccessTokenReq);
+
+    /**
+     * 获取凭据 ticket
+     * 文档：<a href="https://developers.weixin.qq.com/doc/offiaccount/Account_Management/Generating_a_Parametric_QR_Code.html">Generating_a_Parametric_QR_Code</a>
+     * <a href="https://mp.weixin.qq.com/cgi-bin/showqrcode?ticket=TICKET">前端根据凭证展示二维码</a>
+     *
+     * @param accessToken            getToken 获取的 token 信息
+     * @param weixinQrCodeReq 入参对象
+     * @return 应答结果
+     */
+    @POST("cgi-bin/qrcode/create")
+    Call<WeixinQrCodeRes> createQrCode(@Query("access_token") String accessToken, @Body WeixinQrCodeReq weixinQrCodeReq);
+
+    /**
+     * 发送微信公众号模板消息
+     * 文档：https://mp.weixin.qq.com/debug/cgi-bin/readtmpl?t=tmplmsg/faq_tmpl
+     *
+     * @param accessToken              getToken 获取的 token 信息
+     * @param weixinTemplateMessageVO 入参对象
+     * @return 应答结果
+     */
+    @POST("cgi-bin/message/template/send")
+    Call<Void> sendMessage(@Query("access_token") String accessToken, @Body WeixinTemplateMessageVO weixinTemplateMessageVO);
+
+}
```
