# Coroutine check list (in progress)

## Inspiration

* [Coroutine](https://kotlinlang.org/docs/reference/coroutines-overview.html)
## Không sử dụng GlobalScope 
+ Ko sử dụng GlobalScope - mà ko quản lý - với **bất cứ** lý do gì

__BAD:__

```kotlin
GlobalScope.launch {
  // to do something
}
```

__GOOD:__

```kotlin
CoroutineScope(coroutineContext).launch {
  // to do something  
}
```
## withContext() vs async-await


__BAD:__

```kotlin
val defer1 = async {// do some thing}
val result1 = defer1.await()

val defer2 = async {// do some thing}
val result2 = defer1.await()
```
__GOOD:__

```kotlin
 val result1 = withContext(coroutineContext){// do some thing}
 val result2 = withContext(coroutineContext){// do some thing}
```
**async-async-await-await** <= *chỉ sử dụng async trong trường hợp cần chạy song song như thế này*
## Không khởi tạo thêm Scope
+ Ko khởi tạo thêm scope ko cần thiết 

__BAD:__

```kotlin
internal suspend fun loadUrl(path: String): Int = suspendCoroutine { continuation ->
        CoroutineScope(Dispatchers.Main).launch { // khởi tạo CoroutineScope(Dispatchers.Main) <= ko cần thiết
            webView?.let {
                it.webViewClient = object : WebViewClient() {
                    override fun shouldOverrideUrlLoading(
                        view: WebView,
                        request: WebResourceRequest
                    ): Boolean {
                        it.loadUrl(request.url.toString())
                        return true
                    }
                    override fun onPageFinished(view: WebView?, url: String?) {
                        isLoadURL = true
                        continuation.resume(0)
                    }
                }
                it.loadUrl(path)
            }
        }
        //
    }
```

__GOOD:__

```kotlin
private suspend fun loadUrl(path: String): Int = withContext(Dispatchers.Main) { // chuyển sang Dispatchers.Main
        suspendCancellableCoroutine<Int> {continuation-> //suspendCancellableCoroutine thường được sử dụng cho các trường hợp 'nắn' callback thành suspend function
            webView?.let {
                it.webViewClient = object : WebViewClient() {
                    override fun shouldOverrideUrlLoading(
                            view: WebView,
                            request: WebResourceRequest
                    ): Boolean {
                        it.loadUrl(request.url.toString())
                        return true
                    }
                    override fun onPageFinished(view: WebView?, url: String?) {
                        isLoadURL = true
                        continuation.resumeWith(Result.success(0)) // resume coroutine - 'nắn' callback thành suspend function

                    }
                }
                it.loadUrl(path)
            }
        }
//        0
    }
```

## Prefer dùng suspendCancellableCoroutine hơn là suspendCoroutine


__BAD:__

```kotlin
suspend fun doSomeThing():State{
      return suspendCoroutine { suspendCoroutine ->
           // do some thing
      }
}
```
__GOOD:__

```kotlin
suspend fun doSomeThing():State{
      return suspendCancellableCoroutine { suspendCancellableCoroutine ->
           // do some thing
      }
}
```

## Channel 
+ Channel dùng cho việc gửi nhận giữa các Coroutine
+ Channel dùng làm blocking queue
=> Ngoài hai lý do ý, ko nên dùng channel. Channel nên được dùng đúng với chức năng mà nó được tạo ra

__BAD:__

```kotlin
suspend fun doSomeThing():State{
      val channel = Channel<State>()
      val job1 = CoroutineScope(Dispatchers.Main + supervisor).launch {
                  // to do some thing
                  channel.send(state1) // (1)
              }
      val job2 = CoroutineScope(Dispatchers.Default + supervisor).launch {
                  // to do some thing
               channel.send(state2) // (2)
      }
      val state = channel.receive() // suspend current coroutine and wait data send from (1) hoặc (2) - item nào đến trước thì cancel cái còn lại
      job1.cancel()  
      job2.cancel() 
      channel.close()
      // do something with state
      return state
}
```
Đoạn lệnh trên dùng channel với một lý do DUY NHẤT là suspend - chờ nhận dữ liệu, nên có thể đổi bằng 1 cách khác như dưới

__GOOD:__

```kotlin
suspend fun doSomeThing():State{
      return suspendCancellableCoroutine { suspendCancellable ->
            CoroutineScope(Dispatchers.Main + supervisor).launch {
                  // to do some thing
                  if (suspendCancellable.isActive) {
                      suspendCancellable.resumeWith(Result.success(state1))
                      suspendCancellable.cancel()
                  }
              }
              CoroutineScope(Dispatchers.Default + supervisor).launch {
                  // to do some thing
                  if (suspendCancellable.isActive) {
                      suspendCancellable.resumeWith(Result.success(state2))
                      suspendCancellable.cancel()
                  }
              }
      }
}
```
## Hãy cẩn thận khi cancel Chanel
+ channel.send() và channel.receive() đều suspend current coroutine(coroutine gọi ra nó), khi gọi channel.cancel() - sender coroutine & receive coroutine đều bị cancel() - nếu sender và receiver là root coroutine - các job sẽ bị huỷ theo

## Shared mutable state and concurrency / Thread safe
T.B.D
## Table of Contents
T.B.D

## Language

Use `vi-VN` Tiếng Việt


## Copyright Statement

The following copyright statement should be included at the top of every source file:

```
/* 
 * Copyright (c) 2020 baka3k
 * 
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 * 
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 * 
 * Notwithstanding the foregoing, you may not use, copy, modify, merge, publish,
 * distribute, sublicense, create a derivative work, and/or sell copies of the
 * Software in any work that is designed, intended, or marketed for pedagogical or
 * instructional purposes related to programming, coding, application development,
 * or information technology.  Permission for such use, copying, modification,
 * merger, publication, distribution, sublicensing, creation of derivative works,
 * or sale is expressly withheld.
 * 
 * This project and source code may use libraries or frameworks that are
 * released under various Open-Source licenses. Use of those libraries and
 * frameworks are governed by their own individual licenses.
 * 
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 * THE SOFTWARE.
 */
 ```

## Credits

- [baka3k](baka3k@gmail.com)

