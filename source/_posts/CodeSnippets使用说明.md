---
title: CodeSnippets使用说明
tags: 实用工具
categories: 效率工具
---

###  CodeSnippets

### 使用方式：

* 第一种 前往文件夹：
前往文件夹：
```
~/Library/Developer/Xcode/UserData/CodeSnippets
```
然后复制粘贴即可
* 第二种 执行脚本即可
进入到当前文件根目录。然后执行脚本即可。
```
sh DMCodeSnippets.sh
```
如果没有权限,在终端添加`sudo`,输入电脑密码，执行脚本即可。
```sudo sh  DMCodeSnippets.sh```

#### 说明：
|快捷方式	 | 功能	 |实现|
| ------ | ------ |------ |
|DMArray|不可变数组的声明|@property (nonatomic, copy) NSArray *<#array#>;|
| DMBOOL | 基本数据类型BOOL |@property (nonatomic, assign) BOOL  <#bool#>; |
| DMDictionary | 不可变字典属性声明 |@property (nonatomic, copy) NSDictionary *<#dictionary#>; |
| DMMutableDictInterface | 可变字典属性声明 |@property (nonatomic, strong) NSMutableDictionary *<#mutableDictionary#>; |
| DMMutableDictLazyLoading | 可变字典懒加载 | 实现见下面 |
| DMInteger| NSInteger基本数据类型 |@property (nonatomic, assign) NSInteger  <#integer#>;|
| DMString | 字符串声明 |@property (nonatomic, copy) NSString *<#string#>;|
| DMMutableArrayInterface | 可变数组声明 |@property (nonatomic, strong) NSMutableArray *<#mutableArray#>;|
| DMMutableArrayLazyLoading | 可变数组懒加载 |实现见下面 |
| DMLabelInterface | UILabel声明 |@property (nonatomic, strong) UILabel *<#contentLabel#>;|
| DMLabelLazyLoading | UILabel懒加载 |实现见下面 |
| DMButtonInterface |UIButton声明 |@property (nonatomic, strong) UIButton *<#customBtn#>; |
| DMButtonLazyLoading |UIButton懒加载 |实现见下面 |
| DMImageViewInterface | UIImageView声明 |@property (nonatomic, strong) UIImageView *<#iconImageView#>; |
| DMImageViewLazyLoading | UIImageView懒加载 | 实现见下面 |
| DMViewInterface | UIView声明 |        @property (nonatomic, strong) UIView *<#contentView#>;|
| DMViewLazyLoading | UIView懒加载 |        实现见下面|
| DMPragmaMark | pragmaMark |#pragma mark   - <#infoMessage#> -  |
| DMWarning |warning，包括作者，以及warning描述 |#warning   <#author#>   <#messageDesc#>|
| DMDispatchOnce | 单例中dispatch_once写法 |实现见下面|
| DMGCDAsyMain | 异步回到主线程 |实现见下面 |
| DMWeakSelf | weak修饰写法 |    __weak typeof(self)weakSelf = self;|
| DMStrongSelf | strong修饰写法 |        __strong typeof(weakSelf)strongSelf = weakSelf;|
| DMLocalBlock | localBlock |实现见下面 |
| DMInitMethod | 初始化方法 | 实现见下面  |
|DMLifecycle | vc生命周期 |实现见下面 |
|DMTableViewInterface | UITabeleView声明 |@property (nonatomic, strong) UITableView *<#tableView#>; |
|DMTableViewLazyLoading | UITableView懒加载 |实现见下面 |
|DMCollectionViewInterface | UICollectionView声明 |@property (nonatomic, strong) UICollectionView *<#collectionView#>; |
|DMCollectionViewLazyLoading | UICollectionView懒加载|实现见下面 |



#### DMButtonLazyLoading 
实现：UIButton实现
```
- (UIButton *)<#customBtn#> {
    if (!<#_customBtn#>) {
        <#_customBtn#> = [UIButton buttonWithType:UIButtonTypeCustom];
        [<#_customBtn#> setTitle:<#(nullable NSString *)#> forState:UIControlStateNormal];
        [<#_customBtn#> setTitleColor:<#(nullable UIColor *)#> forState:UIControlStateNormal];
        [<#_customBtn#> setBackgroundColor:<#(UI_APPEARANCE_SELECTOR UIColor *)#>];
        [<#_customBtn#> addTarget:<#(nullable id)#> action:<#(nonnull SEL)#> forControlEvents:UIControlEventTouchUpInside];
    }
    return <#_customBtn#>;
}
```
#### DMDispatchOnce
实现：单例中dispatch_once写法
```
  + (instancetype)sharedInstance {
      static dispatch_once_t onceToken;
      dispatch_once(&onceToken, ^{
          <#code#>
      });
      return <#expression#>;
  }
``` 
#### DMGCDAsyMain
实现：异步回到主线程
```
dispatch_async(dispatch_get_main_queue(), ^{
       <#code#>
   });
```
#### DMImageViewLazyLoading
实现：UIImageView懒加载
```
- (UIImageView *)<#iconImageView#> {
    if (!<#_iconImageView#>) {
        <#_iconImageView#> =[[UIImageView alloc]init];
        
    }
    return <#_iconImageView#>;
}
```

#### DMInitMethod
实现：初始化方法
```
- (instancetype)init {
    
    if (self = [super init]) {
        
    }
    return self;
}
```
#### DMLabelLazyLoading
实现：UILabel懒加载
```
- (UILabel *)<#contentLabel#> {
    if (!<#_contentLabel#>) {
        <#_contentLabel#> = [[UILabel alloc]initWithFrame:CGRectMake(<#CGFloat x#>, <#CGFloat y#>, <#CGFloat width#>, <#CGFloat height#>)];
        <#_contentLabel#>.backgroundColor = [UIColor clearColor];
    }
    return <#_contentLabel#>;
}
```

#### DMLifecycle
实现：vc生命周期
```

- (void)pageInit {
    [super pageInit];
    
}

- (void)pageWillForwardToMe {
    [super pageWillForwardToMe];
    
}

- (void)pageDidForwardToMe {
    [super pageDidForwardToMe];
    
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
}

- (void)pageWillBeShown {
    [super pageWillBeShown];
    
}

- (void)pageDidShown {
    [super pageDidShown];
    
}

- (void)pageWillForwardFromMe {
    [super pageWillForwardFromMe];
    
}

- (void)pageDidForwardFromMe {
    [super pageDidForwardFromMe];
    
}

- (void)pageWillBackwardFromMe {
    [super pageWillBackwardFromMe];
    
}

- (void)pageDidBackwardFromMe {
    [super pageDidBackwardFromMe];
    
}

- (void)pageWillBackwardToMe {
    [super pageWillBackwardToMe];
    
}

- (void)pageDidBackwardToMe {
    [super pageDidBackwardToMe];
    
}

- (void)pageWillBeHidden {
    [super pageWillBeHidden];
    
}

- (void)pageDidHidden {
    [super pageDidHidden];
    
}

- (void)pageDestroy {
    [super pageDestroy];
    
}

- (void)pageReload {
    [super pageReload];
    
}
```

#### DMLocalBlock
实现：localBlock
```
<#returnType#>(^<#blocKname#>)(<#parameterTypes#>) = ^<#returnType#> (<#parameters#>) {
    <#statements#>
    };
```
#### DMMutableArrayLazyLoading
实现：可变数组懒加载
```
- (NSMutableArray *)<#mutableArray#> {
    if (!<#_mutableArray#>) {
        <#_mutableArray#> = [NSMutableArray array];
    }
    return <#_mutableArray#>;
}
```

#### DMMutableDictLazyLoading
实现：可变字典懒加载
```
- (NSMutableDictionary *)<#mutableDictionary#> {
    if (!<#_mutableDictionary#>) {
        <#_mutableDictionary#> = [NSMutableDictionary dictionary];
    }
    return <#_mutableDictionary#>;
}
```
#### DMViewLazyLoading
实现：UIView懒加载
```
- (UIView *)<#contentView#> {
    if (!<#_contentView#>) {
        <#_contentView#> = [[UIView alloc]initWithFrame:CGRectMake(<#CGFloat x#>, <#CGFloat y#>, <#CGFloat width#>, <#CGFloat height#>)];
        <#_contentView#>.backgroundColor = [UIColor clearColor];
    }
    return <#_contentView#>;
}
```

# DMTableViewLazyLoading
实现：UITableView懒加载
```
- (UITableView *)<#tableView#> {
    if (!<#_tableView#>) {
        <#_tableView#> = [[UITableView alloc]initWithFrame:<#(CGRect)#> style:UITableViewStylePlain];
        <#_tableView#>.delegate = self;
        <#_tableView#>.dataSource = self;
        [self.view addSubview:<#_tableView#>];
        [<#_tableView#> mas_makeConstraints:^(MASConstraintMaker *make) {
            make.edges.insets(UIEdgeInsetsMake(<#CGFloat top#>, <#CGFloat left#>, <#CGFloat bottom#>, <#CGFloat right#>));
        }];
    }
    return <#_tableView#>;
}
```
# DMCollectionViewLazyLoading
实现：UICollectionView懒加载
```
- (UICollectionView *)<#collectionView#> {
    if (!<#_collectionView#>) {
        <#_collectionView#> = [[UICollectionView alloc]initWithFrame:<#(CGRect)#> collectionViewLayout:<#(nonnull UICollectionViewLayout *)#>];
        <#_collectionView#>.delegate = self;
        <#_collectionView#>.dataSource = self;
        [self.view addSubview:<#_collectionView#>];
        [<#_collectionView#> mas_makeConstraints:^(MASConstraintMaker *make) {
            make.edges.insets(UIEdgeInsetsMake(<#CGFloat top#>, <#CGFloat left#>, <#CGFloat bottom#>, <#CGFloat right#>));
        }];
    }
    return <#_collectionView#>;
}
```

