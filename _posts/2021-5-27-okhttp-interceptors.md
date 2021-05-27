---
title: "自定义Okhttp拦截器"
date: 2016-04-18T11:06:21+08:00
draft: true
---

Okhttp提供拦截器来对请求进行拦截做自己想要的一些操作，比如官方提供的[okhttp-logging-interceptor](https://github.com/square/okhttp/tree/master/okhttp-logging-interceptor)拦截请求打印请求和返回的数据的日志。

有时候我们的每个请求都要求添加公共参数或者对每个接口进行加密，采用拦截器可以进行统一处理。

### 添加公共参数
```kotlin
class CommonParametersInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request() //获取request
        val requestBody = request.body() //获取body
        val method = request.method() //获取请求方法
        val url = request.url() //获取url
        val commonParameters = getCommonParameters() //获取公共参数
        if ("GET" == method) {
            val builder = url.newBuilder()
            //添加公共参数
            for ((key, value) in commonParameters) {
                builder.addQueryParameter(key, value)
            }
            return chain.proceed(request.newBuilder().url(builder.build()).build())
        } else if ("POST" == method) {
            //未考虑文件上传的情况
            if (requestBody is FormBody) { //当post请求没有参数时requestBody不是FormBody
                //FormBody没有HttpUrl类似的newBuilder操作，所以需要将原来的参数添加到map中然后构建新的FormBody
                for (i in 0 until requestBody.size()) {
                    commonParameters[requestBody.name(i)] = requestBody.value(i)
                }
            }
            val builder: FormBody.Builder = FormBody.Builder()
            for ((key, value) in commonParameters) {
                if (!TextUtils.isEmpty(key) && !TextUtils.isEmpty(value)) {
                        builder.add(key, value)
                }
            }
            return chain.proceed(request.newBuilder().post(builder.build()).build())
        }
        return chain.proceed(request)
    }
    private fun getCommonParameters(): MutableMap<String, String?> {
        val paramsMap = HashMap<String, String?>()
        //添加公共参数 ...
        return paramsMap
    }
}
```
