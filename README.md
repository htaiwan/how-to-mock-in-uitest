## 如何在UI test中進行stub和mock

### 前言

**一般來說，我們會在Unit test進行stub和mock的方式來排除網路連線問題或者資料格式不正確所造成的測試失敗，那stub和mock是否也可以用在UI test(Functional test)，進而讓我們的UI test的結果可以更加穩定？**
****

### 問題
**首先，要先了解在UI test進行stub和mock會遇到哪些問題。UI test，是測試一連串行為動作是否正確(ex.是否可以順利刊登商品)，是一連串不同API的requset和response串接而成整個動作，所以我們必須可以正確mock所有response，並對每個不同request進行stub且回傳對應正確的mock。**

**1. 如何mock所有API的response?**

**2. 如何stub所有API的request，並回傳正確的mock?**
****

### 方式

**我利用到這兩個套件[SWHttpTrafficRecorder](https://github.com/capitalone/SWHttpTrafficRecorder)，[OHHTTPStubs](https://github.com/AliSoftware/OHHTTPStubs)。**

> **如何mock所有API的response ??**
> 
**在SWHttpTrafficRecorder中有提供方法可以幫助我們紀錄下所有API的response，並提供不同檔案格式儲存這些response，因此接下來重點就是`如何命名儲存下來的檔案名稱`。為何說這是重點，因為接下來在進行stub API request時，我們會比對出正確的檔案名稱，回傳正確mock。**

>命名方式: 記錄此response對應request的`http method`，`host`，`一串md5字串`
>
>(request.HTTPMethod) + (request.URL.host) + (md5)
>
>ex. GET_Google.com_XXXXXXXX.json
>
>>其中md5到底是哪些東西進行md5 encode呢？
>>如果HTTPMethod是GET或DELETE，就是整串的URL進行encode。如果是POST，則是對body進行encode。


> **如何stub所有API的request，並回傳正確的mock ??**
> 
**在上個步驟中，我們已經mock所有response，且根據對應的request資訊來進行檔案的命名，並存檔下來。這個步驟就是根據要stub的request的特徵取得相對md5字串，然後比對檔案中相符md5，回傳正確的mock。**

****

### 程式碼

**1. 如何mock所有API的response?**

```objective-c
    [SWHttpTrafficRecorder sharedRecorder].recordingFormat = SWHTTPTrafficRecordingFormatHTTPMessage;
    [SWHttpTrafficRecorder sharedRecorder].fileNamingBlock = ^NSString *(NSURLRequest *request, NSURLResponse *response, NSString *defaultName) {
        if ([request.URL.host isEqualToString:@"XXXXX"]) {
            return [NSString stringWithFormat:@"%@_%@_%@.response", request.HTTPMethod, request.URL.host, [self getMd5StringFromRequest:request]];
        } else {
            return @"unknown";
        }
    };

    [[SWHttpTrafficRecorder sharedRecorder] startRecording];
```

**2. 根據對應的request資訊來進行md5 encode**

```objective-c
+ (NSString *)md5:(NSString *)original
{
    const char *cStr = [original UTF8String];
    unsigned char digest[CC_MD5_DIGEST_LENGTH];
    CC_MD5( cStr, strlen(cStr), digest );

    NSMutableString *output = [NSMutableString stringWithCapacity:CC_MD5_DIGEST_LENGTH * 2];

    for(int i = 0; i < CC_MD5_DIGEST_LENGTH; i++)
        [output appendFormat:@"%02x", digest[i]];

    return  output;
}

+ (NSString *)getMd5StringFromRequest:(NSURLRequest *)request
{
    NSString *md5String = @"";
    if ([request.HTTPMethod isEqualToString:@"GET"] || [request.HTTPMethod isEqualToString:@"DELETE"]) {
        md5String = [self md5:request.URL.absoluteString];
    } else {
        NSData* body = request.OHHTTPStubs_HTTPBody;
        NSString* bodyString = [[NSString alloc] initWithData:body encoding:NSUTF8StringEncoding];
        md5String =  [self md5:bodyString];;
    }

    return md5String;
}

```

**3. 如何stub所有API的request，並回傳正確的mock?**

```objective-c
+ (void)startMockingResponse
{
    [OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
        if ([self mockResponseFilenameForRequest:request]) {
            return YES;
        } else {
            return NO;
        }
    } withStubResponse:^OHHTTPStubsResponse*(NSURLRequest *request) {
        NSData *data = [[NSData alloc] initWithContentsOfURL:[NSURL fileURLWithPath:[self mockResponseFilenameForRequest:request]]];
        return [OHHTTPStubsResponse responseWithHTTPMessageData:data];
    }];
}

+ (NSString *)mockResponseFilenameForRequest:(NSURLRequest *)request
{
    NSString *md5 = @"";

    if ([request.URL.host isEqualToString:@"XXX"]) {
        md5 = [self getMd5StringFromRequest:request];
    }

    NSString *path = nil;
    NSError *error = nil;
    NSString *bundlePath = [[NSBundle mainBundle] pathForResource:@"MockingResponse" ofType:@"bundle"];
    NSArray *mockFiles = [[NSFileManager defaultManager] contentsOfDirectoryAtPath:bundlePath error:&error];
    if (!error) {
        for (NSString *file in mockFiles) {
            if ([file rangeOfString:md5].location != NSNotFound) {
                path = [bundlePath stringByAppendingPathComponent:file];
                break;
            }
        }
    }

    return path;
}

```
****

### 討論

**目前這樣的設計方式，基本上可以滿足目前大部分的需求，讓UI test可以順利進行stub跟mock。**

**但還是有不足的地方，我目前仍要解決一種狀況，就是在不同時間點下的若是發出相同request，可能會取得不同內容的respone，我必須要區別出這樣的requst。以我目前設計方式，因為requst具有相同特徵值，對應的md5字串也會是相同，即使是respone不同，我所產生的檔案名稱會是一樣，無法區別。**



