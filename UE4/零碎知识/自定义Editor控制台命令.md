1. 定义一个`TUniquePtr`指针，用于存储对应的命令
```cpp
TUniquePtr<FAutoConsoleCommand> ConsoleCommand_BatchMinLOD;
```

2. 注册命令:
```cpp
// 强制LOD设置工具，将场景中名称结尾为__p的StaticMeshActor的LOD设定为指定等级
ConsoleCommand_BatchMinLOD = MakeUnique<FAutoConsoleCommand>(TEXT("Tools.ForceLOD"), TEXT("Force LOD level to StaticMesh Actors which name end with \"__p\""), FConsoleCommandWithArgsDelegate::CreateLambda(
    [this](const TArray<FString>& Args)
    {
        UE_LOG(LogTemp, Display, TEXT("Tianyi@ Processing ForceSetMeshLOD Function..."));
        if (Args.IsValidIndex(0))
        {
            auto Arg = FCString::Atoi(*Args[0]);
            UEditorCommonFunctionLibrary::ForceSetMeshMinLOD(Arg);
        }
        else
        {
            UE_LOG(LogTemp, Warning, TEXT("Tianyi@ Args is invalid."))
        }
    }));
```