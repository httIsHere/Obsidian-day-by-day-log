<%*
// 获取当前日期
const today = window.moment().format("YYYY-MM-DD");
const targetFolder = "Diary"; 
const filePath = `${targetFolder}/${today}.md`;

// 检查今天的日记是否已存在
const fileExists = await app.vault.adapter.exists(filePath);

if (fileExists) {
    // 如果存在，直接在当前页面打开
    const file = app.vault.getAbstractFileByPath(filePath);
    app.workspace.getLeaf(false).openFile(file);
    new Notice("今天已经写过日记啦，已为你打开！");
} else {
    // 如果不存在，生成新内容并创建文件
    const content = `---
date: ${today}
weather: 
mood: 不错
tags: 
  - 日记
---

> [!info] 今天，值得被记下来。
> 一切的日常，都可以成为今天的故事。

## 🔄 今日复盘
- 

## 💡 今日新知
- 

## 🎬 书影音/美食
- 
`;
    const newFile = await app.vault.create(filePath, content);
    app.workspace.getLeaf(false).openFile(newFile);
    new Notice("今日日记已创建！");
}
_%>