---
layout: post
title:  "AVAssetResourceLoaderDeleagte를 이용한 비디오 캐시 구현"
date:   2016-06-23 10:49:00
categories: iOS
---

`AVAssetResourceLoaderDeleagte`는 custom protocol을 통해 재생하고자 하는 미디어파일을 받아올 때 사용하는 `AVURLAsset`의 프로퍼티이다.
이 프로퍼티가 설정된 상태에서 `test://{MOVIE_FILE}`과 같은 custom protocol 주소의 미디어 재생 요청이 들어오면

{% highlight objc %}
id<AVAssetResourceLoaderDelegate> loaderDelegate = ...;

NSURL * url = [NSURL URLWithString:@"aaa://techslides.com/demos/sample-videos/small.mp4"];
AVURLAsset * asset = [AVURLAsset assetWithURL:url];
[asset.resourceLoader setDelegate:loaderDelegate queue:dispatch_get_main_queue()];
AVPlayerItem * item = [[AVPlayerItem alloc] initWithAsset:asset];
AVPlayer * player = [AVPlayer playerWithPlayerItem:item];

{% endhighlight %}

`AVAssetResourceLoaderDelegate` 프로토콜을 구현한 객체에 `AVAssetResourceLoadingRequest`를 전달하여 재생에 필요한 미디어 조각을 요청한다.


```objc
- (BOOL)resourceLoader:(AVAssetResourceLoader *)resourceLoader shouldWaitForLoadingOfRequestedResource:(AVAssetResourceLoadingRequest *)loadingRequest {
	...
}
```

만약, 이미 요청한 미디어 조각이 더이상 필요없게 되었을 경우(ex.이미 다른 요청으로 받았거나 재생 취소가 되었을 때 등)에는 아래 메서드가 호출되어 해당하는 미디어 조각을 더이상 받지 말 것을 요청한다.


```objc
- (void)resourceLoader:(AVAssetResourceLoader *)resourceLoader didCancelLoadingRequest:(AVAssetResourceLoadingRequest *)loadingRequest {
	...
}
```


이 방법은 AVPlayer가 원격에 위치한 미디어 파일을 받아서 재생하기 전에 직접 네트워크 요청을 핸들링할 수 있기 때문에 미디어파일을 캐싱하는데 사용할 수도 있다.

> 주의할 점! 일반적인 프로토콜로 된 미디어를 요청할 경우(ex. http://{MEDIA_URL}) `AVAssetResourceLoaderDelegate`를 구현한 객체가 프로퍼티로 설정되어 있더라도 실제로 delegate method 들이 호출되지는 않는다.


```objc
- (BOOL)resourceLoader:(AVAssetResourceLoader *)resourceLoader shouldWaitForLoadingOfRequestedResource:(AVAssetResourceLoadingRequest *)loadingRequest
{
    NSMutableURLRequest * request = loadingRequest.request.mutableCopy;
    NSURLComponents * comps = [[NSURLComponents alloc] initWithURL:request.URL resolvingAgainstBaseURL:NO];
    comps.scheme = @"http";
    request.URL = comps.URL;
    request.cachePolicy = NSURLRequestReloadIgnoringLocalCacheData;

    NSLog(@"request: %@", request.allHTTPHeaderFields[@"Range"]);

    if (loadingRequest.dataRequest) {
        long long offset = loadingRequest.dataRequest.requestedOffset;
        NSInteger length = loadingRequest.dataRequest.requestedLength;
        NSString *rangeValue = [NSString stringWithFormat:@"bytes=%llu-%llu", offset, offset + length - 1];
        [request setValue:rangeValue forHTTPHeaderField:@"Range"];
    }

    NSURLConnection * connection = [[NSURLConnection alloc] initWithRequest:request delegate:self startImmediately:NO];

    _connectionMap[[NSString stringWithFormat:@"%p", loadingRequest]] = connection;
    _requestMap[[NSString stringWithFormat:@"%p", loadingRequest]] = loadingRequest;

    [connection start];

    return YES;
}

- (void)resourceLoader:(AVAssetResourceLoader *)resourceLoader didCancelLoadingRequest:(AVAssetResourceLoadingRequest *)loadingRequest
{
    // The resources that's no longer required are canceled from this method.
    // ** In case of running on simulator is doing correctly.
    // ** but, this method never call at running on device only.

    NSLog(@"cancel: %@", loadingRequest.request.allHTTPHeaderFields[@"Range"]);

    [self removeRequest:loadingRequest];
}

- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response
{
    NSString *contentType = [response MIMEType];
    unsigned long long contentLength = [response expectedContentLength];

    NSString *rangeValue = [(NSHTTPURLResponse *)response allHeaderFields][@"Content-Range"];
    if (rangeValue)
    {
        NSArray *rangeItems = [rangeValue componentsSeparatedByString:@"/"];
        if (rangeItems.count > 1)
        {
            contentLength = [rangeItems[1] longLongValue];
        }
        else
        {
            contentLength = [response expectedContentLength];
        }
    }

    AVAssetResourceLoadingRequest * request = [self loadingRequestForConnection:connection];
    request.response = response;
    request.contentInformationRequest.contentLength = contentLength;
    request.contentInformationRequest.contentType = contentType;
    request.contentInformationRequest.byteRangeAccessSupported = YES;
}

- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)data
{
    NSLog(@"download bytes: %f", self.downloadBytes);

    self.downloadBytes += data.length;
    AVAssetResourceLoadingRequest * request = [self loadingRequestForConnection:connection];
    [request.dataRequest respondWithData:data];
}

- (void)connectionDidFinishLoading:(NSURLConnection *)connection
{
    NSLog(@"finished: %@", connection.originalRequest.allHTTPHeaderFields[@"Range"]);
    AVAssetResourceLoadingRequest * request = [self loadingRequestForConnection:connection];
    [request finishLoading];
    [self removeRequest:request];
}
```


**문제는 `resourceLoader:didCancelLoadingRequest:`가 시뮬레이터와는 달리 디바이스에서는 절대로 호출되지 않는다.**
이 문제는 아직 해결하지 못한 문제인데, 이로 인해 디바이스에서는 겹치는 부분에 대한 네트워크 요청 취소가 정상적으로 이뤄지지 않아서 실제 동영상 파일 크기보다 훨씬 더 많은 네트워크 트래픽을 소모하는 문제를 야기한다.

이로 인해 디바이스에서는 겹치는 부분에 대한 네트워크 요청 취소가 정상적으로 이뤄지지 않아서 실제 동영상 파일 크기보다 훨씬 더 많은 네트워크 트래픽을 소모하는 문제를 야기한다.

이 문제에 대해 **WWDC16**에서 애플 엔지니어에게 문의했는데 제대로 된 답변을 듣지 못했다.
다만, 문의하러 갔을 때 같은 문제를 겪고 있는 다른 2팀을 만났는데 모두들 동일한 문제를 겪고 있는 것으로 보아 **OS 버그로 추측**된다.
해당 이슈에 대해서 애플에 우선 버그리포팅을 해두었으니 해결되기를 기다리는 수밖에 없을 듯 하다.
