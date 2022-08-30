# Actor高亮描边

百度搜索能搜到相关

场景中加入 PostProcessVolume

修改 PostProcessVolume 中的

Post Process Materials

数组添加描边材质

Infinite Extent 勾上

Actor的StaticMesh调用以下两个函数SetRenderCustomDepth(true) 和 SetCustomDepthStencilValue(255)

如果发现不亮，看看Config/DefaultEngine.ini 中的 r.CustomDepth 改成 3

[/Script/Engine.RendererSettings]<br>
r.CustomDepth=3
