---
description: Git 常用操作快捷指令
---

## 支持的指令关键词

用户说以下任意关键词时，直接执行对应命令，无需再次确认。
项目目录：`c:\Users\lenovo\Desktop\langgragh_skills`

### 拉取 / pull / 同步
```
git pull
```

### 添加 / add / 暂存
```
git add .
```

### 提交 / commit
询问用户提交信息（commit message），然后执行：
```
git commit -m "<用户提供的信息>"
```
如果用户没有提供信息，使用默认：`git commit -m "update"`

### 推送 / push / 上传
```
git push
```

### 一键同步 / 全部 / 一把梭
依次执行：
```
git add .
git commit -m "<询问用户或默认 update>"
git push
```

### 状态 / status / 看看
```
git status
```

### 日志 / log / 历史
```
git log --oneline -10
```

### 撤销 / 回退
```
git restore .
```
> 执行前必须警告用户：这会丢弃所有未提交的修改，需要用户再次确认。
