# Sacn-QRCode-BarCode
支持iOS7及以上级版本，支持二维码和条形码。iOS8还支持DM码等，具体可以查看sdk<br>
适配iOS10，info.plist文件添加相机权限Privacy - Camera Usage Description<br>

部分测试结果表明，最好是允许使用相机的情况下再获取设备，否则device为nil，会崩溃。<br>
```
AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
```

AVMetadataObject类，使用原生Api扫描和处理的效率非常高，瞬间完成。<br>

已经封装成CKScanHelper，只有复制这两个文件到项目，5个方法就可以快速实现扫描。不需要使用第三方扫描，第三方文件多，配置麻烦，还有版本限制，32位、64位区分。

百度经验http://jingyan.baidu.com/article/eb9f7b6d7bc5ba869264e863.html

###使用方法
```
//扫描框定义（可不要，全屏扫描）
CGSize windowSize = [UIScreen mainScreen].bounds.size;    
CGSize scanSize = CGSizeMake(windowSize.width*3/4, windowSize.width*3/4);
CGRect scanRect = CGRectMake((windowSize.width-scanSize.width)/2, 30, scanSize.width, scanSize.height);
UIView *scanRectView = [UIView new];
scanRectView.layer.borderColor = [UIColor redColor].CGColor;
scanRectView.layer.borderWidth = 1;
//封装调用方法
[[CKScanHelper manager] showLayer:self.view];
[[CKScanHelper manager] setScanningRect:scanRect scanView:scanRectView];
[[CKScanHelper manager] setScanBlock:^(NSString *scanResult){
     NSLog(@"%@", scanResult);
}];

[[CKScanHelper manager] startRunning];//开始扫描

[[CKScanHelper manager] stopRunning];//结束扫描
```
###1.包含头文件：AVFoundation/AVFoundation.h
###2.引用协议代理： AVCaptureMetadataOutputObjectsDelegate

###3.声明对象
```
AVCaptureSession *_session;             //输入输出的中间桥梁
AVCaptureVideoPreviewLayer *_layer;     //捕捉视频预览层
AVCaptureMetadataOutput *_output;       //捕获元数据输出
AVCaptureDeviceInput *_input;           //采集设备输入
UIView *_superView;                     //图层的父类
```
###4.实例化对象
```
     //初始化链接对象
     _session = [[AVCaptureSession alloc]init];
     //高质量采集率
     [_session setSessionPreset:AVCaptureSessionPresetHigh];
        
     // MARK: 避免模拟器运行崩溃
     if(!TARGET_IPHONE_SIMULATOR) {
          //获取摄像设备
          AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
          //创建输入流
          _input = [AVCaptureDeviceInput deviceInputWithDevice:device error:nil];
          [_session addInput:_input];
            
          //创建输出流
          _output = [[AVCaptureMetadataOutput alloc]init];
          //设置代理 在主线程里刷新
          [_output setMetadataObjectsDelegate:self queue:dispatch_get_main_queue()];
          [_session addOutput:_output];
          //设置扫码支持的编码格式(如下设置条形码和二维码兼容)
            _output.metadataObjectTypes = @[AVMetadataObjectTypeQRCode,
                                            AVMetadataObjectTypeEAN13Code,
                                            AVMetadataObjectTypeEAN8Code,
                                            AVMetadataObjectTypeCode128Code];
            
          // 要在addOutput之后，否则iOS10会崩溃
          _layer = [AVCaptureVideoPreviewLayer layerWithSession:_session];
          _layer.videoGravity = AVLayerVideoGravityResizeAspectFill;
     }
```
###5.实现扫描代理方法 成功输出
```
#pragma mark - AVCaptureMetadataOutputObjects Delegate
- (void)captureOutput:(AVCaptureOutput *)captureOutput didOutputMetadataObjects:(NSArray *)metadataObjects fromConnection:(AVCaptureConnection *)connection{
    if (metadataObjects.count > 0) {
        //[_session stopRunning];
        AVMetadataMachineReadableCodeObject * metadataObject = [metadataObjects objectAtIndex :0];
        if (self.scanBlock) {
            self.scanBlock(metadataObject.stringValue);
        }
        //输出扫描字符串
        NSLog(@"%@",metadataObject.stringValue);
    }
}
```
###6.开始结束扫描
```
#pragma mark 开始捕获
- (void)startRunning {
    // MARK: 避免模拟器运行崩溃
    if(!TARGET_IPHONE_SIMULATOR) {
        [_session startRunning];
    }
    
}
#pragma mark 停止捕获
- (void)stopRunning {
    // MARK: 避免模拟器运行崩溃
    if(!TARGET_IPHONE_SIMULATOR) {
        [_session stopRunning];
    }
}
```
###7.优化扫描区域
CGRectMake（y的起点/屏幕的高，x的起点/屏幕的宽，扫描的区域的高/屏幕的高，扫描的区域的宽/屏幕的宽）
```
- (void)setScanningRect:(CGRect)scanRect scanView:(UIView *)scanView
{
    CGFloat x,y,width,height;
    
    x = scanRect.origin.y / _layer.frame.size.height;
    y = scanRect.origin.x / _layer.frame.size.width;
    width = scanRect.size.height / _layer.frame.size.height;
    height = scanRect.size.width / _layer.frame.size.width;
    
    _output.rectOfInterest = CGRectMake(x, y, width, height);
    
    self.scanView = scanView;
    if (self.scanView) {
        self.scanView.frame = scanRect;
        if (_viewContainer) {
            [_viewContainer addSubview:self.scanView];
        }
    }
}
```
###8.添加显示图层
```
- (void)showLayer:(UIView *)viewContainer
{
    _viewContainer = viewContainer;
    _layer.frame = _viewContainer.layer.frame;
    [_viewContainer.layer insertSublayer:_layer atIndex:0];
}
```
