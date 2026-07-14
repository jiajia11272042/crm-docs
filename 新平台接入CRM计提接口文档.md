# 新平台接入CRM计提接口文档

## 一、接口概述

| 项目 | 说明 |
|------|------|
| **接口名称** | 计提数据推送接口 |
| **请求方式** | POST |
| **接口地址** | `http://{CRM域名}/crm/api/o/provision/external/accept` |
| **Content-Type** | application/json |
| **认证方式** | 签名验证（MD5） |
| **调用方向** | 新平台 → CRM |

---

## 二、请求参数

### 2.1 外层参数结构（ExternalParamBo）

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| sign | String | ✅ | 签名字符串（每次请求不同） |
| timestamp | Date | ✅ | 请求时间戳 |
| count | Integer | ✅ | data数组的长度 |
| data | List\<CrmData\> | ✅ | 计提数据列表 |

### 2.2 计提数据字段（CrmData）

| 参数名 | 类型 | 必填 | 说明 | 示例值 |
|--------|------|------|------|--------|
| contractCode | String | ✅ | 合同号（CRM中已存在的合同） | `HT202607001` |
| country | String | ✅ | 国家 | `US` |
| media | String | ✅ | 媒体 | `Google` |
| provisionMoney | BigDecimal | ✅ | 计提收入（结算币种金额） | `10000.00` |
| origUsdMoney | BigDecimal | ✅ | 原始预计收入（美金） | `1200.00` |
| usdMoney | BigDecimal | ✅ | 预计收入（美金） | `1200.00` |
| earningType | String | ✅ | 收入类型 | `收入` |
| contractType | String | ✅ | 合同类型（0-收款，1-付款） | `0` |
| settlementMonth | String | ✅ | 结算月份（格式：yyyy-MM-dd HH:mm:ss） | `2026-07-01 00:00:00` |
| remark | String | ⚪ | 备注 | `7月计提` |
| accrualBy | String | ⚪ | 计提人 | `张三` |

### 2.3 重要说明

| 说明项 | 内容 |
|--------|------|
| **不需要传的字段** | id、errorReason（CRM会自动生成provisionId） |
| **contractCode必须存在** | CRM中必须有这个合同，否则会报错"合同不存在" |
| **country+media唯一** | 同一合同+月份下，country+media必须唯一 |
| **金额单位** | provisionMoney是结算币种金额，usdMoney是美金金额 |
| **日期格式** | settlementMonth格式必须是 `yyyy-MM-dd HH:mm:ss` |

---

## 三、签名算法

### 3.1 签名生成规则

```java
// 1. 准备参数
Map<String, Object> param = new HashMap<>();
param.put("data", data);           // 计提数据（JSON数组）
param.put("secretKey", APP_SECRET); // 固定密钥
param.put("timestamp", timestamp);  // 时间戳
param.put("count", count);          // 数据条数

// 2. 排序并拼接
// 按key排序，拼接成: key1=value1&key2=value2&...&secretKey=xxx

// 3. MD5加密
String sign = MD5(拼接后的字符串).toUpperCase();
```

### 3.2 固定密钥

```java
public static final String APP_SECRET = "244e7f76db84b915ff10dd9d239eeb4b";
```

### 3.3 签名特点

- **每次请求都不同**：因为timestamp和data内容都在变化
- **安全性**：保证请求未被篡改
- **幂等性**：相同参数+时间戳产生相同签名

---

## 四、请求示例

### 4.1 完整请求示例

```json
{
  "sign": "A1B2C3D4E5F6G7H8I9J0K1L2M3N4O5P6",
  "timestamp": "2026-07-07T10:30:00",
  "count": 2,
  "data": [
    {
      "contractCode": "HT202607001",
      "country": "US",
      "media": "Google",
      "provisionMoney": 10000.00,
      "origUsdMoney": 1200.00,
      "usdMoney": 1200.00,
      "earningType": "收入",
      "contractType": "0",
      "settlementMonth": "2026-07-01 00:00:00",
      "remark": "7月程序化收入计提",
      "accrualBy": "张三"
    },
    {
      "contractCode": "HT202607001",
      "country": "JP",
      "media": "Facebook",
      "provisionMoney": 8000.00,
      "origUsdMoney": 960.00,
      "usdMoney": 960.00,
      "earningType": "收入",
      "contractType": "0",
      "settlementMonth": "2026-07-01 00:00:00",
      "remark": "7月程序化收入计提",
      "accrualBy": "张三"
    }
  ]
}
```

---

## 五、响应参数

### 5.1 响应结构

| 参数名 | 类型 | 说明 |
|--------|------|------|
| code | String | 状态码 |
| messageInfo | String | 提示信息 |
| data | List\<Pair\<String, String\>\> | 失败的记录（key=id, value=错误原因） |

### 5.2 状态码说明

| 状态码 | 含义 | 说明 |
|--------|------|------|
| 200 | 成功 | 所有数据处理成功，data为空 |
| 4002 | 部分成功 | 部分数据失败，data中包含失败记录 |
| 400 | 失败 | 所有数据处理失败 |

### 5.3 成功响应示例

```json
{
  "code": "200",
  "messageInfo": "success",
  "data": []
}
```

### 5.4 部分成功响应示例

```json
{
  "code": "4002",
  "messageInfo": "部分数据处理失败",
  "data": [
    {
      "key": "HT202607001-2026-07-01 00:00:00-US-Google",
      "value": "合同不存在"
    }
  ]
}
```

### 5.5 失败响应示例

```json
{
  "code": "400",
  "messageInfo": "签名验证失败",
  "data": null
}
```

---

## 六、错误码说明

| 错误码 | 错误原因 | 解决方案 |
|--------|----------|----------|
| 400 | 签名验证失败 | 检查签名算法和密钥是否正确 |
| 400 | 数据为空 | 检查data数组是否为空 |
| 400 | 参数数量有误 | 检查count是否与data数组长度一致 |
| 400 | 合同不存在 | 检查contractCode是否在CRM中存在 |
| 4002 | 部分失败 | 查看data中的具体错误原因 |

---

## 七、常见错误原因

| 错误原因 | 说明 | 解决方案 |
|----------|------|----------|
| 合同不存在 | contractCode在CRM中不存在 | 确认合同已在CRM中创建 |
| 签名验证失败 | sign计算错误 | 检查签名算法实现 |
| 数据重复 | 同一合同+月份+国家+媒体已存在 | 检查是否重复推送 |
| 金额格式错误 | 金额不是数字格式 | 检查provisionMoney等字段类型 |
| 日期格式错误 | settlementMonth格式不正确 | 使用yyyy-MM-dd HH:mm:ss格式 |

---

## 八、接入流程

### 8.1 接入步骤

```
1. 确认合同已在CRM中创建
   ↓
2. 按照CrmData格式准备计提数据
   ↓
3. 实现签名算法（MD5）
   ↓
4. 构造请求参数（ExternalParamBo）
   ↓
5. 发送HTTP POST请求
   ↓
6. 处理响应结果
   ↓
7. 失败数据重新推送
```

### 8.2 注意事项

| 事项 | 说明 |
|------|------|
| **合同必须存在** | 推送前确保contractCode在CRM中已创建 |
| **数据唯一性** | 同一合同+月份+国家+媒体不能重复推送 |
| **失败重试** | 失败的数据可以重新推送，CRM会自动更新 |
| **签名时效** | 建议timestamp使用当前时间，避免过期 |
| **批量推送** | 可以一次推送多条数据，建议每批不超过100条 |

---

## 九、Java代码示例

### 9.1 发送请求示例

```java
import cn.hutool.http.HttpRequest;
import cn.hutool.http.HttpResponse;
import com.alibaba.fastjson.JSON;
import com.yourplatform.util.SignUtil;

public class CrmProvisionClient {
    
    private static final String CRM_URL = "http://crm.example.com/crm/api/o/provision/external/accept";
    private static final String APP_SECRET = "244e7f76db84b915ff10dd9d239eeb4b";
    
    /**
     * 推送计提数据到CRM
     */
    public String pushProvisionData(List<CrmData> dataList) {
        // 1. 构造请求参数
        Map<String, Object> params = new HashMap<>();
        Date timestamp = new Date();
        
        // 2. 生成签名
        String sign = SignUtil.generateSign(dataList, timestamp, dataList.size(), APP_SECRET);
        
        // 3. 组装参数
        params.put("sign", sign);
        params.put("timestamp", timestamp);
        params.put("count", dataList.size());
        params.put("data", dataList);
        
        // 4. 发送请求
        String json = JSON.toJSONString(params);
        HttpResponse response = HttpRequest.post(CRM_URL)
                .header("Content-Type", "application/json")
                .body(json)
                .execute();
        
        return response.body();
    }
}
```

### 9.2 CrmData实体类

```java
import lombok.Data;
import java.math.BigDecimal;

@Data
public class CrmData {
    private String contractCode;      // 合同号
    private String country;           // 国家
    private String media;             // 媒体
    private BigDecimal provisionMoney; // 计提收入
    private BigDecimal origUsdMoney;  // 原始预计收入
    private BigDecimal usdMoney;      // 预计收入
    private String earningType;       // 收入类型
    private String contractType;      // 合同类型
    private String settlementMonth;   // 结算月份
    private String remark;            // 备注
    private String accrualBy;         // 计提人
}
```

### 9.3 签名工具类

```java
import cn.hutool.crypto.digest.DigestUtil;
import java.util.*;

public class SignUtil {
    
    private static final String SECRET_KEY = "secretKey";
    private static final String TIMESTAMP_KEY = "timestamp";
    private static final String COUNT_KEY = "count";
    
    /**
     * 生成签名
     */
    public static String generateSign(Object data, Date timestamp, 
                                      Integer count, String appSecret) {
        Map<String, Object> param = new HashMap<>();
        param.put("param", data);
        param.put(SECRET_KEY, appSecret);
        param.put(TIMESTAMP_KEY, timestamp);
        param.put(COUNT_KEY, count);
        
        return createSign(param, appSecret);
    }
    
    /**
     * 创建签名
     */
    private static String createSign(Map<String, Object> params, String secret) {
        // 1. 排序
        Set<String> keysSet = params.keySet();
        Object[] keys = keysSet.toArray();
        Arrays.sort(keys);
        
        // 2. 拼接
        StringBuilder temp = new StringBuilder();
        for (Object key : keys) {
            temp.append("&");
            temp.append(key).append("=");
            Object value = params.get(key);
            String valueString = value != null ? String.valueOf(value) : "";
            temp.append(valueString);
        }
        
        // 3. 删除第一个&
        if (temp.length() > 0) {
            temp.deleteCharAt(0);
        }
        
        // 4. 添加secretKey
        temp.append("&").append(SECRET_KEY).append("=").append(secret);
        
        // 5. MD5加密
        return DigestUtil.md5Hex(temp.toString()).toUpperCase();
    }
}
```

---

## 十、CRM生成的字段

### 10.1 CRM自动生成的字段

| 字段 | 生成规则 | 说明 |
|------|----------|------|
| id | 雪花算法 | 数据库主键，19位数字（用户不可见） |
| provisionId | 合同号+月份 | 格式：{合同类型前缀}{contractCode}{settlementMonth} |

### 10.2 provisionId格式示例

```
收款合同：S0042272026-06
付款合同：P0042282026-07

说明：
- S = 收款合同（Sale）
- P = 付款合同（Purchase）
- 004227 = contractCode
- 2026-06 = settlementMonth
```

---

## 十一、总结

### 新平台需要做的事

1. ✅ **准备数据**：按照CrmData格式准备计提数据
2. ✅ **实现签名**：按照签名算法生成sign
3. ✅ **发送请求**：POST到CRM接口
4. ✅ **处理响应**：根据状态码处理成功/失败
5. ✅ **失败重试**：失败的数据可以重新推送

### 不需要做的事

- ❌ 不需要生成provisionId（CRM自动生成）
- ❌ 不需要生成id（CRM自动生成）
- ❌ 不需要关心计提平台的数据库
- ❌ 不需要关心计提平台的内部逻辑

### 关键字段

| 必传字段 | 说明 |
|----------|------|
| contractCode | 合同号（CRM中必须存在） |
| settlementMonth | 结算月份 |
| country | 国家 |
| media | 媒体 |
| provisionMoney | 计提金额 |
| usdMoney | 美金金额 |
| earningType | 收入类型 |
| contractType | 合同类型 |

---

## 十二、联系方式

如有问题，请联系CRM系统管理员。

---

**文档版本**：v1.0  
**更新时间**：2026年7月  
**维护人**：CRM开发团队
