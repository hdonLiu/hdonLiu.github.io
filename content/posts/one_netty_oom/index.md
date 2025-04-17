---
title: 'Documenting an Out-of-Memory (OOM) Error'
date: 2025-02-21T21:30:20+08:00
draft: false
---

## Time
2025.2.21  8.30 PM

## Background
In the project, MQTT is used for IoT device communication, and Hook is utilized on top of MQTT for ACL and message enhancement. During a load test in the test environment, the Hook service encountered an OOM (Out Of Memory) situation.

## Problem: 
The MQTT monitoring platform is experiencing widespread hook errors.
### Log

```
10:30:09,812 |-INFO in c.q.l.core.rolling.helper.TimeBasedArchiveRemover - Removed  100 MB of files
Exception in thread "exhook-134244" java.lang.OutOfMemoryError: Cannot reserve 2097152 bytes of direct buffer memory (allocated: 2146302585, limit: 2147483648)
at java.base/java.nio.Bits.reserveMemory(Bits.java:178)
at java.base/java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:121)
at java.base/java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:332)
at io.grpc.netty.shaded.io.netty.buffer.PoolArena$DirectArena.allocateDirect(PoolArena.java:717)
at io.grpc.netty.shaded.io.netty.buffer.PoolArena$DirectArena.newChunk(PoolArena.java:692)
at io.grpc.netty.shaded.io.netty.buffer.PoolArena.allocateNormal(PoolArena.java:215)
at io.grpc.netty.shaded.io.netty.buffer.PoolArena.tcacheAllocateNormal(PoolArena.java:197)
at io.grpc.netty.shaded.io.netty.buffer.PoolArena.allocate(PoolArena.java:139)
at io.grpc.netty.shaded.io.netty.buffer.PoolArena.allocate(PoolArena.java:129)
at io.grpc.netty.shaded.io.netty.buffer.PooledByteBufAllocator.newDirectBuffer(PooledByteBufAllocator.java:395)
at io.grpc.netty.shaded.io.netty.buffer.AbstractByteBufAllocator.directBuffer(AbstractByteBufAllocator.java:188)
at io.grpc.netty.shaded.io.netty.buffer.AbstractByteBufAllocator.buffer(AbstractByteBufAllocator.java:124)
at io.grpc.netty.shaded.io.grpc.netty.NettyWritableBufferAllocator.allocate(NettyWritableBufferAllocator.java:51)
at io.grpc.internal.MessageFramer.writeKnownLengthUncompressed(MessageFramer.java:226)
at io.grpc.internal.MessageFramer.writeUncompressed(MessageFramer.java:172)
at io.grpc.internal.MessageFramer.writePayload(MessageFramer.java:143)
at io.grpc.internal.AbstractStream.writeMessage(AbstractStream.java:66)
at io.grpc.internal.ServerCallImpl.sendMessageInternal(ServerCallImpl.java:168)
at io.grpc.internal.ServerCallImpl.sendMessage(ServerCallImpl.java:152)
at io.grpc.ForwardingServerCall.sendMessage(ForwardingServerCall.java:32)
at io.opentelemetry.javaagent.shaded.instrumentation.grpc.v1_6.TracingServerInterceptor$TracingServerCall.sendMessage(TracingServerInterceptor.java:107)
at io.grpc.stub.ServerCalls$ServerCallStreamObserverImpl.onNext(ServerCalls.java:380)
at xxx.xxxx.account.core.mqttHook.HookProviderImpl.onMessagePublish(HookProviderImpl.java:319)
at xxx.xxxx.account.core.mqttHook.exhook.HookProviderGrpc$MethodHandlers.invoke(HookProviderGrpc.java:1518)
at io.grpc.stub.ServerCalls$UnaryServerCallHandler$UnaryServerCallListener.onHalfClose(ServerCalls.java:182)
at io.grpc.PartialForwardingServerCallListener.onHalfClose(PartialForwardingServerCallListener.java:35)
at io.grpc.ForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:23)
at io.grpc.ForwardingServerCallListener$SimpleForwardingServerCallListener.onHalfClose(ForwardingServerCallListener.java:40)
at io.grpc.Contexts$ContextualizedServerCallListener.onHalfClose(Contexts.java:86)
at io.opentelemetry.javaagent.shaded.instrumentation.grpc.v1_6.TracingServerInterceptor$TracingServerCall$TracingServerCallListener.onHalfClose(TracingServerInterceptor.java:152)
at io.grpc.internal.ServerCallImpl$ServerStreamListenerImpl.halfClosed(ServerCallImpl.java:356)
at io.grpc.internal.ServerImpl$JumpToApplicationThreadServerStreamListener$1HalfClosed.runInContext(ServerImpl.java:861)
at io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37)
at io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:133)
at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
at java.base/java.lang.Thread.run(Thread.java:833)
```

### code
```Java
private void start() throws IOException {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(corePoolSize, maxPoolSize, 0L,
                TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(100), new CustomizableThreadFactory("exhook-"));

        server = NettyServerBuilder.forPort(port)
                .addService(hookProvider)
                .executor(threadPoolExecutor)
                .build()
                .start();

        log.info("EMQX gRPC Server started, listening on {}", port);
        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                // Use stderr here since the logger may have been reset by its JVM shutdown hook.
                log.error("*** shutting down EMQX gRPC server since JVM is shutting down");
                try {
                    threadPoolExecutor.shutdown();
                    ExhookServer.this.stop();
                } catch (InterruptedException e) {
                    e.printStackTrace(System.err);
                }
                log.error("*** EMQX gRPC server shut down");
            }
        });
    }
```

### Explanation

I used **Netty** to bind and listen on a port for handling **gRPC** services, which serves as the **MQTT hook**.

But just like the aforementioned error, the callback **onMessagePublish** threw the above error.

## Problem Investigation
1. The startup parameters do not include the configuration of **-XX:MaxDirectMemorySize.**. Therefore, the actual size of MaxDirectMemorySize is equal to the value of **-Xmx**, which is 2GB as shown in the logs.
2. Check the GC (Garbage Collection) situation.Determine if excessive GC pressure led to off-heap memory overflow.
3. add **-Dio.netty.leakDetectionLevel=PARANOID/ADVANCED/SIMPLE/DISABLED**
4. add **-XX:NativeMemoryTracking=detail**  **-XX:+PrintNMTStatistics**