---
{"dg-publish":true,"permalink":"/Program/Mixed/OKHttp实现自定义证书签名请求/","noteIcon":"","created":"2024-12-13T11:46:31.422+08:00"}
---

```kotlin
data class SSLConfig(  
    val keyType: String? = "PKCS12",  
    // 证书签名的base64编码
    val keyFile: String? = null,  
    /**  
     * 密码  
     */  
    val passphrase: String? = null,  
)
fun getCustomCertificateClient(  
    sslConfig: SSLConfig,  
    timeout: Long = 120_000,  
    flowRedirect: Boolean = true,  
    proxy: AreaProxy  
): OkHttpClient {  
    // 加载自定义证书  
    val keyStore = KeyStore.getInstance(sslConfig.keyType ?: KeyStore.getDefaultType())  
    keyStore.load(  
        ByteArrayInputStream(Base64.decodeBase64(sslConfig.keyFile)),  
        sslConfig.passphrase?.toCharArray()  
    )  
    val keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm())  
    keyManagerFactory.init(keyStore, sslConfig.passphrase?.toCharArray())  
  
    val trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm())  
    trustManagerFactory.init(keyStore)  
    val trustManager = trustManagerFactory.trustManagers  
        .first { it is X509TrustManager } as X509TrustManager  
    val trustAllCerts = object : X509TrustManager {  
        override fun checkClientTrusted(chain: Array<out X509Certificate>?, authType: String?) {  
            return trustManager.checkClientTrusted(chain, authType)  
        }  
  
        override fun checkServerTrusted(chain: Array<out X509Certificate>?, authType: String?) {}  
        override fun getAcceptedIssuers(): Array<X509Certificate> = arrayOf()  
    }  
  
    // 创建SSL上下文  
    val sslContext = SSLContext.getInstance("TLS")  
    sslContext.init(keyManagerFactory.keyManagers, arrayOf(trustAllCerts), SecureRandom())  
  
    val builder = OkHttpClient.Builder()  
    builder.connectionSpecs(  
        listOf(  
            ConnectionSpec.CLEARTEXT,  
            ConnectionSpec.COMPATIBLE_TLS,  
            ConnectionSpec.MODERN_TLS,  
            ConnectionSpec.RESTRICTED_TLS  
        )  
    )  
    builder.sslSocketFactory(sslContext.socketFactory, trustManager)  
    builder.hostnameVerifier { hostname: String, session: SSLSession -> true } // 忽略主机名验证  
    builder.connectTimeout(timeout, TimeUnit.MILLISECONDS)  
    builder.readTimeout(timeout, TimeUnit.MILLISECONDS)  
    builder.writeTimeout(timeout, TimeUnit.MILLISECONDS)  
        .followRedirects(flowRedirect)  
        .dispatcher(dispatcher)  
    builder.addInterceptor(HttpUnzippingInterceptor())  
        .proxy(proxy.proxy).proxyAuthenticator(  
            getProxyAuth(username = proxy.username, password = proxy.password)  
        )  
    return builder.build()  
}

```