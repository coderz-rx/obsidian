1、如果创建公钥，且添加到自己git仓库后，本地还是无法完成身份验证，可能是本地有多个公钥配置，而机器没有正确识别到对应的公钥文件，此时需要添加公钥文件到ssh agent
- 先创建ssh anget：eval "$(ssh-agent -s)"
- 添加公钥文件命令： ssh-add ~/.ssh/id_rsa_new
- 添加后查看是否可以认证成功命令：ssh -T git@github.com

2、身份验证成功后要将本地文件与远程仓库之间建立连接，以此执行以下操作：
- 添加当前目录下的所有文件：git add --all
- 提交日志：git commit -m “first commit” 
- 连接远程仓库：git remote add origin git@github.com:coderz-rx/obsidian.git
- 推送到远程仓库分支master：git push -u origin master 
3、

