## 1.前言

今天准备通过Image Tag Parameter插件，从harbor获取镜像tag作为构建参数，Pipeline代码如下

```
        imageTag(
          credentialId: 'HARBOR_PASS', 
          defaultTag: 'latest', 
          filter: '.*', 
          image: IMAGE_NAME, 
          name: 'IMAGE_TAG', 
          registry: 'https://10.xxx.xxx.xxx:8443', 
          verifySsl: false,
          tagOrder: 'DSC_VERSION'
        )
```





在配置了认证信息后，仍然报错401未通过认证。

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230724091943.png)

查看Jenkins运行日志，日志说明应该SS是自签发的harbor证书，无法找到符合的根证书。

```
2023-07-24 01:19:58.607+0000 [id=30]	SEVERE	i.j.p.luxair.ErrorInterceptor#onFail: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
2023-07-24 01:19:58.608+0000 [id=30]	WARNING	i.j.plugins.luxair.ImageTag#getAuthService: Unknown authorization type 
2023-07-24 01:19:58.608+0000 [id=30]	INFO	i.j.plugins.luxair.ImageTag#getAuthToken: Basic authentication
2023-07-24 01:19:58.609+0000 [id=30]	SEVERE	i.j.p.luxair.ErrorInterceptor#onFail
2023-07-24 01:19:58.610+0000 [id=30]	WARNING	i.j.plugins.luxair.ImageTag#getAuthToken: Token not received
2023-07-24 01:19:58.622+0000 [id=30]	WARNING	i.j.plugins.luxair.ImageTag#getImageTagsFromRegistry: HTTP status: Unauthorized
```

## 2.解决

### 2.1 通过浏览器导出harbor域名证书

浏览器点击域名左边的证书，查看证书详细信息，导入格式为：DER编码二进制，单一证书(*.der)

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230724092438.png)

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230724092511.png)

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230724092638.png)

### 2.2 将运行jenkins的jdk导入该证书

将上一步导出的证书文件上传到服务器，我这里使用的jdk版本是openjdk-11，具体的路径因版本不同会有差异，然后执行如下导入命令

```bash
keytool -import -alias harbor -keystore /opt/java/openjdk/lib/security/cacerts  -file 10.xxx.xx.xxx.der
```

然后会询问密码，密码为`changeit`

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230724093153.png)

输入`yes`

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230724093231.png)

### 2.3 然后重启Jenkins

点击构建报错的项目，成功获取到镜像tag

![](https://opszz-1257146428.cos.ap-beijing.myqcloud.com/images/20230724093431.png)