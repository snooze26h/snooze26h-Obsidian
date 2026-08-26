# 用 AI 整理了一下我乱成一锅粥的 Chrome 书签，效果出乎意料 

[原帖链接](https://linux.do/t/topic/2063046/1)

**作者：despriber**  
**时间：Apr 26, 2026 10:35 am**  

不知道佬们有没有和我一样的毛病——收藏书签的时候从来不分类，随手一个 Ctrl+D 就完事了。

日积月累下来，书签栏变成了这样：
> Gmail、YouTube、某个不知道什么时候存的github链接、chatGPT、学校教务系统、甚至还有一些不知道什么时候误触保存的垃圾书签……


这就导致每次想找个之前收藏的东西，要么靠搜索，要么靠记忆翻半天。

## 突然想到：这活儿不是正好给 AI 干吗？

Chrome 的书签本质上就是一个 JSON 文件，路径在：

```
# Windows
%LOCALAPPDATA%\Google\Chrome\User Data\Default\Bookmarks

# macOS
~/Library/Application Support/Google/Chrome/Default/Bookmarks

# Linux
~/.config/google-chrome/Default/Bookmarks
```

直接用文本编辑器打开就能看到结构，大概长这样：

```
{
  "roots": {
    "bookmark_bar": {
      "children": [
        { "name": "Gmail", "type": "url", "url": "..." },
        { "name": "某个帖子", "type": "url", "url": "..." },
        ...
      ]
    }
  }
}
```

## 我是怎么做的

1. 先关掉 Chrome
2. 把 `Bookmarks` 文件的内容丢给 AI（我用的 Claude Code，其他 AI 工具也行）
3. 告诉它：**帮我按内容分类，建文件夹整理**
4. AI 会先分析所有书签的名称和 URL，提出分类方案让你确认
5. 确认后它直接生成新的 JSON，替换原文件
6. 重新打开 Chrome，搞定

## 几个小建议

- **操作前一定要备份！** 复制一份 `Bookmarks` 文件改个名就行，万一翻车可以恢复（只不过不太可能
- 如果你书签特别多（几百上千），可以先让 AI 列出分类方案，确认后再执行，避免它自作主张
- 整理完**重启 Chrome** 即可生效，不需要其他操作

## 适用场景

不限于 Chrome，其他基于 Chromium 的浏览器（Edge、Brave、Arc 等）书签格式基本一样。Firefox 的书签是 SQLite 数据库格式（`places.sqlite`），理论上也能让 AI 处理，但操作稍微复杂一点。

---

说实话这个需求看起来很小，但用过之后才觉得真的舒服。毕竟谁的书签不是堆了几年的垃圾场呢 附上claudecode整理完的书签  

![image](https://cdn3.ldstatic.com/original/4X/4/f/e/4fe0448dd5d0d6e16645976cb9a43172e255eae4.png)
