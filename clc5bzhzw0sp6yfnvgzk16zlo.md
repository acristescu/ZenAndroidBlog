# Monitoring WebSockets with Stetho

![Monitoring WebSockets with Stetho](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091290818/TKv7lE6PW.png)

In this article I'm going to discuss how you can extend the functionality of Facebook's excellent Stetho library to debug WebSockets connections on Android. While Stetho allows you to inspect HTTP(S) requests, SQLite databases and Shared Preferences values out of the box, it does omit WebSockets. We are going to write a simple utility class that fixes this.

## Brief Stetho overview

Stetho is a library developed by Facebook that allows an android app (running on an emulator or on a device connected via `adb` to a laptop) to be debugged using the excellent Chrome Debug Tools as if it were a website. You can monitor the HTTP requests your app does, view and modify SQLite databases, inspect the view hierarchy, or view and change shared preferences on the fly. You can check out the library [here](http://facebook.github.io/stetho/).

## The problem

Chrome Debug Tools do support WebSockets, however the folks at Facebook decided to implement support for it in Stetho only halfway. Looking at the code, there seems to be an object called `NetworkEventReporter` which has methods for reporting events such as `webSocketFrameReceived` or `webSocketFrameSent`, but there seems to be no utility classes to call these at the appropriate time for any WebSockets library, not even the popular OkHTTP3.

While working on my pet project [OnlineGo](https://play.google.com/store/apps/details?id=io.zenandroid.onlinego) I often struggled with debugging the fairly complex protocol that is built by the fine folks at OGS on top of Socket.IO (which is in itself a protocol built on top of WebSockets). Sure, you can log the communication on the console, but logcat has its limitations. I thus decided to spend some time making Stetho play nice with OkHttp.

## The solution

Looking at the `NetworkEventReporter` we can identify the following events that we need to intercept and forward to the reporter:

* `webSocketCreated`
* `webSocketClosed`
* `webSocketFrameError`
* `webSocketFrameReceived` (binary and text)
* `webSocketFrameSent` (binary and text)

Luckily, OkHTTP 3 has a nice `WebSocket.Factory` interface that you can simply plug in. This interface is implemented by `OkHttpClient`, but we can override it with our own implementation. It has a single method that gets called by the library when a new websocket connection needs to be initialized:

```kotlin
fun newWebSocket(request: Request, listener: WebSocketListener): WebSocket
```
The plan is then to create a new `WebSocket.Factory` class that has a plain `OkHttpClient` member to which it delegates the `newWebSocket` calls. This new class also wraps the returned `WebSocket` and the provided `WebSocketListener` into new objects that notify the `NetworkEventReporter` when the relevant events occour. Here is the code:

```kotlin
class StethoWebSocketsFactory(private val httpClient: OkHttpClient) : WebSocket.Factory {
    private val reporter = NetworkEventReporterImpl.get()

    override fun newWebSocket(request: Request, listener: WebSocketListener): WebSocket {
        val requestId = reporter.nextRequestId()
        val newListener = StethoWebSocketListener(listener, requestId)
        val wrappedSocket = httpClient.newWebSocket(request, newListener)
        return StethoWebSocket(wrappedSocket, requestId)
    }

    inner class StethoWebSocketListener(
            private val listener: WebSocketListener,
            private val requestId: String
    ) : WebSocketListener() {

        override fun onOpen(webSocket: WebSocket, response: Response) {
            listener.onOpen(webSocket, response)
            reporter.webSocketCreated(requestId, webSocket.request().url().toString())
        }

        override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
            listener.onClosed(webSocket, code, reason)
            reporter.webSocketClosed(requestId)
        }

        override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
            listener.onFailure(webSocket, t, response)
            reporter.webSocketFrameError(requestId, t.message)
        }

        override fun onMessage(webSocket: WebSocket, bytes: ByteString) {
            listener.onMessage(webSocket, bytes)
            reporter.webSocketFrameReceived(
                    SimpleBinaryInspectorWebSocketFrame(requestId, bytes.toByteArray())
            )
        }

        override fun onMessage(webSocket: WebSocket, text: String) {
            listener.onMessage(webSocket, text)
            reporter.webSocketFrameReceived(
                    SimpleTextInspectorWebSocketFrame(requestId, text)
            )
        }
    }

    inner class StethoWebSocket(
            private val wrappedSocket: WebSocket,
            private val requestId: String
    ) : WebSocket {
        private val reporter = NetworkEventReporterImpl.get()

        override fun queueSize() = wrappedSocket.queueSize()

        override fun send(text: String): Boolean {
            reporter.webSocketFrameSent(SimpleTextInspectorWebSocketFrame(requestId, text))
            return wrappedSocket.send(text)
        }

        override fun send(bytes: ByteString): Boolean {
            reporter.webSocketFrameSent(SimpleBinaryInspectorWebSocketFrame(requestId, bytes.toByteArray()))
            return wrappedSocket.send(bytes)
        }

        override fun close(code: Int, reason: String?) = wrappedSocket.close(code, reason)

        override fun cancel() = wrappedSocket.cancel()

        override fun request() = wrappedSocket.request()

    }
}
```
If you are using the raw `WebSockets` directly, then you can simply use the factory as is instead of your `OkHttpClient`, however for `Socket.IO` the following code is needed:

```kotlin
        socket = IO.socket("https://online-go.com", IO.Options().apply {
            transports = arrayOf("websocket")
            if(BuildConfig.DEBUG) {
                webSocketFactory = StethoWebSocketsFactory(httpClient)
            }
        })
```

And that's it really, you can now run the app, fire up Stetho and look under the Network / WebSockets tab and enjoy this:

![Monitoring WebSockets with Stetho](https://cdn.hashnode.com/res/hashnode/image/upload/v1672091292517/WXEx1ZvAh.png)

You can see the class at work in the OnlineGo app [repo](https://github.com/acristescu/OnlineGo/blob/master/app/src/main/java/io/zenandroid/onlinego/ogs/StethoWebSocketsFactory.kt).

## Conclusion

Using the drop-in utility class described above you can easily enable Stetho debugging for your WebSockets based Android application. Happy debugging!