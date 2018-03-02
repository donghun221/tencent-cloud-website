# 临时密钥生成及使用指引 
CAM 的 STS 临时密钥，以云 API 的形式提供。目前云 API 提供各种语言的
SDK，用户可以使用云 API 或 SDK 来申请临时密钥三元组。

## 通过 API 获取
### STS 云 API 接口参数

- STS 云 API 的接口参数说明 如下：

| 字段            | 是否必选 | 类型   | 描述                                                                                                                     |
|:-----------------|:----------:|:--------:|:--------------------------------------------------------------------------------------------------------------------------|
| name            | 是       | String | 用户昵称                                                                                                                 |
| policy          | 是       | String | 策略语法                                                                                                                 |
| durationSeconds | 否       | Int    | 指定临时证书的有效期，单位：秒。不传的话默认 1800 秒, 最长有效期: 2 小时（7200 秒）                                                                       |
| Signature       | 是       | String | 请求签名，用来验证此次请求的合法性，需要用户根据实际的输入参数计算得出。 计算方法可参考 [*签名方法*](https://cloud.tencent.com/document/api/377/4214) 章节                                         |
| Timestamp       | 是       | Int    | 时间戳                                                                                                                   |
| Nonce           | 是       | Int    | 随机数                                                                                                                   |
| SecretId        | 是       | String | 在云 API 密钥上申请的标识身份的 SecretId，一个 SecretId 对应唯一的 SecretKey , 而 SecretKey 会用来生成请求签名 Signature |
| Action          | 是       | String | Action = GetFederationToken                                                                                              |
| Region          | 否       | String | 地域参数，用来标识希望操作哪个地域的实例                                                                                 |

- 返回值：

| 字段            | 类型   | 描述                                                 |
|----------------|:--------:|:------------------------------------------------------|
| credentials    | Object | 对象里面包含 Token，tmpSecretId，tmpSecretKey 三元组     |
| --token        | String | Token 值做鉴权时使用                               	  |
| --tmpSecretId  | String | tmpSecretId 签名时使用                               |
| --tmpSecretKey | String | tmpSecretKey 签名时使用                              |
| federatedUser  | String | 返回标识用户。示例：qcs::sts::123456789012:federated-user/Bob       |
| expiredTime    | Int    | 证书无效的时间戳                                     |

- 访问请求示例
> ht<span>tps://</span>sts.api.qcloud.com/v2/index.php?Action=GetFederationToken&Nonce=745162946&Region=ap-guangzhou&RequestClient=SDK\_JAVA\_2.0.4&SecretId=AKIDgFgFj4bSobJ2e8Lgq7JK9nFcsyESoVKB&Signature=UvuOAMrQSDOz2eFKReaNgnw90bRLwJuVZRn13i9vqyQ=&SignatureMethod=HmacSHA256&Timestamp=1519968698&name=my nick name&policy={“statement":\[{"action":\["cos:GetObject"\],"effect":"allow","resource":\["qcs::cos:ap-guangzhou:uid/123456789:prefix//123456789/demo-bucket/\*"\]}\],"condition":{"ip\_equal":{"qcs:ip": “192.168.0.1/24"}}"version":"2.0"}*


Policy内容示例
``` json
{
    "statement": [
        {
            "effect": "allow",
            "action": [
                "name/cos:GetObject"
            ],
            "resource": [
                "qcs::cos:ap-guangzhou:uid/123456789:prefix//123456789/demo-bucket/*"
            ],
            "condition": {
                "ip_equal": {
                    "qcs:ip": "192.168.0.1/24"
                }
            }
        }
    ],
    "version":"2.0",
}           
```

返回内容示例
``` json
{
	"codeDesc": "Success",
	"message": "",
	"data": {
		"expiredTime": 1494563462,
		"credentials": {
			"sessionToken": "sessionTokenXXXXX",
			"tmpSecretId": "tmpSecretIdXXXXX",
			"tmpSecretKey": "tmpSecretKeyXXXXX"
			}
		},
	"code": 0
}
```

### 策略说明

资源 resource：
----------

资源 resource 元素描述一个或多个操作对象，如 COS 存储桶等。这里主要介绍CAM 的资源描述信息。

**六段式**

所有资源均可采用下述的六段式描述方式。每种产品都拥有其各自的资源和对应的资源定义详情。有关如何指定资源的信息，对于COS的资源描述，如下图所示

	qcs::service\_type:region:account:resource 

- **qcs** 是 qcloud service 的简称，表示是腾讯云的云资源。该字段是必填项。
* **service_type** 描述产品简称，如COS。值为\*的时候表示所有产品。该字段是必填项。
* **region** 描述地域信息。值为空的时候表示所有地域。腾讯云新版地域统一命名方式请参考 [地域和可用区](https://cloud.tencent.com/document/product/213/6091)。腾讯云COS现有的地域命名方式定义如下：

  | 字段         | 类型     |
  |--------------|----------|
  | ap-guangzhou | 广州     |
  | ap-chengdu   | 成都     |
  | ap-beijing   | 北京     |
  | ap-beijing-1 | 北京1区  |
  | ap-shanghai  | 上海     |
  | ap-hongkong  | 香港     |
  | ap-singapore | 新加坡   |
  | na-toronto   | 多伦多   |
  | eu-frankfurt | 法兰克福 |

* **account** 描述资源拥有者的根账号信息。目前支持两种方式描述资源拥有者，uin
和 uid 方式。
	- uin 方式，即根账号的 QQ 号，表示为uin/${uin}，如 uin/12345678；
	- uid 方式，即根账号的 APPID，表示为uid/${appid}，如 uid/10001234。

值为空的时候表示创建策略的 CAM 用户所属的根账号。目前 COS 和 CAS 业务的资源拥有者只能用 uid 方式描述，其他业务的资源拥有者只能用 uin 方式描述。

**resource** 描述各产品的具体资源详情。表示某个资源子类下的带路径的资源 ID，可以有多个资源。COS 产品中，以如下的形式表示：

 1. 使用通配符“\*”来标识 bucket 存储桶中的所有对象

		qcs::cos:ap-guangzhou:uid/123456789:prefix//123456789/bucket/\*

 2. 用Object名字来标识 bucket 存储桶中名为Object的对象
		
		qcs::cos:ap-guangzhou:uid/123456789:prefix//123456789/bucket/object
		

 3. 使用通配符“\*”来标识 bucket 存储桶中以/a/b/c为前缀的所有对象，该方式下，支持目录级的前缀匹配。

		qcs::cos:ap-guangzhou:uid/123456789:prefix//123456789/bucket/a/b/c/\*


效力 effect
----------
用于描述声明产生的结果，包括 allow (允许)和 deny (显式拒绝)两种情况。该元素是必填项。

- 生效条件 condition

用于描述策略生效的约束条件。条件包括操作符、操作键和操作值组成。条件值可包括时间、IP 地址等信息。在COS中目前支持IP地址条件。该元素是非必填项。

下表是COS条件操作符、条件名以及示例的信息。

| 条件操作符     | 含义     | 条件名 | 举例                                                                  |
|----------------|----------|--------|-----------------------------------------------------------------------|
| ip\_equal      | ip等于   | qcs:ip | {"ip\_equal":{"qcs:ip ":"10.121.2.10/24"}}                            |
| ip\_not\_equal | ip不等于 | qcs:ip | {"ip\_not\_equal":{"qcs:ip ":\["10.121.2.10/24", "10.121.2.20/24"\]}} |

请注意，在条件中指定的 IP地址 使用 RFC 4632 中描述的 CIDR表示法。有关详细信息，请转到 <http://www.rfc-editor.org/rfc/rfc4632.txt>。

操作 action
----------
用于描述允许或拒绝的操作。操作可以是 API （以 name前缀描述）。该元素是必填项。

在COS中，可以有多个action。表示方式如下：

	name/cos:操作

- Policy中关于COS的操作列表

COS 对象相关Action列表

| Action                  | COS 操作                                                                           |
|-------------------------|------------------------------------------------------------------------------------|
| PutObject               | [*Put Object*](https://cloud.tencent.com/document/product/436/7749)                |
| PutObjectACL            | [*Put Object ACL*](https://cloud.tencent.com/document/product/436/7748)                                                                 |
| PutObjectCopy           | [*Put Object Copy*](https://cloud.tencent.com/document/product/436/10881)          |
| PutObjectTagging        | Put Object Tagging                                                                        |
| PutObjectVersionAcl     | Put Object Version Acl                                                                        |
| UploadPart              | [*Upload Part*](https://cloud.tencent.com/document/product/436/7750)               |
| UploadPartCopy          | [*Upload Part Copy*](https://cloud.tencent.com/document/product/436/8287)          |
| OptionsObject           | [*Options Object*](https://cloud.tencent.com/document/product/436/8288)            |
| PostObject              | Post Object                                                                          |
| GetObject               | [*Get Object*](https://cloud.tencent.com/document/product/436/7753)                |
| GetObjectACL            | [*Get Object ACL*](https://cloud.tencent.com/document/product/436/7744)            |
| GetObjectTagging        | Get Object Tagging                                                                         |
| GetObjectVersionAcl     | Get Object Version Acl                                                                        |
| HeadObject              | [*Head Object*](https://cloud.tencent.com/document/product/436/7745)               |
| AppendObject            | Append Object                                                                      |
| AbortMultipartUpload    | [*Abort Multipart Upload*](https://cloud.tencent.com/document/product/436/7740)    |
| CompleteMultipartUpload | [*Complete Multipart Upload*](https://cloud.tencent.com/document/product/436/7742) |
| DeleteObject            | [*Delete Object*](https://cloud.tencent.com/document/product/436/7743)             |
| DeleteObjectTagging     | Delete Object Tagging                                                                         |
| InitiateMultipartUpload | [*Initiate Multipart Upload*](https://cloud.tencent.com/document/product/436/7746) |
| ListMultipartUploads    | List Multipart Uploads                                                                         |
| ListParts               | [*List Parts*](https://cloud.tencent.com/document/product/436/7747)                |

COS 存储桶相关Action列表

| Action       | COS 操作                                                                                                |
|--------------|---------------------------------------------------------------------------------------------------------|
| DeleteBucket | [*Delete Bucket*](https://cloud.tencent.com/document/product/436/7732)                                  |
| GetBucket    | [*Get Bucket*](https://cloud.tencent.com/document/product/436/7734)                                     |
| HeadBucket   | [*Head Bucket*](https://cloud.tencent.com/document/product/436/7735)                                    |
| PutBucket    | [*Put Bucket*](https://cloud.tencent.com/document/product/436/7738)                                     |
| GetService   | [*Get Service*](https://cloud.tencent.com/document/product/436/8291) （是一个等同于List Objects的接口） |

COS 存储桶资源相关Action列表

| Action                  | COS 操作                                                                         |
|-------------------------|----------------------------------------------------------------------------------|
| DeleteBucketCORS        | [*Delete Bucket CORS*](https://cloud.tencent.com/document/product/436/8283)      |
| DeleteBucketLifecycle   | [*Delete Bucket Lifecycle*](https://cloud.tencent.com/document/product/436/8284) |
| DeleteBucketOrigin      | Delete Bucket Origin                                                                       |
| DeleteBucketReferer     | Delete Bucket Referer                                                                       |
| DeleteBucketReplication | Delete Bucket Replication                                                                      |
| DeleteBucketTagging     | Delete Bucket Tagging                                                                      |
| DeleteBucketWebsite     | Delete Bucket Website                                                                      |
| DeleteMultipleObjects   | Delete Multiple Objects                                                                     |
| GetBucketACL            | [*Get Bucket ACL*](https://cloud.tencent.com/document/product/436/7733)          |
| GetBucketCORS           | [*Get Bucket CORS*](https://cloud.tencent.com/document/product/436/8274)         |
| GetBucketLifecycle      | [*Get Bucket Lifecycle*](https://cloud.tencent.com/document/product/436/8278)    |
| GetBucketLocation       | [*Get Bucket Location*](https://cloud.tencent.com/document/product/436/8275)     |
| GetBucketLogging        | Get Bucket Logging                                                                     |
| GetBucketNotification   | Get Bucket Notification                                                                       |
| GetBucketObjectVersions | Get Bucket Object Versions                                                                      |
| GetBucketOrigin         | Get Bucket Origin                                                                      |
| GetBucketPolicy         | Get Bucket Policy                                                                     |
| GetBucketReferer        | Get Bucket Referer                                                                     |
| GetBucketReplication    | Get Bucket Replication                                                                     |
| GetBucketTagging        | Get Bucket Tagging                                                                  |
| GetBucketVersionAcl     | Get Bucket Version Acl                                                                  |
| GetBucketVersioning     | Get Bucket Versioning                                                                     |
| GetBucketWebsite        | Get Bucket Website                                                                |
| PutBucketACL            | [*Put Bucket ACL*](https://cloud.tencent.com/document/product/436/7737)          |
| PutBucketCORS           | [*Put Bucket CORS*](https://cloud.tencent.com/document/product/436/8279)         |
| PutBucketLifecycle      | [*Put Bucket Lifecycle*](https://cloud.tencent.com/document/product/436/8280)    |
| PutBucketLogging        | Put Bucket Logging                                                                       |
| PutBucketNotification   | Put Bucket Notification                                                                  |
| PutBucketOrigin         | Put Bucket Origin                                                                       |
| PutBucketPolicy         | Put Bucket Policy                                                                       |
| PutBucketReferer        | Put Bucket Referer                                                                      |
| PutBucketReplication    | Put Bucket Replication                                                                      |
| PutBucketVersionAcl     | Put Bucket Version Acl                                                                     |
| PutBucketVersioning     | Put Bucket Versioning                                                                       |
| PutBucketWebsite        | Put Bucket Website                                                                     

## 通过 SDK 获取

以云 **API** 提供的 [**Java SDK**](https://github.com/QcloudApi/qcloudapi-sdk-java) 为例：

``` java
import com.qcloud.Utilities.Json.JSONObject;

public class Demo {
    public static void main(String[] args) {
        /* 如果是循环调用下面举例的接口，需要从此处开始你的循环语句。切记！ */
        TreeMap<String, Object> config = new TreeMap<String, Object>();
        config.put("SecretId", “Paste your SecretId here");
        config.put("SecretKey", “Paste your SecretKey here");

        /* 请求方法类型 POST、GET */
        config.put("RequestMethod", "GET");

        /* 区域参数，可选: gz: 广州; sh: 上海; hk: 香港; ca: 北美; 等。 */
        config.put("DefaultRegion", “ap-guangzhou");

        QcloudApiModuleCenter module = new QcloudApiModuleCenter(new Sts(), config);

        TreeMap<String, Object> params = new TreeMap<String, Object>();
        /* 将需要输入的参数都放入 params 里面，必选参数是必填的。 */
        /* DescribeInstances 接口的部分可选参数如下 */
        params.put("name", “Paste your preferred name here");
        String policy = "{\"statement\": [{\"action\": [\"name/cos:GetObject\",\"name/cos:PutObject\"],\"effect\": \”allow\",\"resource\":[\"qcs::cos:ap-guangzhou:uid/12345678910:prefix//12345678910/demo-bucket/*\"]}],\"version\": \"2.0\"}";
        params.put(“policy", policy);

        /* 在这里指定所要用的签名算法，不指定默认为 HmacSHA1*/
        //params.put("SignatureMethod", "HmacSHA256");

        /* generateUrl 方法生成请求串, 可用于调试使用 */
        System.out.println(module.generateUrl("GetFederationToken", params));
        String result = null;
        try {
            /* call 方法正式向指定的接口名发送请求，并把请求参数 params 传入，返回即是接口的请求结果。 */
            result = module.call("GetFederationToken", params);
            JSONObject json_result = new JSONObject(result);
            System.out.println(json_result);
        } catch (Exception e) {
            System.out.println("error..." + e.getMessage());
        }
    }
}
```

申请临时三元组时，需要描述策略 Policy，示例中以 IP 做限制的 Policy为例：
``` json
{
	"statement": [
	    {
	        "action": [
	            "name/cos:GetObject",
	            "name/cos:HeadObject"
	        ],
	        "condition": {
	            "ip_equal": {
	                "qcs:ip": [
	                    "101.226.226.185/32"
	                ]
	            }
	        },
	        "effect": "allow",
	        "resource": [
	            "qcs::cos:ap-guangzhou:uid/12345678910:prefix//12345678910/demo-bucket/*"
	        ]
	    }
	],
	"version": "2.0"
}
```

> Policy 描述请参考 [策略语法](https://cloud.tencent.com/document/product/598/10603)。

>注意：  
resource 字段中的 prefix 后跟//。uid后面跟的是appid，而不是UIN。

### 使用临时密钥访问 COS

COS 访问通过x-cos-security-token字段来传递临时 Token，而临时 SecretId 和 SecretKey 则用来生成密钥，以 Jave SDK 为例使用临时密钥访问 COS。

示例如下：从 Github 下载：[Java SDK](https://github.com/tencentyun/cos-java-sdk-v5)
``` java
public class Demo {
	public static void main(String\[\] args) throws Exception {
		// 用户基本信息
		String appId = "12345678910";
		String secretId = "Your SecretId";
		String secretKey = "Your SecretKey";
		String sessionToken = "Your Temporal Token";

		// 设置秘钥
		COSCredentials cred = new BasicSessionCredentials(appId, secretId, secretKey, sessionToken);

		// 设置区域, 这里设置为华北
		ClientConfig clientConfig = new ClientConfig(new Region("ap-guangzhou"));

		// 生成  cos 客户端对象
		COSClient cosClient = new COSClient(cred, clientConfig);

		// 创建 bucket
		// bucket 数量上限 200 个, bucket 是重操作, 一般不建议创建如此多的 bucket
		// 重复创建同名 bucket 会报错
		String bucketName = "demo-bucket";

		// 上传 object, 建议 20M 以下的文件使用该接口
		File localFile = new File("Your file path here");
		String key = "/put\_object\_demo.txt";
		PutObjectRequest putObjectRequest = new PutObjectRequest(bucketName, key, localFile);
		PutObjectResult putObjectResult = cosClient.putObject(putObjectRequest);
		System.out.println(putObjectResult);

		// 关闭客户端 (关闭后台线程)
		cosClient.shutdown();
	}
}
```

## 怎样通过策略生成器方便地生成Policy？

 1. 登陆腾讯云控制台
<img src="/image/Step-1.png" width="624" height="310" />

 2. 进入访问管理页面
<img src="/image/Step-2.png" width="624" height="310" />

 3. 进入策略管理面，并点击新建自定义策略
<img src="/image/Step-3.png" width="624" height="310" />

 4. 在选择服务和操作页面中选择对象存储服务
<img src="/image/Step-4.png" width="624" height="310" />

 5. 选择需要的操作以及资源描述
在这里选择了GetObject操作，资源描述可参照（[*说明*](https://cloud.tencent.com/document/product/598/10606)）。然后，点击添加声明。
<img src="/image/Step-5.png" width="624" height="310" />

 6. 选择下一步
<img src="/image/Step-6.png" width="624" height="310" />

 7. 选择创建策略
<img src="/image/Step-7.png" width="624" height="310" />

 8. 自定义策略列表
完成之后，就能在控制台中看到创建好的策略了。
<img src="/image/Step-8.png" width="624" height="310" />
