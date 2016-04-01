# iOS-UITableView之delegate方法的缓存
最近在写JSPatch，想做一个简单的Demo，利用JSPatch动态修改方法的能力，写了一个简单的Demo。
代码如下：

`ViewController.h`

```Objective-C
#import <UIKit/UIKit.h>

@interface ViewController : UIViewController


@end
```
头文件很简单。我们还是看.m文件吧

```Objective-C

@interface ViewController ()<UITableViewDataSource, UITableViewDelegate>

@property (nonatomic, strong) UITableView *tableView;
@property (nonatomic, strong) NSArray<NSString *> *dataSource;

@property (nonatomic, strong) NSTimer *timer;
@property (nonatomic, strong) UIButton *button;

@end

@implementation ViewController


- (void)loadView
{
    [super loadView];
    self.automaticallyAdjustsScrollViewInsets = NO;
    _timer = [NSTimer scheduledTimerWithTimeInterval:0.5
                                              target:self
                                            selector:@selector(onTimer:)
                                            userInfo:nil
                                             repeats:YES];
    [_timer fire];

    _dataSource = @[@"1", @"2", @"3", @"4", @"5", @"6", @"7", @"8", @"9", @"10",
                    @"11", @"12", @"13", @"14", @"15", @"16",];
    _tableView = [[UITableView alloc] initWithFrame:CGRectMake(0, 20, CGRectGetWidth([UIScreen mainScreen].bounds), 400)
                                              style:UITableViewStylePlain];
    _tableView.delegate = self;
    _tableView.dataSource = self;
    [self.view addSubview:_tableView];

    _button = [[UIButton alloc] initWithFrame:CGRectMake(0, 20+400, 200, 80)];
    _button.backgroundColor = [UIColor blackColor];
    [_button addTarget:self
                action:@selector(onButtonClicked:)
      forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:_button];
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    return _dataSource.count;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    NSString *const identifier = @"identifier";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:identifier];
    if (!cell) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:identifier];
    }
    cell.textLabel.text = _dataSource[indexPath.row];
    return cell;
}

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    return 32;
}

//- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
//{
//    [tableView deselectRowAtIndexPath:indexPath animated:YES];
//    UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"来自JSPatch的信息"
//                                                    message:[NSString stringWithFormat:@"%ld行被点击", (long)indexPath.row]
//                                                   delegate:nil
//                                          cancelButtonTitle:@"好吧"
//                                          otherButtonTitles:nil];
//    [alert show];
//}

- (void)onTimer:(id)sender
{
    NSLog(@"%@", [self respondsToSelector:@selector(onButtonClicked:)] ? @"方法存在" : @"方法不存在");
}

//- (void)onButtonClicked_fake
//{
//    NSLog(@"fake clicked! Will do nothing!");
//}
//
//- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
//{
//    NSMethodSignature *sig = [super methodSignatureForSelector:aSelector];
//    if (!sig) {
//        if ([NSStringFromSelector(aSelector) isEqualToString:NSStringFromSelector(@selector(onButtonClicked:))]) {
//            sig = [self methodSignatureForSelector:@selector(onButtonClicked_fake)];
//        }
//    }
//
//    return sig;
//}
//
//- (void)forwardInvocation:(NSInvocation *)anInvocation
//{
////    NSLog(@"%@", anInvocation);
////    if ([NSStringFromSelector(anInvocation.selector) isEqualToString:NSStringFromSelector(@selector(onButtonClicked:))]) {
////        anInvocation.selector = @selector(onButtonClicked_fake:);
////        [anInvocation invoke];
////    }
//}
//
//- (BOOL)respondsToSelector:(SEL)aSelector
//{
//    NSLog(@"%@", NSStringFromSelector(aSelector));
//    BOOL suc = [NSStringFromSelector(aSelector) isEqualToString:NSStringFromSelector(@selector(tableView:didSelectRowAtIndexPath:))];
//    return suc ?: [super respondsToSelector:aSelector];
//}
@end
```

我的想法是在运行的时候，通过JSPatch生成`- (void)tableView: didSelectRowAtIndexPath:`方法，在点击UITableViewCell的时候，能响应，弹出对话框。然而在生成了tableview后，再执行JSPatch把`- (void)tableView: didSelectRowAtIndexPath:`添加进去，点击cell的时候居然不响应。
为此，我写了一个button来测试。从上面的代码可以看出，我用button关联一个不存在的方法selector(onButtonClicked:)——这个方法我在JSPatch里实现，在运行的时候动态加载。最后，我点击button后成功响应了。我就在想是不是UITableView本身的问题。

> 请注意，先生成了Table和Button并成功显示在界面上后延迟10秒加载JSPatch。

> JS文件，请参考：[onTableCellClicked](./res/onTableCellClicked.js)

于是，