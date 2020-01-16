---
layout: post
title: 谨慎使用 iOS 中的计时器
categories: iOS
description: 谨慎使用 iOS 中的计时器
keywords: keyword1, keyword2
---

```
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        
});
    
    NSInteger a = 0;
    for (NSInteger i = 0; i < 10000000000; i ++) {
        a = a + i;
    }
```

