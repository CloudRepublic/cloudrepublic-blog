# Cloud Republic's blog

Created with Hexo blogging software written in NodeJS.
Hosted on github pages, using a [separate repository](https://github.com/CloudRepublic/cloudrepublic.github.io).

## Create new branch
```bash
git checkout -b feature/<blogpost>
```

## A new draft post
```
hexo new draft <post>
```

## Publish new draft
```
hexo publish draft <post>
```

## A new post without draft
```
hexo new <post>
```

## Clean before you Generate and Deploy to Github pages
```
hexo clean
```

## Deploy to Github pages
Change branch in _config.yml
```yml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: https://github.com/CloudRepublic/cloudrepublic.github.io.git
  branch: <branch>
```
Deploy to Github
```
hexo generate --deploy
```

## Create a Pull Request
Create a Pull Request to merge it into master.


You need to have write rights on the [cloudrepublic.github.io](https://github.com/CloudRepublic/cloudrepublic.github.io) repository.
