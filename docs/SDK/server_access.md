# 支付服务端接入

## 支付流程说明

CP对接233平台交易时序图【重客户端】

![](https://cdn.233xyx.com/online/5CAFX6BeiI4x1716343001813.jpg)

**在233平台游戏道具充值购买场景**

1. 涉及对接方：游戏开发商（CP）、233游戏平台、第三方支付平台（微信支付、支付宝等）；
2. 涉及业务对接节点： 游戏开发商游戏程序、233平台交易SDK、游戏开发商服务器端、233服务器端、第三方支付平台；
3. CP在233平台下单的订单号必须要保证在CP交易系统中唯一性（与其他CP无关），如果重复，会对后续订单查询等接口造成不可逆影响，平台与CP也无法根据订单明细进行对账；
4. 233平台在新版本发货（V2）版本接口中新增了平台唯一性订单交易编号（tradeNo），推荐CP对接发货功能对接到V2的流程中；平台的tradeNo是具备唯一性；
5. CP在确认收到发货接口回调时（流程图第8步）建议回查233平台订单交易状态（接入参考文档）
6. CP需要自己保证发货接口的幂等性，233平台不保证同一笔订单发货接口只调用1次，且可能存在并发回调；
7. CP推荐在确认发货成功后再给出约定的已发货响应值，如果处理是失败，233平台还会重试多次发货回调；
8. CP接收回调后，推荐根据tradeNo查询233平台交易结果，并判断  订单原值金额 = CP本地订单金额；
- 如金额不一致，可认为是受到攻击；
- 如CP仍然发货，233平台与CP结算将以实际订单收款数据做结算，产生损失将由CP自行承担；
- 优惠券抵扣金额推荐不做是否发货逻辑判断，可用作财务结算对账使用；

## 发货通知接口  

**接口说明：**

 发货通知接口即支付结果通知接口，当玩家游戏内充值成功后，平台游戏服务端向开发者平台CP配置的回调地址发送订单通知，CP需要根据按照以下规则进行验签，以及返回平台要求的返回值。

::: danger 注意
因为网络或其他原因，可能会出现重复通知的情况，CP需要做好去重处理，避免重复发货
:::

### 回调验签规则 

平台目前采用的是**SHA1**方式对参数加密验签， 以下使用的**secret**为开放平台分配的**AppSecret**
签名规则如下：
> 参数大小写敏感（区分大小写）；
> 多参数拼接采用 **&** 符号作为分隔符；
> 参数的value为**null**或者是空字符串【**""**】是不参与签名，签名的参数（**sign**）也不参与签名；
> 参数签名顺序是按照参数名的**ASCII**码从小到大排序；
> 所有业务参数顺序拼接完成后，再在最后拼接cp秘钥参数，参数名为:**secret**；
> 签名值：**<font color="red">将签名参数拼接完成后的字符串进行SHA1加密，截取后32位字符，并转换为大写；</font>**
> 在原有参数列表中增加签名参数 sign ，与普通参数平级；

**签名示例：**

参数原始数据有：
```java
{
    "orderId" : "202001101301002" , 
    "productName" : "pizza" , 
    "year" : 2020 ,  
    "desc" : "" , 
    "sort" :  107
}
```

**CP在233平台秘钥为:** `4D2CD76B80C40B3B4EAE2E04BACA46B8`

**签名字符串为:**  
`orderId=202001101301002&productName=pizza&sort=107&year=2020&secret=4D2CD76B80C40B3B4EAE2E04BACA46B8`

**签名值(SIGN)：**`9AD9B18B1E0E59287AB8E5E3E414D072`
> 该值为SHA1加密后截取后32位并转大写

在参数体内再额外增加签名的值，提交的参数即为

```json
{
    "orderId" : "202001101301002" , 
    "productName" : "pizza" , 
    "year" : 2020 ,  
    "desc" : "" , 
    "sort" :  107, 
    "sign" : "9AD9B18B1E0E59287AB8E5E3E414D072" 
}
```

### V2版本接入

**接口说明：** 充值回调接口，默认都走V2版本，V1接口逐步废弃

**接收方：** 由CP在233开发者平台配置发货地址 和 选择接入版本【V2】

**请求方式：**  POST

**Content-Type:**  APPLICATION/JSON

**接口安全类型：** SHA1接口参数验签

**请求参数**
```json
{
"tradeNo"           :   String,
"cpOrderId"         :   String,
"productCode"       :   String,
"productName"       :   String,
"productPrice"      :   unsigned int,
"count"             :   unsigned int,
"nonce"             :   String,
"amount"            :   unsigned int,
"couponDeductAmount":   unsigned int,
"extra"             :   String,
"sign"              :   String
}
```

**参数说明：**
| 字段 | 是否为空 | 描述|
| ---- | :----: | ---- |
| tradeNo | N | 233平台对应的交易订单编号（具备唯一性）|
| cpOrderId | N | 供应商交易订单id（由供应商下单后出传入）|
| productCode | N | 商品编号（供应商下单时提供）|
| productName | N | 商品名称（供应商下单时提供）|
| productPrice | N | 商品单价（单位：分【人民币分】）|
| count | N | 订单购买购买数量|
| nonce | N | 随机安全码（无业务意义，增加接口安全随机性）|
| amount| N | 订单原值金额（单位：分【人民币】）,建议服务端做一下金额校验，避免被刷|
| couponDeductAmount | Y |   优惠券抵扣金额（因优惠券而优惠的金额）（单位：分【人民币】），当该笔交易未使用优惠券时，值为0|
| extra | Y | 由供应商预下单时传入扩展透传参数，原封不动给到供应商系统，当CP交易系统下单未提供扩展字段时，该字段为空|
| sign | N | 签名字段，<a href="#回调验签规则">签名规则</a> |

**游戏方响应结果**

游戏方需要按照以下格式返回结果给平台，平台会根据对应的code做不同的处理。

```json
{
"code"      :   int, //响应状态码
"message"   :   String  //状态码描述信息
}
```

**`code`响应码说明： **

 | 状态码 | 描述 | 是否重试请求 |
| ---- | ---- | ---- |
| 200 | 业务处理成功,发货成功 | 否 |
| 22100 | 数据签名不正确 | 是 |
| 22101 | 接口参数验证不通过 | 是 |
| 22103 | 未知业务错误（CP运行时未知错误异常）| 是 |	
| 22102 | CP无法发货（例如因为无库存、商品下架等原因，明确指出该笔订单无法履约【233平台会立刻给用户退款，并取消该笔订单】）;	| 否(平台退款给玩家) |

> PS: 当游戏服务端返回非200给平台，平台会按照`斐波那契的数列间隔`向游戏服务重试请求

### V1版本接入（废弃）

**接口说明：** 充值回调接口，V1接口已废弃

**接收方：** 由CP在233开发者平台配置发货地址 和 选择接入版本【V2】

**请求方式：**  POST

**Content-Type:**  APPLICATION/JSON

**接口安全类型：** SHA1接口参数验签
```json
{
"resultCode"        :   String,
"resultDesc"        :   String,
"orderId"           :   String,
"orderAmount"       :   unsigned int,
"orderProductCode"  :   String,
"voucherAmount"     :   unsigned int,
"voucherType"       :   int,
"voucherId"         :   String,
"cpExtra"           :   String,
}
 
 数据项解释：
- resultCode        @NotNull        订单交易结果编码； "SUCCESS" ，常量， SUCCESS 标识订单付款完成；
- resultDesc        @NotNull        订单结果描述，"OK"；常量；
- orderId           @NotNull        CP交易系统订单编号（由CP方订单系统产生的唯一性订单编号），CP应保证在其系统中的唯一性，订单编号不能重复，如果重复，会对后续订单查询等接口造成不可逆影响，平台与CP也无法根据订单明细进行对账；
- orderAmount       @NotNull        订单实际支付金额，（单位：分【人民币】）
- orderProductCode  @NotNull        商品编号（供应商下单时提供）
- voucherAmount     @NotNull        优惠券抵扣金额（因优惠券而优惠的金额），（单位：分【人民币】），当优惠券金额 > 0时，标识该笔交易使用了优惠券优惠了的金额，为0时标识并未使用优惠券；
- voucherType       @NotNull        代金券类型（接入方可忽略该字段类型解释），当未使用优惠券时，会有默认值0；
- voucherId         @Nullable       当该笔交易使用优惠券时，才会有值，
- cpExtra           @Nullable       由供应商预下单时传入扩展透传参数，原封不动给到供应商系统，当CP交易系统下单未提供扩展字段时，该字段为空
```

**游戏方响应值**

HTTP 状态码必须为200，Content-Type: APPLICATION/JSON

CP 需要给出准确的如下响应内容233平台才认为CP发货成功，否则还会继续重试发货回调；
`当CP返回明确响应内容体后，233平台会将订单流程处理完成，不会再进行发货接口回调，请接入方确认货物已经发送成功了，再给出准确的响应内容，发货流程响应CP自己处理幂等性（针对该笔订单）`

```json
{
"code"      :   int, //响应状态码，200:对接成功，其它标识失败；
"cpRewarded":   int //发货业务码; 1 : CP成功发货； 0:CP无法发货（例如因为无库存、商品下架等原因，明确指出该笔订单无法履约【233平台会立刻给用户退款，并取消该笔订单】）
} 
```
 


## 订单查询接口
 
CP根据订单号从233平台服务器查询指定订单的情况

**前置说明**

CP对接233开放平台接口有一套标准接口协议【只对接B端CP的服务器端功能接口】，
涉及CP的游戏身份认证和接口功能认证，统一接口鉴权请阅读 **[开放平台鉴权接入指南](./open_platform_authentication.md)** 

### V2版本

**接口地址：** `https://openapi.metaapp.cn/v2/trade/order/query` 

**请求方式:** `POST` 

**入参约束:** `Content-Type: APPLICATION/JSON`

**Header信息:** **[查看header参数规则](./open_platform_authentication.md)**

**请求参数**  
```json
{
"tradeNo": "233平台订单号,建议使用该字段",
"cpOrderId" :"游戏方自定义订单号"
}
```
参数说明：
| 字段 | 是否为空 | 描述 |
| ---- | :----: | ---- | 
| tradeNo | Y | 233平台对应的交易订单编号（具备唯一性）|
| cpOrderId | Y | 供应商交易订单id（由供应商下单后出传入）【不推荐使用，只是对老系统兼容，当CP无法保证其cpOrderId在CP交易系统中的唯一性时，会出现查询的交易数据非预期的目标数据】| 

::: danger 注意
两个参数不能同时为空或空字符串！必须有一个有值，如果同时传入，优先使用tradeNo【不会再使用cpOrderId，就算对应的tradeNo的订单数据不存在，也不会使用cpOrderId尝试获取数据】
:::

**响应数据：**  
```json
{
"code" : int, // 接口状态码，200：正常，其它均为异常； 1016:标识订单数据不存在；
"message" : String, //  接口响应状态描述或异常原因；
"data" : {
           "cpOrderId"         :       String, 
            "tradeNo"           :       String,
            "productCode"       :       String,
            "productName"       :       String,
            "productPrice"      :       unsigned int,
            "count"             :       unsigned int,
            "payed              :       bool,
            "amount"            :       unsigned int,
            "couponDeductAmount":       unsigned int,
            "extra"             :       String,
            "createTime"        :       long
          }
} 

```

**data字段说明：**

| 字段 | 是否为空 | 描述 |
| ---- | :----: | ---- | 
| cpOrderId | N | 供应商交易订单id（由供应商下单后出传入）    
| tradeNo   | N |   233平台对应的交易订单编号（具备唯一性）       |
| productCode | N | 商品编号（供应商下单时提供）       |
| productName | N | 商品名称（供应商下单时提供）       |
| productPrice | N | 商品单价（单位：分【人民币分】）       |
| count     | N |   订单购买购买数量       |
| payed     | N |   该笔交易是否已经付款        |
| amount    | N |   订单原值金额（单位：分【人民币】）       |
| couponDeductAmount | Y | 优惠券抵扣金额（单位：分【人民币】），当该笔交易未使用优惠券时，该字段为空       |
| extra | Y | 由供应商预下单时传入扩展透传参数，原封不动给到供应商系统，当CP交易系统下单未提供扩展字段时，该字段为空       |
| createTime  | N | 233平台订单创建时间（时间戳【毫秒】）时区：北京时区（东八区）  |

### V1版本【废弃】 

**接口地址:** `https://www.233xyx.com/apiserv/api/intermodal/queryCpOrder`

**请求方式:** `GET`

**请求参数:**

```json
"cpOrderId"  :  String
```

**参数说明：**

`cpOrderId` 供应商交易订单id（由供应商下单后出传入）
> 不推荐使用，只是对老系统兼容，当CP无法保证其cpOrderId在CP交易系统中的唯一性时，会出现查询的交易数据非预期的目标数据

::: danger 注意
该版本接口因为无约束CP的交易系统的订单号唯一性和部分数据安全协议，会导致接入方（CP）查询判断交易数据不准确性和较大数据安全风险；
故强烈推荐不使用该版本，应升级到V2版本保证数据安全和低风险性；
后期233平台提供的openApi交易接口也都会要求CP提供tradeNo作为交易订单号的要求【该接口并不能提供tradeNo给CP】；
:::

**响应数据：**

```json
{
    "return_code"           :           int, //状态码，200：正常，其它均为异常； 
    "return_msg"            :           String, //查询操作返回信息     
    "data"  :{
        "resultCode"            :           String,
        "resultDesc"            :           String,
        "orderId"               :           String,
        "orderAmount"           :           unsigned int,
        "orderProductCode"      :           String,
        "voucherAmount"         :           unsigned int,
        "voucherType"           :           int,
        "voucherId"             :           String,
        "cpExtra"               :           String
    }
} 

```

**data字段说明**
| 字段 | 是否为空 | 描述 |
| ---- | :----: | ---- |    
| resultCode | N | 订单付款状态编码【多个常量】； "SUCCESS" 、"PAYING"、"FAIL"等等， SUCCESS 标识订单付款完成，其它状态标识订单未付完款；     |
| resultDesc | N | 订单付款状态语义解释；      |
| orderId | Y | 供应商交易订单id（由供应商下单后出传入），当订单存在时，该字段@NotNull(不会为空)      |
| orderAmount | Y | 订单实际支付金额（单位：分【人民币】）;   当订单存在时，该字段@NotNull(不会为空)      |
| orderProductCode | Y |  商品编号（供应商下单时提供）              当订单存在时，该字段@NotNull(不会为空)      |
| voucherAmount    | Y |  优惠券抵扣金额（因优惠券而优惠的金额）（单位：分【人民币】）;     当订单存在时，该字段@NotNull(不会为空)，当优惠券金额 &gt; 0时，标识该笔交易使用了优惠券优惠了的金额，为0时标识并未使用优惠券；      |
| voucherType | Y | 代金券类型（接入方可忽略该字段类型解释），当未使用优惠券时，会有默认值0；      |
| voucherId   | Y | 当该笔交易使用优惠券时，才会有值，      |
| cpExtra | Y | 由供应商预下单时传入扩展透传参数，原封不动给到供应商系统，当CP交易系统下单未提供扩展字段时，该字段为空 |
