# 萤石开放平台 Node.js SDK

## 安装

```bash
npm install @ezviz/openplatform-sdk
```
**注意：本 SDK 依赖 Node.js 14 及以上版本。**

## Release Notes
1.0.0-beta.1

Release Date 2025-01-21

* 提供设备类、非设备类、RTC操作、取流操作、资源访问五类Token的生成。

## 基本问题FAQ
* Q: 生成Token的用处是什么？意味着什么？

  A: Token的中文含义是票据。它就象是一张高铁车票（非常类似）——在售票处，铁路公司在验明你的身份证，并且收到你的钱之后，颁发给你这样一个【凭证】，你可以持有这个【凭证】乘坐对应车次的高铁。
  在我们业务中，SaaS业务方可以在验明终端用户的身份和权限后，颁发一个票据（Token），作为该终端后续可以访问萤石云平台资源的凭证。因此一个Token（票据），意味着SaaS业务方将自己访问萤石云平台的权限，（临时性地）授予了某个终端（或三方）。

  上一句话中，“SaaS业务自己访问萤石云平台的权限”，意味着这次授予，无法超过SaaS业务方自身访问萤石云平台的权限范围。
  “临时性地”，表示授予的权限是存在有效期的，并且这个有效期不会很长。

* Q: Token的实际用法是怎么样的？

  A： 在我们的设想中，终端（PC、移动端、设备端）向SaaS服务（云眸/云耀/融合云、三方开发者）发起申请，告知SaaS服务自己要在萤石云上做一个什么操作。SaaS服务需要检查这次操作是否符合终端的身份权限，在检查后，颁发Token对终端进行授权。
  终端凭借Token(票据)来萤石云进行操作，萤石会检查——

  * Token（票据）是否伪造的。
  * Token（票据）是否过期了。
  * 这个Token来自哪个开发者（SaaS服务）
  * 这个Token授权的行为，是否与终端本次申请的操作的行为一致。（例如：终端不能拿着一个授权其操作设备A的票据，到萤石云平台来操作设备B）。
  * 颁发Token（票据）的开发者，是否具有执行此项操作的权限。

  上述检查都通过后，萤石云按终端本次的要求进行动作。

* Q：Token是如何防止伪造的？

  A: Token是SaaS服务颁发权限的凭证，就像一张动车票一样，有很多限制。终端在获得Token(票据)后，有可能试图修改和伪造Token来萤石云操作。如果这种行为能够得逞，那么整套机制也就没有存在的价值了。因此Token必须有防伪机制，在铁路系统中，这一行为由负责检票的检票员（现在是检票机）进行。

  * 一个开发者不能仿冒其他开发者生成Token
  * 终端要操作的行为和目标，如果发生了变化，能被检票方察觉。

  采用的方法是Token中会包含大量与本次操作相关信息—— 操作参数、操作目标、Token颁发者的AppKey这些都会被记录Token中。由于Token算法是公开的，因此防伪的任务被交付给现代密码学的研究成果。
  我们采用HMACSHA256算法，对操作参数进行摘要并计算出签名。
  为了防止开发者A冒充另一个开发者生成签名，签名使用开发者的SecretKey进行加密，因此每个开发者都必须保护好自己的SecretKey。**不要把SecretKey存储或传输到不安全的地方——比如移动端、网页代码中。** 由于生成Token需要传入SecretKey，**Token应当只在开发者的服务端程序生成。**

* Q： 如果给终端颁发了一个Token，然后有后悔了，能撤除吗？

  A: Token是基于加密运算来验证的，因此目前萤石云平台不提供撤回Token的功能。如果出现安全问题，您可以更换您的Appkey和SecretKey。要注意：更换后，开发者之前所有颁发的Token将全部失效。各个终端需要重新向SaaS服务申请新的Token才能访问萤石云。

## 开发者指南
### Q: 要怎么使用SDK来颁发Token

A: 请使用TokenGenerator的子类，这些子类都支持接口方法generateToken。例如生成非设备类操作的Token的方法是这样的：

```javascript
// 您需要用AppKey和 SecretKey 来初始化生成器对象。生成器对象可以缓存，并支持多线程并发。
// 请注意保管好您的 AppKey 和 SecretKey。

const { NonDeviceOpsTokenGenerator } = require('@ezviz/openplatform-sdk').Auth;

const generator = new NonDeviceOpsTokenGenerator();
generator.init(APP_KEY, SECRET_KEY);

const attributes = new Map();
attributes.set('role', 'admin');

const token = generator.generateToken({
  appId: 'app01',
  userId: 'user01',
  expire: 900,
  urlPattern: '/api/v3/conference/**',
  attributes,
});
```

不同的业务类型，可能需要传入不同的参数。具体参数请参考API章节。

### Q: 目前有那几类Tokengenerator，有什么用？

A:

| 类名 | 说明 | 用途 |
| - | - | - |
|NonDeviceOpsTokenGenerator | 非设备类操作授权Token | 可以访问开放平台网关，需要指定AppId，userId（终端标识）等参数后进行授权。 |
| DeviceGeneralTokenGenerator | 设备操作类的授权Token | 可以访问开放平台网关，需要指定DeviceSerial，ChannelNo等参数后进行授权。 |
| RTCTokenGenerator | RTC操作Token | RTCToken，用于RTC终端加入RTC会议使用。不能用于访问开放平台网关。 |
| StreamTokenGenerator | 取流操作Token | 取流Token，用于进行私有流取流。不能用于访问开放平台网关。 |
| GeneralResourceTokenGenerator。 | 资源访问Token | 用于访问资源服务器，可通过开放平台网关验证token。

### Q: 如何成为在颁发票据的时候贯彻安全原则

A: 为什么Token的颁发者要注意颁发Token的安全？因为任何终端，只要获得了票据后，就可以凭票据访问萤石开放平台的资源。这些资源被使用往往会有以下限制——

* 产生流量的操作会被计费
* 可能造成使用萤石开放平台的存储资源计费
* 可能造成网关接口访问次数被统计或因超出频率而被限流。

因为Token的颁发行为，本质上是开发者将自己的操作权限授权给Token的获得者，因此相关的费用和其他访问后果将由Token的颁发者负责。

为此，Token颁发者需要根据自己的业务特点，并且按照【权限最小原则】进行授权。**由于SecretKey保管不善而造成的信息泄漏，或者由于Token权限过大（有效期过长）造成的安全隐患，应由Token签发者自行负责。**

萤石开放平台设计中考虑了较多安全机制，来帮助Token颁发者限制颁发的Token的使用范围。萤石对开发者在颁发Token时的安全建议如下——

* 设置尽可能小的Token有效时间。如果Token马上就会被接收方使用，你可以控制Token过期时间仅为10秒钟。10秒之后Token会自动失效，这将大大增加攻击者利用的难度。
* 在Token中写入授权设备的特征码、设备序列号等操作参数，当终端持有Token在萤石云进行验证时，这些参数会被检查与Token中的是否一致。
* 针对开放平台网关操作，设置Token参数的UrlPattern属性，限定Token持有者仅可能凭借Token访问符合该模式的URL。
* 针对支持一次性操作的Token，设置useOnceOnly属性为true，指定该Token在萤石云上只能使用一次。
* 针对开放平台网关操作，设置Token参数的自定义属性（setAttribute），在访问网关时也会校验这部分参数。

总之，这些措施的目的都是为了更精确的秒数Token持有者在访问萤石云平台时的行为，确保Token颁发后不会被滥用。

## 示例

### E1：取流操作的Token

以取流Token为例，取流授权中，您需要指定取流的设备序列号和通道号，从而方式持有人利用该Token去访问其他设备。

```javascript
const { StreamTokenGenerator } = require('@ezviz/openplatform-sdk').Auth;

let generator = new StreamTokenGenerator();
generator.init(APP_KEY, SECRET_KEY);

const token = generator.generateToken({
  actionType: 1,                 // 是预览还是回放
  deviceSerial: 'D12356643',     // 设备序列号
  channel: '1',
  expire: 900,                   // 取流过期时间，15分钟
  expire2: 28800                 // 取流后用户播放有效期，8小时
  terminalIP: '172.56.22.134',   // 终端IP
  isUseOnceOnly: false           // 指定该取流Token只能使用1次
});

```

### E2: 颁发Token访问开放平台网关：授权可访问的API URL

可以在代码中设置UrlPattern，来限制Token持有者访问网关的API接口

```javascript
const { NonDeviceOpsTokenGenerator } = require('@ezviz/openplatform-sdk').Auth;

let generator = new NonDeviceOpsTokenGenerator();
generator.init(APP_KEY, SECRET_KEY);

const token = generator.generateToken({
  appId: 'app01',
  userId: '',
  expire: 3600,
  urlPattern: '/api/v3/conference/**'
});

//上述代码生成了一个Token，与应用app01关联，持有者可以在1小时（3600秒）内，访问/api/v3/conference/ 开头的所有API。
//其中**可以匹配任意层级的任意URL。
```

URLPattern的使用规则如下

* `?` 可以匹配任意单个字符。（不含目录分隔符 `/`）
* `*` 可以匹配任意0到n个字符。（不含目录分隔符 `/`）
* `** ` 可以匹配任意层级的目录和字母。

### E3：颁发Token访问开放平台网关：授权可操作的设备

对于设备操作，Token颁发者应该将操作范围限定在一个设备上或一个通道上。

```javascript
const { DeviceGeneralTokenGenerator } = require('@ezviz/openplatform-sdk').Auth;

let generator = new DeviceGeneralTokenGenerator();
generator.init(APP_KEY, SECRET_KEY);

const token = generator.generateToken({
  action: 'ALL',
  deviceSerial: 'D12356643',                // 设备序列号
  channel: '1',
  terminalIP: '172.56.22.134',              // 设定该Token只能被指定终端IP使用
  urlPattern: '/api/lapp/device/capture',   // 设置该Token只能调用抓图接口
  isUseOnceOnly: true,                      // 指定该取流Token只能使用1次
  expire: 60                                // 设定有效时间
});
```

### E4: 颁发Token访问开放平台网关：限定HTTP请求参数

对于其他类型的操作，Token颁发者应当限制请求中的关键参数。以防止Token使用者越权操作。

```javascript
const { NonDeviceOpsTokenGenerator } = require('@ezviz/openplatform-sdk').Auth;

let generator = new NonDeviceOpsTokenGenerator();
generator.init(APP_KEY, SECRET_KEY);

// 指定访问开放平台API时，以下两个HTTP参数必须为指定的值。
// 注意参数名（roomid、pairid）必须和实际请求开放平台时的HTTP query参数完全一致，包括大小写。
const attributes = new Map();
attributes.set('roomid', 'room001');
attributes.set('pairid', 'pair001');

const token = generator.generateToken({
  appId: 'app01',
  userId: 'user01',
  expire: 1000,
  urlPattern: '/api/v3/conference/**',
  attributes
});
```

### E5: 颁发RTC Token
```javascript
const { RTCTokenGenerator } = require('@ezviz/openplatform-sdk').Auth;

let generator = new RTCTokenGenerator();
generator.init(APP_KEY, SECRET_KEY);

const token = generator.generateToken({
  appId: 'app01',
  userId: 'user01',   // 设置用户id
  expire: 1000,       // 设置过期时间
  roomId: '12345'     // 设置房间号
});
```

### E6: 颁发资源访问Token
```javascript
const { GeneralResourceTokenGenerator } = require('@ezviz/openplatform-sdk').Auth;

let generator = new GeneralResourceTokenGenerator();
generator.init(APP_KEY, SECRET_KEY);

const attributes = new Map();
attributes.set('strRoomId', 'ID1699430483');            //设置业务参数：strRoomId
attributes.set('customId', '7ca19da6c7164bc5ad7e0a');   //设置业务参数：customId

// 创建一个终端入会的action
const policy = [{name: 'JOIN_ROOM', attributes }];

const token = generator.generateToken({
  appid: 'f758a146b2b24fc7b9705e232bce9f02',
  expire: 604800,
  policy
});

```

## API
### Auth模块
#### 模块方法导入
```javascript
const {
  NonDeviceOpsTokenGenerator,
  DeviceGeneralTokenGenerator,
  RTCTokenGenerator,
  StreamTokenGenerator,
  GeneralResourceTokenGenerator
} = require('@ezviz/openplatform-sdk').Auth;
```

#### API详细说明
| 类名 | 方法名 | 参数 | 返回值 | 说明 |
| - | - | - | - | - |
| NonDeviceOpsTokenGenerator | init | (appKey: string, secretKey: string) | void | 初始化非设备类操作授权Token生成器 |
| NonDeviceOpsTokenGenerator | generateToken | (options: NonDeviceOpsTokenOptions) | string | 生成非设备类操作授权Token |
| DeviceGeneralTokenGenerator | init | (appKey: string, secretKey: string) | void | 初始化设备操作类的授权Token生成器 |
| DeviceGeneralTokenGenerator | generateToken | (options: DeviceGeneralTokenOptions) | string | 生成设备操作类的授权Token |
| RTCTokenGenerator | init | (appKey: string, secretKey: string) | void | 初始化RTC操作Token生成器 |
| RTCTokenGenerator | generateToken | (options: RTCTokenOptions) | string | 生成RTC操作Token |
| StreamTokenGenerator | init | (appKey: string, secretKey: string) | void | 初始化取流操作Token生成器 |
| StreamTokenGenerator | generateToken | (options: StreamTokenOptions) | string | 生成取流操作Token |
| GeneralResourceTokenGenerator | init | (appKey: string, secretKey: string) | void | 初始化资源访问Token生成器 |
| GeneralResourceTokenGenerator | generateToken | (options: GeneralResourceTokenOptions) | string | 生成资源访问Token |

#### 类型定义
##### NonDeviceOpsTokenOptions
```typescript
interface NonDeviceOpsTokenOptions {
  appId?: string;                     // 应用ID
  userId?: string;                    // 用户ID
  expire: number;                     // 过期时间，单位秒
  urlPattern?: string;                // URL模式
  time?: number;                      // 当前时间，单位秒
  attributes?: Map<string, string>;   // 自定义属性
  isUseOnceOnly?: boolean;            // 是否一次性，默认false
}
```

##### DeviceGeneralTokenOptions
```typescript
interface DeviceGeneralTokenOptions {
  appId?: string;                     // 应用ID
  action: string;                     // 操作类型，此处使用‘ALL’
  deviceSerial: string;               // 设备序列号
  channel: string;                    // 通道号
  expire: number;                     // 过期时间，单位秒
  resourceCatagory?: string;          // 资源类型
  terminalIP?: string;                // 终端IP
  urlPattern?: string;                // URL模式
  time?: number;                      // 当前时间，单位秒
  attributes?: Map<string, string>;   // 自定义属性
  isUseOnceOnly?: boolean;            // 是否一次性
  
}
```

##### RTCTokenOptions
```typescript
interface RTCTokenOptions {
  appId: string;          // 应用ID
  userId: string;         // 用户ID
  expire: number;         // 过期时间，单位秒
  roomId: string;         // 房间号
  time?: number;          // 当前时间，单位秒
}
```

##### StreamTokenOptions
```typescript
interface StreamTokenOptions {
  appId?: string;              // 应用ID
  actionType: number;          // 操作类型，0-预览，1-回放，2-对讲
  deviceSerial: string;        // 设备序列号
  channel: string;             // 通道号
  expire: number;              // 过期时间，单位秒
  expire2: number;             // 持续播放时间，单位秒
  resourceCatagory?: string;   // 资源类型
  terminalIP: string;          // 终端IP
  time?: number;               // 当前时间，单位秒
  isUseOnceOnly?: boolean;     // 是否一次性
}
```

##### GeneralResourceTokenOptions
```typescript
interface GeneralResourceTokenOptions {
  appid: string;    // 应用ID
  expire: number;   // 过期时间，单位秒
  policy: Array<{name: string, attributes: Map<string, string>}>; // 策略
  time?: number;    // 当前时间，单位秒
}
```