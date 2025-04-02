1、如果创建公钥，且添加到自己git仓库后，本地还是无法完成身份验证，可能是本地有多个公钥配置，而机器没有正确识别到对应的公钥文件，此时需要添加公钥文件到ssh agent
- 先创建ssh anget：eval "$(ssh-agent -s)"
- 添加公钥文件命令： ssh-add ~/.ssh/id_rsa_new
- 添加后查看是否可以认证成功命令：ssh -T git@github.com

2、身份验证成功后要将本地文件与远程仓库之间建立连接，以此执行以下操作：
- 添加当前目录下的所有文件：git add --all
- 提交日志：git commit -m “first commit” 
- 连接远程仓库：git remote add origin git@github.com:coderz-rx/obsidian.git
- 推送到远程仓库分支master：git push -u origin master 

比较彻底的配置方法，直接为不同的host配置不同的公钥即可，例如：为gitlab（私库）和github配置不同的公钥，这样在访问不同host时就会读取不同公钥文件

Host gitlab.corp.qunar.com
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa

Host github.com
    HostName ssh.github.com
    User git
    port 443
    IdentityFile ~/.ssh/id_rsa_new
