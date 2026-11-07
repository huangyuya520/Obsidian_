==**git init       初始化本地git仓库（创建新仓库）**==
**git clone git/ssh://git@192.168.53.168/VT.git      clone远程仓库**
**git status              查看当前版本状态（是否修改）**
rm -rf .git       删除或隐藏Git 仓库文件夹
git log            显示提交日志
git log -stat    查看每次提交修改了哪些文件
git reflog       找到之前的 commit ID
git reset --hard `<commit_id>`     恢复特定分支的版本  
git diff HEAD `<commit-id>`      比较当前版本和想查看版本之间的变化
git branch          显示本地分支
git branch -a             显示所有分支
git branch -r             显示所有原创分支
git branch --merged        显示所有已合并到当前分支的分支
git branch --no-merged          显示所有未合并到当前分支的分支
git reset --hard HEAD^    回滚到上一个版本，彻底回退（丢弃上个版本之后的所有修改和代码）
git revert HEAD       安全回退：通过 git revert（适用于已经推送到远程、多人协作的分支）


git branch -M `<xxx>`     将当前主分支重命名为 xxx
git remote add origin `<Your-Repo-URL>`    关联远程 GitHub 仓库


git add <文件名>    添加指定文件到修改区
git add .       添加当前目录下的所有文件到修改区
git commit -m "你的提交说明（例如：修复了登录按钮样式）"    提交到本地仓库（并添加提交说明）
git push origin <分支名>   将本地分支推送到远程对应分支



git rm --cached `<文件名>`        
git commit -m "<停止追踪某文件>"    从 Git 追踪中移除文件，但保留在本地














