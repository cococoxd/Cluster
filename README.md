# adventure

## Fast, non-deadlocking parallel image downloader and cache for iOS

[![CocoaPods compatible](https://img.shields.io/cocoapods/v/build_tools.svg?style=flat)](https://cocoapods.org/pods/build_tools)
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![Build status](https://github.com/pinterest/build_tools/workflows/CI/badge.svg)](https://github.com/pinterest/build_tools/actions?query=workflow%3ACI+branch%3Amaster)

[build_toolsManager](Source/Classes/build_toolsManager.h) is an image downloading, processing and caching manager. It uses the concept of download and processing tasks to ensure that even if multiple calls to download or process an image are made, it only occurs one time (unless an item is no longer in the cache). build_toolsManager is backed by **GCD** and safe to **access** from **multiple threads** simultaneously. It ensures that images are decoded off the main thread so that animation performance isn't affected. None of its exposed methods allow for synchronous access. However, it is optimized to call completions on the calling thread if an item is in its memory cache.

build_tools supports downloading many types of files. It, of course, **supports** both **PNGs** and **JPGs**. It also supports decoding **WebP** images if Google's library is available. It even supports **GIFs** and **Animated WebP** via PINAnimatedImageView.

build_tools also has two methods to improve the experience of downloading images on slow network connections. The first is support for **progressive JPGs**. This isn't any old support for progressive JPGs though: build_tools adds an attractive blur to progressive scans before returning them.

![Progressive JPG with Blur](/progressive.gif "Looks better on device.")

[build_toolsCategoryManager](Pod/Classes/build_toolsCategoryManager.h) defines a protocol which UIView subclasses can implement and provide easy access to
build_toolsManager's methods. There are **built-in categories** on **UIImageView**, **PINAnimatedImageView** and **UIButton**, and it's very easy to implement a new category. See [UIImageView+build_tools](/Pod/Classes/Image Categories/UIImageView+build_tools.h) of the existing categories for reference.


### Download an image and set it on an image view:

**Objective-C**
```objc
UIImageView *imageView = [[UIImageView alloc] init];
[imageView pin_setImageFromURL:[NSURL URLWithString:@"http://pinterest.com/kitten.jpg"]];
```

**Swift**
```swift
let imageView = UIImageView()
imageView.pin_setImage(from: URL(string: "https://pinterest.com/kitten.jpg")!)
```

### Download a progressive jpeg and get attractive blurred updates:

**Objective-C**
```objc
UIImageView *imageView = [[UIImageView alloc] init];
[imageView setPin_updateWithProgress:YES];
[imageView pin_setImageFromURL:[NSURL URLWithString:@"http://pinterest.com/progressiveKitten.jpg"]];
```

**Swift**
```swift
let imageView = UIImageView()
imageView.pin_updateWithProgress = true
imageView.pin_setImage(from: URL(string: "https://pinterest.com/progressiveKitten.jpg")!)
```

### Download a WebP file

**Objective-C**
```objc
UIImageView *imageView = [[UIImageView alloc] init];
[imageView pin_setImageFromURL:[NSURL URLWithString:@"http://pinterest.com/googleKitten.webp"]];
```

**Swift**
```swift
let imageView = UIImageView()
imageView.pin_setImage(from: URL(string: "https://pinterest.com/googleKitten.webp")!)
```

### Download a GIF and display with PINAnimatedImageView

**Objective-C**
```objc
PINAnimatedImageView *animatedImageView = [[PINAnimatedImageView alloc] init];
[animatedImageView pin_setImageFromURL:[NSURL URLWithString:@"http://pinterest.com/flyingKitten.gif"]];
```

**Swift**
```swift
let animatedImageView = PINAnimatedImageView()
animatedImageView.pin_setImage(from: URL(string: "http://pinterest.com/flyingKitten.gif")!)
```

### Download and process an image

**Objective-C**
```objc
UIImageView *imageView = [[UIImageView alloc] init];
[self.imageView pin_setImageFromURL:[NSURL URLWithString:@"https://i.pinimg.com/736x/5b/c6/c5/5bc6c5387ff6f104fd642f2b375efba3.jpg"] processorKey:@"rounded" processor:^UIImage *(build_toolsManagerResult *result, NSUInteger *cost)
 {
     CGSize targetSize = CGSizeMake(200, 300);
     CGRect imageRect = CGRectMake(0, 0, targetSize.width, targetSize.height);
     UIGraphicsBeginImageContext(imageRect.size);
     UIBezierPath *bezierPath = [UIBezierPath bezierPathWithRoundedRect:imageRect cornerRadius:7.0];
     [bezierPath addClip];

     CGFloat sizeMultiplier = MAX(targetSize.width / result.image.size.width, targetSize.height / result.image.size.height);

     CGRect drawRect = CGRectMake(0, 0, result.image.size.width * sizeMultiplier, result.image.size.height * sizeMultiplier);
     if (CGRectGetMaxX(drawRect) > CGRectGetMaxX(imageRect)) {
         drawRect.origin.x -= (CGRectGetMaxX(drawRect) - CGRectGetMaxX(imageRect)) / 2.0;
     }
     if (CGRectGetMaxY(drawRect) > CGRectGetMaxY(imageRect)) {
         drawRect.origin.y -= (CGRectGetMaxY(drawRect) - CGRectGetMaxY(imageRect)) / 2.0;
     }

     [result.image drawInRect:drawRect];

     UIImage *processedImage = UIGraphicsGetImageFromCurrentImageContext();
     UIGraphicsEndImageContext();
     return processedImage;
 }];
```

**Swift**
```swift
let imageView = FLAnimatedImageView()
imageView.pin_setImage(from: URL(string: "https://i.pinimg.com/736x/5b/c6/c5/5bc6c5387ff6f104fd642f2b375efba3.jpg")!, processorKey: "rounded")  { (result, unsafePointer) -> UIImage? in

    guard let image = result.image else { return nil }

    let radius : CGFloat = 7.0
    let targetSize = CGSize(width: 200, height: 300)
    let imageRect = CGRect(x: 0, y: 0, width: targetSize.width, height: targetSize.height)

    UIGraphicsBeginImageContext(imageRect.size)

    let bezierPath = UIBezierPath(roundedRect: imageRect, cornerRadius: radius)
    bezierPath.addClip()

    let widthMultiplier : CGFloat = targetSize.width / image.size.width
    let heightMultiplier : CGFloat = targetSize.height / image.size.height
    let sizeMultiplier = max(widthMultiplier, heightMultiplier)

    var drawRect = CGRect(x: 0, y: 0, width: image.size.width * sizeMultiplier, height: image.size.height * sizeMultiplier)
    if (drawRect.maxX > imageRect.maxX) {
        drawRect.origin.x -= (drawRect.maxX - imageRect.maxX) / 2
    }
    if (drawRect.maxY > imageRect.maxY) {
        drawRect.origin.y -= (drawRect.maxY - imageRect.maxY) / 2
    }

    image.draw(in: drawRect)

    UIColor.red.setStroke()
    bezierPath.lineWidth = 5.0
    bezierPath.stroke()

    let ctx = UIGraphicsGetCurrentContext()
    ctx?.setBlendMode(CGBlendMode.overlay)
    ctx?.setAlpha(0.5)

    let logo = UIImage(named: "white-pinterest-logo")
    ctx?.scaleBy(x: 1.0, y: -1.0)
    ctx?.translateBy(x: 0.0, y: -drawRect.size.height)

    if let coreGraphicsImage = logo?.cgImage {
        ctx?.draw(coreGraphicsImage, in: CGRect(x: 90, y: 10, width: logo!.size.width, height: logo!.size.height))
    }

    let processedImage = UIGraphicsGetImageFromCurrentImageContext()
    UIGraphicsEndImageContext()

    return processedImage

}
```

### Handle Authentication

**Objective-C**
```objc
[[build_toolsManager sharedImageManager] setAuthenticationChallenge:^(NSURLSessionTask *task, NSURLAuthenticationChallenge *challenge, build_toolsManagerAuthenticationChallengeCompletionHandler aCompletion) {
aCompletion(NSURLSessionAuthChallengePerformDefaultHandling, nil)];
```

**Swift**
```swift
build_toolsManager.shared().setAuthenticationChallenge { (task, challenge, completion) in
  completion?(.performDefaultHandling, nil)
}
```

### Support for high resolution images
Currently there are two ways build_tools is supporting high resolution images:
1. If the URL contains an `_2x.` or an `_3x.` postfix it will be automatically handled by build_tools and the resulting image will be returned at the right scale.
2. If it's not possible to provide an URL with an `_2x.` or `_3x.` postfix, you can also handle it with a completion handler:
```objc
NSURL *url = ...;
__weak UIImageView *weakImageView = self.imageView;
[self.imageView pin_setImageFromURL:url completion:^(build_toolsManagerResult * _Nonnull result) {
  CGFloat scale = UIScreen.mainScreen.scale;
  if (scale > 1.0) {
    UIImage *image = result.image;
    weakImageView.image = [UIImage imageWithCGImage:image.CGImage scale:scale orientation:image.imageOrientation];
    }
}];
```

### Set some limits
```
// cache is an instance of PINCache as long as you haven't overridden defaultImageCache
PINCache *cache = (PINCache *)[[build_toolsManager sharedImageManager] cache];
// Max memory cost is based on number of pixels, we estimate the size of one hundred 600x600 images as our max memory image cache.
[[cache memoryCache] setCostLimit:600 * [[UIScreen mainScreen] scale] * 600 * [[UIScreen mainScreen] scale] * 100];

// ~50 MB
[[cache diskCache] setByteLimit:50 * 1024 * 1024];
// 30 days
[[cache diskCache] setAgeLimit:60 * 60 * 24 * 30];
```

## Installation

### CocoaPods

Add [build_tools](http://cocoapods.org/?q=name%3Abuild_tools) to your `Podfile` and run `pod install`.

If you'd like to use WebP images, add [build_tools/WebP](http://cocoapods.org/?q=name%3Abuild_tools) to your `Podfile` and run `pod install`.


### Carthage

Add `github "pinterest/build_tools"` to your Cartfile . See [Carthage's readme](https://github.com/Carthage/Carthage) for more information on integrating Carthage-built frameworks into your project.

### Manually

[Download the latest tag](https://github.com/Pinterest/build_tools/tags) and drag the `Pod/Classes` folder into your Xcode project. You must also manually link against [PINCache](https://github.com/pinterest/PINCache).

Install the docs by double clicking the `.docset` file under `docs/`, or view them online at [cocoadocs.org](http://cocoadocs.org/docsets/build_tools/)

### Git Submodule

You can set up build_tools as a submodule of your repo instead of cloning and copying all the files into your repo. Add the submodule using the commands below and then follow the manual instructions above.

    git submodule add https://github.com/pinterest/build_tools.git
    git submodule update --init



## Requirements

__build_tools__ requires iOS 12.0 or greater.

## Contact

[Garrett Moon](mailto:garrett@pinterest.com)
[@garrettmoon](https://twitter.com/garrettmoon)
[Pinterest](https://www.pinterest.com/garrettlunar/)

## License

Copyright 2015 Pinterest, Inc.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. [See the License](LICENSE.txt) for the specific language governing permissions and limitations under the License.
