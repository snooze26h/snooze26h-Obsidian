# [开源]科研图表(曲线图)数据提取工具(导出excle数据) 

[原帖链接](https://linux.do/t/topic/1435502/1)

**作者：噜噜**  
**时间：Jan 12, 2026 7:40 am**  

佬们好~不知不觉已经二级了，闲来无事用CC做了一个提取2D数据曲线图数据为excle的工具，相信各位硕博佬可能会用得到，所以开源供各位佬使用~（这里感谢 [【WONG公益站】弄个主贴，有问题在这里说吧～（首页有教程和常见问题，记得看） - 开发调优 / 开发调优, Lv1 - LINUX DO](https://linux.do/t/topic/1179964/716)）

开源地址： [yyy-OPS/SciDataExtractor: SciDataExtractor 是一款开源的科学图表数据提取工具，专为科研人员设计。基于 FastAPI 和 React 开发，它结合计算机视觉与 AI 技术，将静态图表图片精准转换为可编辑 Excel 数据。支持交互式坐标校准、HSV 颜色提取，并具备 AI 数据清洗与断点修复功能，能有效去除噪点并补全曲线，辅助高效科研分析。](https://github.com/yyy-OPS/SciDataExtractor)

主要功能就是：基于 OpenCV (HSV 颜色空间) 做分割，提取图片中该颜色的曲线数据，然后描点，最后输出excle。加了AI数据清洗/修复/平滑的功能，效果不理想，还不如使用手工绘制。

亮点可能就是支持手工绘制，如果实在不行，自己手动描一下，设置一下平滑度，出来的数据也堪堪能用。我测试下来，50的颜色容差一般可以把大体轮廓绘制出来了，再不济自己手动点几个点。

**这是原图:**

![image](https://cdn3.ldstatic.com/original/4X/3/7/8/378a23213474cf32cfde826c2eeca2cc2fd31535.png)
**识别出来大概这样（颜色容差50直出）:**

![image](https://cdn3.ldstatic.com/original/4X/9/4/4/94415369cb1acf543c7c7f7bb598835c42cc6390.png)
**这是预览的效果：**

![image](https://cdn3.ldstatic.com/original/4X/f/f/9/ff9d9b33b607459bb5588c7a7a62b5daa6701734.png)
**也可以在origin中复现，工具可以识别改颜色的RGB，在origin中直接设置一样的颜色**

![image](https://cdn3.ldstatic.com/original/4X/b/6/c/b6cf77be75b960495808aba8b79cf0f57ebb7f29.png)
欢迎各位佬支持~~
