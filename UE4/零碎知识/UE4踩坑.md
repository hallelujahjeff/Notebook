## UE4踩坑
***

### CreateDefaultSubObject 构造对象
在构造函数中使用CreateDefaultSubobject构造对象时，要注意对象的UPROPERTY要设置为Instanced。 否则当蓝图复制时，两张蓝图的CDO其实在共用一个对象实例，导致报错：
![](./Image/2025-01-10-15-53-44.png)
![](./Image/2025-01-10-15-54-03.png)