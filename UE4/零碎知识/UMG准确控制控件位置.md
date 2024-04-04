1. 从控件的FGeometry中可以拿到该控件的绝对坐标
```lua
    local AbsPos = USlateBlueprintLibrary.LocalToAbsolute(ItemWidget:GetCachedGeometry(), FVector2D(0, 0))
```
2. 可以将绝对坐标转换为任意父节点下的相对坐标：
```lua
local TipLocalPos = USlateBlueprintLibrary.AbsoluteToLocal(self.Content.CanvasHandle:GetCachedGeometry(), AbsPos)
```
3. 拿到相对坐标后，就可以通过这个坐标设置控件的位置了