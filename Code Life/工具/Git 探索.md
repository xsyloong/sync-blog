
## Git 初始化 .gitignore 文件不生效

```bash
# 首先删除git缓存，重新提交并推送
git rm -r --cached .

git add .

git commit -m 'update .gitignore'

git push -u origin/master
```

