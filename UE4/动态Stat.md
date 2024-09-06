```cpp
// 头文件:
#include "Stats/Stats2.h"
#include "Stats/Stats.h"

// 定义:
DECLARE_STATS_GROUP(TEXT("BasePlayerControllerAAAAA"), STATGROUP_BasePlayerControllerAAAAA, STATCAT_Advanced);

// 使用
#if    STATS
    const FString ___StatName = GetName()+"this Test Stat";
    const TStatId StatId = FDynamicStats::CreateStatId<FStatGroup_STATGROUP_BasePlayerControllerAAAAA>(___StatName);
    FScopeCycleCounter CycleCounter(StatId);
#endif // STATS
```