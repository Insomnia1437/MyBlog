## MyBlog

Based on Hugo and [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme

```shell
# install theme
# git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

git clone https://github.com/Insomnia1437/MyBlog
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)

# update theme
git submodule update --remote --merge

# new page
hugo new posts/a-title.md

# start local server
hugo server
```