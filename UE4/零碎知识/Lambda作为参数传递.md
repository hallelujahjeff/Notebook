[UE4中Lambda的一些用法](https://blog.csdn.net/xoyojank/article/details/52859518)

可以使用`TFunctionRef`来定义一个函数引用，在传参时构造一个Lambda函数即可。

举例:
```cpp
/** Iterates over all bodies below and executes Func. Returns number of bodies found */
int32 ForEachBodyBelow(FName BoneName, bool bIncludeSelf, bool bSkipCustomType, TFunctionRef<void(FBodyInstance*)> Func);

void USkeletalMeshComponent::SetAllBodiesBelowSimulatePhysics( const FName& InBoneName, bool bNewSimulate, bool bIncludeSelf )
{
	int32 NumBodiesFound = ForEachBodyBelow(InBoneName, bIncludeSelf, /*bSkipCustomPhysicsType=*/ false, [bNewSimulate](FBodyInstance* BI)
	{
		
		BI->SetInstanceSimulatePhysics(bNewSimulate);
	});
...
}
```