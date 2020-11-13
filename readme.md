## blog 创建步骤

## step 1. command 

```bash 
hugo new site jack-blog

cd jack-blog

git init

git add .

git commit -m "initial check"

## add the remote git repo with name jack-blog

git remote add origin http://192.168.0.188:9199/jack20170608/jack-blog.git

git push -u origin master


git submodule add https://github.com/jack20170608/maupassant-hugo.git  themes/maupassant

git add .

git commit -m "initial check"

git push

```

## step 2. deploy 

```bash 
ssh docker110

cd ~

git clone http://192.168.0.188:9199/jack20170608/jack-blog.git

## cd to the submodule directly 

cd jack-blog/themes/maupassant

## update the submodule 
git submodule update

## cd to the blog home folder 

cd ~/jack-blog 

## execute the hugo to generate the public 

```

### 版权许可

本书采用“保持署名—非商用”创意共享4.0许可证。

只要保持原作者署名和非商用，您可以自由地阅读、分享、修改本书。

详细的法律条文请参见[创意共享](http://creativecommons.org/licenses/by-nc/4.0/)网站。