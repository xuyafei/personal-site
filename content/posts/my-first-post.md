baseURL = 'https://example.org/'
languageCode = 'zh-cn'
title = '我的博客'
theme = 'PaperMod'

# PaperMod 主题的基本配置
[params]
defaultTheme = "auto"
ShowReadingTime = true
ShowShareButtons = true
ShowPostNavLinks = true
ShowBreadCrumbs = true
ShowCodeCopyButtons = true
ShowRssButtonInSectionTermList = true
ShowToc = true

[params.homeInfoParams]
Title = "欢迎来到我的博客 👋"
Content = "这里是我的个人博客，我会在这里分享一些想法和经验。"

[params.profileMode]
enabled = false

[menu]
main = [
    {identifier = "categories", name = "分类", url = "/categories/", weight = 10},
    {identifier = "tags", name = "标签", url = "/tags/", weight = 20},
    {identifier = "archives", name = "归档", url = "/archives/", weight = 30},
    {identifier = "search", name = "搜索", url = "/search/", weight = 40},
]

# 启用搜索功能
[outputs]
home = [ "HTML", "RSS", "JSON" ]
