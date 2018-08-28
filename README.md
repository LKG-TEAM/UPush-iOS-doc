---
description: UPushSDK(iOS)文档
---

# UPush-iOS-SDK doc

## 使用到的开源库

* MQTT-Client-Framework\([https://github.com/novastone-media/MQTT-Client-Framework](https://github.com/novastone-media/MQTT-Client-Framework)\)

## 实现机制

## 1. tcp、apns的处理

sdk监听了UIApplication的通知\(UIApplicationWillResignActiveNotification、UIApplicationDidBecomeActiveNotification\)

1. **UIApplicationWillResignActiveNotification**\(app即将失去响应时，mqtt向服务器发送下线消息，再断开mqtt连接，此后的消息服务器会通过apns进行推送\)；
2. **UIApplicationDidBecomeActiveNotification**\(app已经得到响应，mqtt连接、订阅话题、像服务器发送上线消息，此后的消息服务器会通过mqtt进行推送\)；

## 2. 定时器的处理

定时器使用的是gcdtimer，sdk中queue默认为dispatch\_get\_main\_queue\(\)

```text
self.timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_source_set_timer(self.timer, dispatch_time(DISPATCH_TIME_NOW, interval * NSEC_PER_SEC), interval * NSEC_PER_SEC, 0);
        dispatch_source_set_event_handler(self.timer, ^{
            if (!repeats) {
                dispatch_source_cancel(self.timer);
            }
            block();
        });
        dispatch_resume(self.timer);
```

sdk在如下场景使用到定时器：

**1.断线重连；**

* 当连接失败、意外断开\(网络问题等\)时会进行重连，重连间隔是2的幂指数递增，64为最大值\(e.g. 2.4.8.16.32.64...\)

**2.心跳包；**

* sdk内置60秒为间隔的心跳

## 3. stream流方面的线程管理

1. 使用**CFWriteStreamSetDispatchQueue**\(writeStream, self.queue\);和**CFReadStreamSetDispatchQueue**\(readStream, self.queue\);来替代常规开发的stream+runloop组合；
2. 对于self.queue的处理如下代码\(sdk内默认为dispatch\_get\_main\_queue\(\)\)：

```text
- (void)setQueue:(dispatch_queue_t)queue {
    _queue = queue;
    
    // We're going to use dispatch_queue_set_specific() to "mark" our queue.
    // The dispatch_queue_set_specific() and dispatch_get_specific() functions take a "void *key" parameter.
    // Later we can use dispatch_get_specific() to determine if we're executing on our queue.
    // From the documentation:
    //
    // > Keys are only compared as pointers and are never dereferenced.
    // > Thus, you can use a pointer to a static variable for a specific subsystem or
    // > any other value that allows you to identify the value uniquely.
    //
    // So we're just going to use the memory address of an ivar.
    
    dispatch_queue_set_specific(_queue, &QueueIdentityKey, (__bridge void *)_queue, NULL);
}
```

3.流的关闭在主线程：

```text
- (void)close {
    // https://github.com/novastone-media/MQTT-Client-Framework/issues/325
    // We need to make sure that we are closing streams on their queue
    // Otherwise, we end up with race condition where delegate is deallocated
    // but still used by run loop event
    if (self.queue != dispatch_get_specific(&QueueIdentityKey)) {
        dispatch_sync(self.queue, ^{
            [self internalClose];
        });
    } else {
        [self internalClose];
    }
}
```

4.读流的处理：

* **NSStreamEventOpenCompleted**\(成功打开\)
* **NSStreamEventHasBytesAvailable**\(有可读字节\)

在确保调用过NSStreamEventOpenCompleted之后，在NSStreamEventHasBytesAvailable中去readstream，避免在有可读字节之前去readstream造成的阻塞。

```text
- (void)stream:(NSStream *)sender handleEvent:(NSStreamEvent)eventCode {
    if (eventCode & NSStreamEventOpenCompleted) {
        DDLogVerbose(@"[MQTTCFSocketDecoder] NSStreamEventOpenCompleted");
        self.state = MQTTCFSocketDecoderStateReady;
        [self.delegate decoderDidOpen:self];
    }
    
    if (eventCode & NSStreamEventHasBytesAvailable) {
        DDLogVerbose(@"[MQTTCFSocketDecoder] NSStreamEventHasBytesAvailable");
        if (self.state == MQTTCFSocketDecoderStateInitializing) {
            self.state = MQTTCFSocketDecoderStateReady;
        }
        
        if (self.state == MQTTCFSocketDecoderStateReady) {
            NSInteger n;
            UInt8 buffer[768];
            
            n = [self.stream read:buffer maxLength:sizeof(buffer)];
            if (n == -1) {
                self.state = MQTTCFSocketDecoderStateError;
                [self.delegate decoder:self didFailWithError:nil];
            } else {
                NSData *data = [NSData dataWithBytes:buffer length:n];
                DDLogVerbose(@"[MQTTCFSocketDecoder] received (%lu)=%@...", (unsigned long)data.length,
                             [data subdataWithRange:NSMakeRange(0, MIN(256, data.length))]);
                [self.delegate decoder:self didReceiveMessage:data];
            }
        }
    }
    if (eventCode & NSStreamEventHasSpaceAvailable) {
        DDLogVerbose(@"[MQTTCFSocketDecoder] NSStreamEventHasSpaceAvailable");
    }
    
    if (eventCode & NSStreamEventEndEncountered) {
        DDLogVerbose(@"[MQTTCFSocketDecoder] NSStreamEventEndEncountered");
        self.state = MQTTCFSocketDecoderStateInitializing;
        self.error = nil;
        [self.delegate decoderdidClose:self];
    }
    
    if (eventCode & NSStreamEventErrorOccurred) {
        DDLogVerbose(@"[MQTTCFSocketDecoder] NSStreamEventErrorOccurred");
        self.state = MQTTCFSocketDecoderStateError;
        self.error = self.stream.streamError;
        [self.delegate decoder:self didFailWithError:self.error];
    }
}
```

