# 部署 mdbook 到 github pages

本网站由 mdbook 生成，通过 github actions 持续集成到 github pages，以下为部署方式

## 1. 为项目创建仓库
如 [BlackCloud37/mdbook-blog](https://github.com/BlackCloud37/mdbook-blog)，其 master 分支为 mdbook 项目

## 2. 添加 Github Action
参考 [GitHub Pages Deploy](https://github.com/rust-lang/mdBook/wiki/Automated-Deployment%3A-GitHub-Actions#github-pages-deploy)，创建 `.github/workflows/deploy.yml`，内容为
```yaml
name: Deploy
on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
      with:
        fetch-depth: 0
    - name: Install mdbook
      run: |
        mkdir mdbook
        curl -sSL https://github.com/rust-lang/mdBook/releases/download/v0.4.14/mdbook-v0.4.14-x86_64-unknown-linux-gnu.tar.gz | tar -xz --directory=./mdbook
        echo `pwd`/mdbook >> $GITHUB_PATH
    - name: Deploy GitHub Pages
      run: |
        # This assumes your book is in the root of your repository.
        # Just add a `cd` here if you need to change to another directory.
        mdbook build
        git worktree add gh-pages
        git config user.name "Deploy from CI"
        git config user.email ""
        cd gh-pages
        # Delete the ref to avoid keeping history.
        git update-ref -d refs/heads/gh-pages
        rm -rf *
        mv ../book/* .
        git add .
        git commit -m "Deploy $GITHUB_SHA to gh-pages"
        git push --force --set-upstream origin gh-pages
```

之后所有到 master 的 push 都会 build 当前的 mdbook，并将产物 push 到本仓库的 gh-pages 分支

## 3. 访问网站

gh-pages 分支下的网页会被部署到 `<Username>.github.io/<Reponame>` 域名下，如本项目即为 `blackcloud37.github.io/mdbook-blog`，直接访问即可


## 4. Trouble shooting

第一次推送触发 github action 后，gh-pages 分支存在并且已经包含产物，但是访问 `blackcloud37.github.io/mdbook-blog` 提示 404，参考 `https://stackoverflow.com/questions/11577147/how-to-fix-http-404-on-github-pages`，推送一个空 commit 到 gh-pages 分支即可：
```bash
# 在 gh-pages 分支
git commit --allow-empty -m "Trigger rebuild"
git push
```

另外，仓库可见性必须为 Public