
# Slate基础框架
***

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Slate基础框架](#slate基础框架)
  - [Slate控件分类](#slate控件分类)
  - [Slate渲染流程](#slate渲染流程)
    - [总体流程一瞥](#总体流程一瞥)
    - [图元信息的收集](#图元信息的收集)
  - [Slate是如何布局的](#slate是如何布局的)
    - [Pass1： ComputeDesiredSize](#pass1-computedesiredsize)
    - [Pass2： ArrangeChildren](#pass2-arrangechildren)
    - [EX： 如何拿到一个控件的真实大小？](#ex-如何拿到一个控件的真实大小)
  - [LayerId与UI合批](#layerid与ui合批)
  - [InvalidationBox优化](#invalidationbox优化)

<!-- /code_chunk_output -->



通过阅读源码，希望搞明白以下内容：
1. Slate如何渲染的
2. Slate如何布局的
3. Slate合批机制，如何优化性能

## Slate控件分类
***
Slate控件通过槽位(Slot)来实现嵌套，不同的控件类型拥有不同数量的槽位，总体可以分为三种类型：
Slate控件分为三类:
SPanel： 存在多个Slot，支持嵌套多个SWidget
SCompoundWidget：只拥有一个Slot，支持放一个SWidget
SLeafWidget：不存在Slot，是Widget树中的叶子节点。

## Slate渲染流程
*** 
### 总体流程一瞥
可参考文档[Slate合批机制剖析](https://zhuanlan.zhihu.com/p/346275251)

首先我们自己写一个SWidget控件，用断点去看一下整个Slate绘制的调用栈，这里用插件简单起了个Slate面板，插入了我们自定义的控件：
```cpp
return SNew(SDockTab)
    .TabRole(ETabRole::NomadTab)
    [
        // Put your tab content here!
        SNew(SBox)
        .HAlign(HAlign_Center)
        .VAlign(VAlign_Center)
        [
            SNew(SBoxEx)
            .HAlign(HAlign_Center)
            .VAlign(VAlign_Center)
            [
                SNew(SButton)
                .OnClicked(FOnClicked::CreateLambda([]()
                {
                    UE_LOG(LogTemp, Warning, TEXT("AAAAAAAAAAAA"));
                    return FReply::Handled();
                }))
            ]
        ]
	];
``` 

看下调用栈：
![](./Image/image.png.png)

通过调用栈可以大致了解绘制过程：
1. 在FEngineLoop中，每一次循环都会拉起FSlateApplication做Slate的绘制流程
2. 各个SWidget自上而下，通过重载统一的接口OnPaint收集图元信息
3. 收集完一个SWindow所有的图元信息后，在FSlateApplication中完成合批操作，并发送渲染指令给渲染线程

下面是一个简单的示意图：
![](./Image/2023-01-17-22-54-22.png)

### 图元信息的收集
在控件的`OnPaint`函数中， 需要提供一个参数`FSlateWindowElementList& OutDrawElements`。这个参数中记录的就是每一个控件的图元信息，它会顺着整根Widget树一路传递下去，最后收集完整根树的图元信息。
我们可以看一个叶子节点SButton来了解一下如何构造图元信息的：
```cpp
int32 SButton::OnPaint(const FPaintArgs& Args, const FGeometry& AllottedGeometry, const FSlateRect& MyCullingRect, FSlateWindowElementList& OutDrawElements, int32 LayerId, const FWidgetStyle& InWidgetStyle, bool bParentEnabled) const
{
	......

	if (BrushResource && BrushResource->DrawAs != ESlateBrushDrawType::NoDrawType)
	{
		FSlateDrawElement::MakeBox(
			OutDrawElements,
			LayerId,
			AllottedGeometry.ToPaintGeometry(),
			BrushResource,
			DrawEffects,
			BrushResource->GetTint(InWidgetStyle) * InWidgetStyle.GetColorAndOpacityTint() * BorderBackgroundColor.Get().GetColor(InWidgetStyle)
		);
	}

	return SCompoundWidget::OnPaint(Args, AllottedGeometry, MyCullingRect, OutDrawElements, LayerId, InWidgetStyle, bEnabled);
}
```

可以看到，SButton通过`FSlateDrawElement::MakeBox`构造了自己的图元信息，并放入`OutDrawElements`中。

在所有Widget的图元信息均收集完毕后，FSlateApplication会将他们都放入一个Buffer中，并调用渲染器执行合批渲染操作：
```cpp
Renderer->DrawWindows( DrawWindowArgs.OutDrawBuffer );
```

Slate的实际渲染在SlateRenderer中执行，会根据硬件的不同调用到不同图形API对应的SlateRendeer上，比如SlateRHIRenderer：
```cpp
FSlateRHIRenderer::DrawWindows(FSlateDrawBuffer& WindowDrawBuffer)
{
  DrawWindows_Private(WindowDrawBuffer);
}
```
在这里，会将`FSlateDrawBuffer`中的所有图元信息通过`SlateElementBatcher::AddElements`封装成`FSlateRenderBatch`，在生成为Batch的同时，会根据**图元类型**，生成所需绘制图元的Vertex信息：
```cpp
void FSlateElementBatcher::AddElementsInternal(const FSlateDrawElementArray& DrawElements, const FVector2D& ViewportSize)
{
	for (const FSlateDrawElement& DrawElement : DrawElements)
	{
		// Determine what type of element to add
		switch ( DrawElement.GetElementType() )
		{
		case EElementType::ET_Box:
		{
			SCOPED_NAMED_EVENT_TEXT("Slate::AddBoxElement", FColor::Magenta);
			STAT(ElementStat_Boxes++);
			DrawElement.IsPixelSnapped() ? AddBoxElement<ESlateVertexRounding::Enabled>(DrawElement) : AddBoxElement<ESlateVertexRounding::Disabled>(DrawElement);
		}
			break;
		case EElementType::ET_Border:
		{
			SCOPED_NAMED_EVENT_TEXT("Slate::AddBorderElement", FColor::Magenta);
			STAT(ElementStat_Borders++);
			DrawElement.IsPixelSnapped() ? AddBorderElement<ESlateVertexRounding::Enabled>(DrawElement) : AddBorderElement<ESlateVertexRounding::Disabled>(DrawElement);
		}
			break;
		case EElementType::ET_Text:
		{
			SCOPED_NAMED_EVENT_TEXT("Slate::AddTextElement", FColor::Magenta);
			STAT(ElementStat_Text++);
			DrawElement.IsPixelSnapped() ? AddTextElement<ESlateVertexRounding::Enabled>(DrawElement) : AddTextElement<ESlateVertexRounding::Disabled>(DrawElement);
		}
			break;
		case EElementType::ET_ShapedText:
		{
			SCOPED_NAMED_EVENT_TEXT("Slate::AddShapedTextElement", FColor::Magenta);
			STAT(ElementStat_ShapedText++);
			DrawElement.IsPixelSnapped() ? AddShapedTextElement<ESlateVertexRounding::Enabled>(DrawElement) : AddShapedTextElement<ESlateVertexRounding::Disabled>(DrawElement);
		}
			break;
		case EElementType::ET_Line:
		{
			SCOPED_NAMED_EVENT_TEXT("Slate::AddLineElement", FColor::Magenta);
			STAT(ElementStat_Line++);
			DrawElement.IsPixelSnapped() ? AddLineElement<ESlateVertexRounding::Enabled>(DrawElement) : AddLineElement<ESlateVertexRounding::Disabled>(DrawElement);
		}
			break;
      ......

      		default:
			checkf(0, TEXT("Invalid element type"));
			break;
		}
	}
}
```


## Slate是如何布局的
***
根据官方文档中的介绍[Slate Architecture](https://docs.unrealengine.com/4.26/en-US/ProgrammingAndScripting/Slate/Architecture/)，Slate的布局主要有个Pass：
1. 调用`SWidget::ComputeDesiredSize`计算该控件所需要分配的空间大小
2. 调用`SWidget::ArrangeChildren`来控制子控件的布局

这两个步骤缺一不可，必须拆为两步。因为父控件若想分配子控件的布局，必须首先知道自己的大小才可。

### Pass1： ComputeDesiredSize
为了布局，首先得确认自己的大小。不同类型的widget有不同的计算逻辑，比如SCompoundWidget会先计算子Widget的大小，再根据Style算出自己的大小；SPanel则会将所有子Widget大小考虑在内。

`ComputeDesiredSize`是递归（自下而上）执行的，这很好理解。`SWidget::CachedDesiredSize`会记录最近一次执行`ComputeDesiredSize`的结果。

### Pass2： ArrangeChildren
在计算完自己所占的大小后，就能够对子控件进行布局。这个操作对于不同的Widget来说，都是不同的。

### EX： 如何拿到一个控件的真实大小？
对于动态的控件，直接调用`MyWidget->GetDesiredSize();`或者`MyWidget->GetCachedGeometry()`拿到的大小可能是不准的，因为这样拿到的是该控件上一个tick时计算出来的结果。如果需要拿当前帧下控件的大小，最好先调用`ForceLayoutPrepass(); `强制计算当前控件的真实大小。实际上就是强制做了`ComputeDesiredSize`的操作。

## LayerId与UI合批
***
在`SWidget::OnPaint`方法中，存在一个`&LayerId`的参数。LayerId从小到大递增，代表了渲染的层数，不同LayerId必然会分到不同的DrawCall中去。
关于LayerId的传递过程，知乎文档里有详细介绍[Slate合批机制剖析](https://zhuanlan.zhihu.com/p/346275251)，简单记录一下：
- SWidget所有控件的基类，不改变LayerId。
- SLeafWidget不包含子控件，不改变LayerId。
- SCompoundWidget`包含一个子控件，它会使子控件的LayerId + 1。
- SPanel包含多个子控件，不改变LayerId，所有子控件都继承父控件的LayerId。

**比较特殊的SOverlay：**SOverlay每绘制一个子控件LayerId + 1，当完成了一个子控件的绘制后，会将子控件返回的LayerId和当前LayerId取Max，然后下一个子控件再LayerId + 1。
过度使用Overlay会导致LayerId数骤增，从而导致性能问题。

## InvalidationBox优化
***
参考知乎文章：[UE4 InvalidationBox优化](https://zhuanlan.zhihu.com/p/531146689)
> **大部分UI都是更新不频繁的，可以将它们放在InvalidationBox下面。
在 Invalidation Box 下的所有 Prepass 和 OnPaint 计算结果都会被缓存下来。如果某个 Child Widget 的渲染信息发生变化，就会通知 Invalidation Box 重新计算一次 Prepass 和 OnPaint 更新缓存信息。** 当某个 Widget 的渲染信息变化时，会通知所在的 Invalidation Box 重新缓存 Vertex Buffer。slate控件的UpdateFlags中都默认带有EWidgetUpdateFlags::NeedsTick标记，手动SetCanTick(false)可关闭。