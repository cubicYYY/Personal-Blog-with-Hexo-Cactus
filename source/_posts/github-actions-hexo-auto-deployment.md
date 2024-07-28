---
title: 如何搭建 Hexo 博客并使用 Github Actions 自动部署到自己服务器
tags:
  - Tutorial
  - GitHub
  - Hexo
  - CICD
categories:
  - Tutorials
permalink: github-actions-hexo-auto-deployment/
date: 2024-07-28 21:04:54
---


## 前言

> 注：本博客目前（2024-07-28）采用的部署方式和本文描述的完全一致。
> 你可以在 [https://github.com/cubicYYY/Personal-Blog-with-Hexo-Cactus](https://github.com/cubicYYY/Personal-Blog-with-Hexo-Cactus) 找到博客源码，其中包含了 GitHub Actions 工作流配置文件。

GitHub Actions 是个好东西，可以帮助我们快速实现 CICD 的需求。
配置好后，可以在某些事件（例如 `git push` 会触发对应分支的 `push` 事件）发生时，自动在 GitHub 为你分配的 Runner 上执行命令。

其实 GitHub Actions 的服务器也许比你想象的性能还强大：

- 2-core CPU
- 7 GB of RAM memory
- 14 GB of SSD disk space

……更不用说还可以自建 runner, 无敌了。不过我们的重点不是 GitHub Actions 的配置细节，这里不展开讲。

关于利用 GitHub Actions 实现 `git push` 后自动部署网上已有很多教程，但大部分都是介绍如何部署到 GitHub Pages 上。
因此本文主要补充介绍**如何把 Hexo 博客部署到自己的服务器/VPS上**。

### 流程/架构设计

博客的源码（包含文章原始 MarkDown 文件等）储存在 GitHub 仓库中。

不使用 GitHub Actions 时，我们在本地 `hexo g` 生成静态文件后，借助 `hexo-deployer-git` 插件执行 `hexo d` 将 `public/` 中的静态文件推送至服务器从而完成部署；而源码则需要手动 push 到 GitHub 上进行版本控制。

而 GitHub Actions 可以把这两步合二为一：**源码 push 到 GitHub 上后，触发 action 自动部署到服务器上**。换言之，本地的部署过程现在由 GitHub Actions 负责进行。

接下来让我们实现它。

## 本地 Hexo 环境准备

你可能已经完成基本的博客环境设置，那样的话**直接跳过本部分即可**。
如果还没有完成，简单来说是以下步骤：

1. 准备好 Node.js 环境以及 Git 工具。在此不赘述。推荐使用 [nvm](https://github.com/nvm-sh/nvm/blob/master/README.md) 完成 Node 的版本管理。
2. 准备 Hexo 的命令行工具：`npm install -g hexo-cli`
3. 新建一个 Hexo 项目，起名为 myblog: `hexo init myblog`
4. 进入项目目录并安装依赖：`cd myblog && npm install`
5. 安装 Hexo 本地服务器：`npm install hexo-server`

详细步骤也可以参考这篇文章的前半部分：[使用 Hexo 搭建个人博客并部署到云服务器- cheyaoyao](https://www.cnblogs.com/cheyaoyao/p/17836522.html).
（要是链接挂了就去 Web Archive 上找找。）

这里我自定义了一下 `.gitignore` 文件，忽略掉不该上传的文件:

```plaintext
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy_git/
.deploy*/
_multiconfig.yml
.vscode/
```

然后你就可以把这个博客项目传到 GitHub 上。

至此，已完成最基本的博客搭建。你可以尝试如下命令，启动一个本地的服务器验证一下：`hexo s`.
![本地环境搭建完毕](build-success.png)

然后你应当创建一个 GitHub 仓库并把这个博客的项目代码提交上去（注意我们已经在 `.gitignore` 文件中排除了用于生成静态文件的文件夹 `public/`）。我们稍后的自动化部署就针对这个仓库进行。

## 服务端配置

### 配置 Nginx

由于 Hexo 是静态博客，我们只需要使用 Nginx 把网站根目录指向 Hexo 静态文件部署到的目录，从而把用户导向对应静态文件即可。
Nginx 安装过程在此不再赘述。

这里，我们选择将网站根目录设定为 `/var/www/html/`:

```bash
# 创建文件夹用于存放博客静态文件
sudo mkdir -p /var/www/html/

# 配置 Nginx
cd /etc/nginx
mkdir blog_vhost
cd blog_vhost
vim blog.conf # 编辑博客配置文件
```

```conf
# blog.conf
server{
    listen 80; # 也可以加一句 default_server, 不过记得全局只能有一个 default_server
    listen [::]:80;
    root /var/www/html/; # 上文提到的根目录
    # error_page 404 /404.html;

    index index.html index.htm index.nginx-debian.html; # 默认提供的文件

    server_name blog.your_domain.com; # 改成你的博客域名，没有就写 IP
    location /{ 
        try_files $uri $uri/ =404; # 如果既不是文件也不是文件夹，返回 404 Not Found
    }

    # location = /404.html {
    #   internal;
    # }
}
```

就像这样。
你也可以为其他子域名创建更多`.conf`文件，其中每个文件只有 `server_name` 不同，这样你就可以从不同子域名访问你的博客：`my_blog_server.com`, `www.my_blog_server.com`, `blog.my_blog_server.com`, etc.
不过我直接选择在 DNS 服务器上配置 CNAME.

接下来，我们需要告诉 Nginx 把这个 `blog_vhost` 文件夹的配置文件 include 进来：`vim /etc/nginx/nginx.conf`.
找到 http 块：

```conf
http {
    # ...
    include /etc/nginx/blog_vhost/*.conf; # 加上这一行
    # ...
}
```

![修改 Nginx 配置文件](nginx-edited.png)

确保配置文件语法正确：`nginx -t`.
![Nginx 检查配置文件语法](nginx-checked.png)

然后 Nginx 该启动启动，该重载重载：

```bash
nginx -s reload
# or
nginx
```

### 添加用于博客部署的用户

我们接下来新建一个用户 `git`, 用于接收 Hexo 部署时上传的静态文件。

```bash
sudo adduser git
sudo chown git /var/www/html # 把网站根目录所有者设为 git 用户，根目录和上面保持一致
```

然后你可以通过 `visudo` 等命令，添加一行 `git ALL=(ALL) ALL` 从而允许 `git` 用户的 `sudo` 操作。
![允许 sudo](git-sudo.png)

### 为 `git` 用户设置 SSH

既然这个用户要接收静态文件，自然需要配置 SSH 进行身份认证.

```bash
su git
mkdir ~/.ssh
cd ~/.ssh
```

如果你要生成新的密钥对用于身份验证：

```bash
ssh-keygen # 生成新密钥对，一路回车
touch authorized_keys
cat id_*.pub >> authorized_keys # 允许这个密钥访问服务器，注意这里是追加到末尾 '>>'
```

然后我们就可以把私钥下载（或者直接复制）到本地 SSH 文件夹中。文件上传下载可用 `rsync` 或者 `scp` 等工具完成。
当然，别像这样把密钥贴出来泄露了：

![放心，这个私钥从来不用](get-private-key.png)

你可以在本地为 SSH 配置一个类似这样的文件方便访问：

```config
# %USERPROFILE%\.ssh\config

Host blog_git
	hostname my_blog_server.com # 改成你的服务器地址/IP
	user git
	identityfile C:\Users\username\.ssh\id_ed25519_git # 改成刚刚下载的私钥文件位置
```

然后 `ssh blog_git` 即可访问这个账户.
这个私钥也会在 GitHub Actions 时用到：由于 GitHub Actions 无法进行用户交互输入密码，所以只能采取密钥认证（也更安全）。

当然，**如果复用已有的密钥，则无需像这样新建密钥对**。只需要把公钥上传然后追加到服务端的 `authorized_keys` 即可。

### 创建用于接收文件的 git 仓库

由于 `hexo-deployer-git` 插件会把博客在本地生成的静态文件部署到一个远程仓库，我们现在在服务端，需要建立一个这样的仓库用于接收：

```bash
su git
cd ~
git init --bare blog.git # 裸仓库
```

接下来我们为这个仓库创建一个钩子(hook): 当静态文件被推送到这个仓库后，这个 hook 会自动触发，把静态文件复制到网站根目录中进行部署：

```bash
vim ~/blog.git/hooks/post-receive
```

然后填入：

```bash
# post-receive
git --work-tree=/var/www/html --git-dir=/home/git/blog.git checkout -f
```

意思就是 `/home/git/blog.git` 仓库接收到静态文件后把这些文件检出到 `/var/www/html`。
所以这里 `--work-tree` 要和之前设定的网站根目录一致。

**记得给脚本加上可执行权限**：`chmod +x ~/blog.git/hooks/post-receive` .
当然也要确保 `git` 用户有权限写入网站根目录。我们上面通过 `chown` 已经解决了。

至此，我们已经完成了服务端的设置。接下来我们在客户端进行配置与测试。

## 本地客户端配置

### 安装并配置部署插件

首先，我们当然得先把部署插件装上：`npm install hexo-deployer-git --save` .
然后在 `_config.yml` 中配置如下：

```yml
# ...

# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: 'git'
  repository:
    my_repo_name: git@my_blog_server.com:/home/git/blog.git,master # 替换成你自己的服务器地址、分支名（尤其注意是 master 还是 main）等

# ...
```

### 手动部署测试

然后执行 `hexo clean` 清除本地缓存、执行 `hexo g -d` 完成静态文件生成并部署：这也可以顺带验证一下服务端配置是否正常。
之后如果有关于配置文件或是页面布局的变更，部署前都尽量先清除缓存再生成静态文件。

如果不出意外，你的博客已经完全搭建完毕！可以访问你的博客看看是否已经正常工作了。

不过我们一开始的目的是要实现 CICD 自动部署，所以先别急着 `git commit`! 现在让我们把服务器源地址藏一下。

## 配置 GitHub Actions

### 创建 GitHub Actions Workflow 脚本

修改 `_config.yml`:

```yml
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: 'git'
  repository:
    my_repo_name: git@${SSH_GIT_HOST}:${GIT_FOLDER},master # 现在不把远程地址写死，而是稍后由 GitHub Actions 工作流脚本填入
```

> 遗憾的是这个插件不会主动把 `${...}` 替换成环境变量，我们稍后会在 GitHub Actions 的脚本中用 `sed` 执行文本替换。
> 虽然这个做法有点 dirty, 但……it just works.

**接下来是重头戏：**设置 GitHub Actions 的 workflow 文件。

在博客项目的文件夹下建立`.github/workflows/deploy.yml`:

```bash
cd ~/myblog # (your blog directory)
mkdir -p .github/workflows
touch .github/workflows/deploy.yml
vim deploy.yml
```

然后填入如下内容：

```yml
name: Deploy Hexo Site

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    name: Build and deploy Hexo Site

    steps:
    - name: Checkout source code # 检出仓库代码，从而在源码目录下工作
      uses: actions/checkout@v2

    - name: Setup Node.js # 设置 Node 环境
      uses: actions/setup-node@v2
      with:
        node-version: '20'

    - name: Install Hexo CLI Tools
      run: npm install -g hexo-cli
    
    - name: Cache node modules # 对 NPM 使用缓存
      uses: actions/cache@v2
      id: cache
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-

    - name: SSH Key Setups # 重点：从 GitHub secrets 中获取 SSH 私钥用于配置 SSH
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa

    - name: SSH Fingerprint Registration # 重点：预先配置好服务器指纹避免 SSH 交互询问
      run: |
        touch ~/.ssh/known_hosts
        chmod 600 ~/.ssh/known_hosts
        echo "${{ secrets.SSH_FINGERPRINT }}" >> ~/.ssh/known_hosts

    - name: Install dependencies # 安装依赖，这里使用 `npm ci` 而非 `npm i` 更符合 CI/CD 场景，会使用 package-lock.json
      run: npm ci
    
    - name: Generate static files # 生成静态文件
      run: hexo generate

    - name: Set Bot Info for Git
      run: |
        git config --global user.email "cicd@example.com"
        git config --global user.name "CICD Bot"

    - name: Set Deploy Target # 重点：在 `_config.yml` 中注入服务器地址以及对应的文件夹地址
      run: |
        sed -i "s#\${SSH_GIT_HOST}#$SSH_GIT_HOST#g" _config.yml
        sed -i "s#\${GIT_FOLDER}#$GIT_FOLDER#g" _config.yml
      env:
        SSH_GIT_HOST: ${{ secrets.SSH_GIT_HOST }}
        GIT_FOLDER: ${{ secrets.GIT_FOLDER }}

    - name: Deploy to Server # 使用插件部署，插件会调用 SSH
      run: hexo deploy
```

### 为博客的 GitHub 项目仓库添加 Secrets 变量

这里我们可以看到，脚本需要 `SSH_GIT_HOST` `GIT_FOLDER` 确定远程仓库的地址，以及 `SSH_PRIVATE_KEY` `SSH_FINGERPRINT` 用于 SSH 用户与服务器指纹的校验。
这些当然不能公开，因此我们要添加的是 **secrets** 而不是普通的 **variables**.
如果你愿意，当然也可以自己改改脚本，比如把用户名也放到 secrets 里。

总之，我们需要在 GitHub 上为这个 repo 添加这些 secrets 变量。
路径是：项目仓库首页 - `Settings` - `Secrets` - `Add a new secret`:
![添加 Repo 秘密](add-secrets.png)

- `SSH_GIT_HOST`: 就是你的博客部署服务器地址。例如 `my_blog_server.com` .
- `GIT_FOLDER`: 就是刚刚设置的裸仓库文件位置。例如 `/home/git/blog.git` .
- `SSH_PRIVATE_KEY`: 就是 SSH 私钥，我们在之前已经下载备用，现在就用上了。它应当以 `-----BEGIN OPENSSH PRIVATE KEY-----` 开头，以 `-----END OPENSSH PRIVATE KEY-----` 结尾。
- `SSH_FINGERPRINT`: 要获取它，可以用 `ssh-keyscan my_blog_server.com` 获取。你可以只选择那行匹配你的密钥算法的，例如密钥使用 ED25519 算法，就可以只选中 `my_blog_server.com ssh-ed25519 AAAAC3N...` 那行作为 secret 变量。

`SSH_FINGERPRINT` 用于让 SSH 确信对方确实就是想要联系的服务器。指纹用于校验服务器身份的。可以防止中间人攻击，当然最主要还是为了避免 SSH 要求用户交互确认指纹。

> 你也可以不把指纹作为 secret 变量，而是直接在脚本里 `ssh-keyscan "${SSH_GIT_HOST}" >> ~/.ssh/known_hosts`. 我认为这样不够安全，尽管我自己从未遇到过中间人攻击。

> 作为练习，你可以试着调用已有的 action（例如[install-ssh-key](https://github.com/marketplace/actions/install-ssh-key)）完成 SSH 相关配置并配置 SSH config 文件，顺带给予部署服务器一个固定的别名（例如`deploy_server`）。不过我懒得改了，你可以询问 ChatGPT 帮你完成这些——如果你愿意的话。

### 完结撒花

然后在本地按 Git 的一般流程进行 `git add` `git commit` `git push`，将提交改动到 GitHub 上。

此时我们就可以看到 GitHub Actions 运行起来，不出意外的话等待几十秒即可运行成功！
此时你可以在仓库首页看到小绿勾，代表最近一次 action 运行成功。
![Action 运行完毕，博客部署完成](action-success.png)

运行完毕后，博客就已经部署成功了！可以访问一下试试。

这样，我们整条“编辑-构建-部署”的链路就打通了。
全文完。

## 参考资料
[只需5步！在轻量应用服务器部署Hexo博客](https://developer.aliyun.com/article/815625)
[在 GitHub Pages 上部署 Hexo | Hexo](https://hexo.io/zh-cn/docs/github-pages)
