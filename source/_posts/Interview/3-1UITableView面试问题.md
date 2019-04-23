---
title: UITableView面试问题
keywords: iOS面试
date: 2019-04-22 15:47:40
categories: 
  - 面试
tags:
  - UITableView
comments: true
---

# UI视图

![UI视图](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/Tech/ui视图.png)

# UITableView相关

- `重用机制`
- `数据源同步`

## 重用机制

```objective-c
cell = [tableView dequeueReuseableCellWithIdentifier:@"cell"];
```

![cellReuse](https://raw.githubusercontent.com/HaviLee/Blog-Images/master/cellreuse.png)

**{% label danger@说明： %}**

虚线是当前屏幕显示的内容，A2-A6显示在当前屏幕中，A1划出屏幕被加入到重用池中，A7即将出现在屏幕，会从重用池中根据identifier找到重用的cell。

## 自定义UI控件

- 字母索引条

自定义重用池实现

```objective-c
#import <Foundation/Foundation.h>
#import <UIKit/UIKit.h>
// 实现重用机制的类
@interface ViewReusePool : NSObject

// 从重用池当中取出一个可重用的view
- (UIView *)dequeueReusableView;

// 向重用池当中添加一个视图
- (void)addUsingView:(UIView *)view;

// 重置方法，将当前使用中的视图移动到可重用队列当中
- (void)reset;

@end
```

```objective-c
#import "ViewReusePool.h"

@interface ViewReusePool ()
// 等待使用的队列
@property (nonatomic, strong) NSMutableSet *waitUsedQueue;
// 使用中的队列
@property (nonatomic, strong) NSMutableSet *usingQueue;
@end

@implementation ViewReusePool

- (id)init{
    self = [super init];
    if (self) {
        _waitUsedQueue = [NSMutableSet set];
        _usingQueue = [NSMutableSet set];
    }
    return self;
}

- (UIView *)dequeueReusableView{
    UIView *view = [_waitUsedQueue anyObject];
    if (view == nil) {
        return nil;
    }
    else{
        // 进行队列移动
        [_waitUsedQueue removeObject:view];
        [_usingQueue addObject:view];
        return view;
    }
}

- (void)addUsingView:(UIView *)view
{
    if (view == nil) {
        return;
    }
    
    // 添加视图到使用中的队列
    [_usingQueue addObject:view];
}

- (void)reset{
    UIView *view = nil;
    while ((view = [_usingQueue anyObject])) {
        // 从使用中队列移除
        [_usingQueue removeObject:view];
        // 加入等待使用的队列
        [_waitUsedQueue addObject:view];
    }
}

```

自定义Tableview:

```objective-c
#import <UIKit/UIKit.h>


@protocol IndexedTableViewDataSource <NSObject>

// 获取一个tableview的字母索引条数据的方法
- (NSArray <NSString *> *)indexTitlesForIndexTableView:(UITableView *)tableView;

@end

@interface IndexedTableView : UITableView
@property (nonatomic, weak) id <IndexedTableViewDataSource> indexedDataSource;
@end
```

```objective-c
#import "IndexedTableView.h"
#import "ViewReusePool.h"
@interface IndexedTableView ()
{
    UIView *containerView;
    ViewReusePool *reusePool;
}
@end

@implementation IndexedTableView

- (void)reloadData{
    [super reloadData];
    
    // 懒加载
    if (containerView == nil) {
        containerView = [[UIView alloc] initWithFrame:CGRectZero];
        containerView.backgroundColor = [UIColor whiteColor];
        
        //避免索引条随着table滚动
        [self.superview insertSubview:containerView aboveSubview:self];
    }
    
    if (reusePool == nil) {
        reusePool = [[ViewReusePool alloc] init];
    }
    
    // 标记所有视图为可重用状态
    [reusePool reset];
    
    // reload字母索引条
    [self reloadIndexedBar];
}

- (void)reloadIndexedBar
{
    // 获取字母索引条的显示内容
    NSArray <NSString *> *arrayTitles = nil;
    if ([self.indexedDataSource respondsToSelector:@selector(indexTitlesForIndexTableView:)]) {
        arrayTitles = [self.indexedDataSource indexTitlesForIndexTableView:self];
    }
    
    // 判断字母索引条是否为空
    if (!arrayTitles || arrayTitles.count <= 0) {
        [containerView setHidden:YES];
        return;
    }
    
    NSUInteger count = arrayTitles.count;
    CGFloat buttonWidth = 60;
    CGFloat buttonHeight = self.frame.size.height / count;
    
    for (int i = 0; i < [arrayTitles count]; i++) {
        NSString *title = [arrayTitles objectAtIndex:i];
        
        // 从重用池当中取一个Button出来
        UIButton *button = (UIButton *)[reusePool dequeueReusableView];
        // 如果没有可重用的Button重新创建一个
        if (button == nil) {
            button = [[UIButton alloc] initWithFrame:CGRectZero];
            button.backgroundColor = [UIColor whiteColor];
            
            // 注册button到重用池当中
            [reusePool addUsingView:button];
            NSLog(@"新创建一个Button");
        }
        else{
            NSLog(@"Button 重用了");
        }
        
        // 添加button到父视图控件
        [containerView addSubview:button];
        [button setTitle:title forState:UIControlStateNormal];
        [button setTitleColor:[UIColor blackColor] forState:UIControlStateNormal];
        
        // 设置button的坐标
        [button setFrame:CGRectMake(0, i * buttonHeight, buttonWidth, buttonHeight)];
    }
    
    [containerView setHidden:NO];
    containerView.frame = CGRectMake(self.frame.origin.x + self.frame.size.width - buttonWidth, self.frame.origin.y, buttonWidth, self.frame.size.height);
}


@end
```

