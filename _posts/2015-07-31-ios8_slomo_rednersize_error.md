---
layout: post
title:  "Slomo 영상의 AVVideoCompostion renderSize 오류"
date:   2015-07-31 11:58:00
categories: iOS
---

Slomo로 촬영된 영상을 Photos framework를 이용해서 `AVAsset`으로 변환할때 Video size(width/height) 참조를 주의해야한다.

{% highlight objc %}
PHAsset *asset = ...;

    PHVideoRequestOptions *requestOptions = [[PHVideoRequestOptions alloc] init];
    [requestOptions setDeliveryMode:PHVideoRequestOptionsDeliveryModeAutomatic];
    [requestOptions setNetworkAccessAllowed:YES];
    [requestOptions setVersion:PHVideoRequestOptionsVersionCurrent];

    [[PHImageManager defaultManager] requestAVAssetForVideo:asset options:requestOptions resultHandler:^(AVAsset *asset, AVAudioMix *audioMix, NSDictionary *info) {

		AVVideoComposition *sComposition = [AVVideoComposition videoCompositionWithPropertiesOfAsset:asset];
		
		// Slomo 세로로 촬영된 영상의 경우 잘못된 Size가 리턴된다! 
		CGSize videoSize = sComposition.renderSize;

{% endhighlight %}

위 코드에서 일반 영상의 경우 `AVVideoComposition`의 `renderSize`가 가로/세로로 촬영된 영상 모두 정상적으로 나타나지만, SloMo로 촬영된 영상의 경우, 세로로 촬영된 영상이라도 가로 해상도로 리턴된다. (ex. 1280*720)

이를 회피하기 위해서는 `AVVideoComposition`의 `renderSize`를 사용하지 말고 `PHAsset`의 `pixelWidth`와 `pixelHeight`를 사용하는 것이 좋다.

```
 CGSize videoSize = CGSizeMake(phAsset.pixelWidth, phAsset.pixelHeight);
```
