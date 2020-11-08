# Anthony Comtois' Portfolio and Blog

This blog is using Hugo. 

## Hugo installation 

## Download hugo

```
wget https://github.com/gohugoio/hugo/releases/download/v0.77.0/hugo_0.77.0_Linux-64bit.tar.gz
tar xzf hugo_0.77.0_Linux-64bit.tar.gz
sudo cp hugo /usr/local/bin/hugo
```

## Starting development server 

```
hugo server -t toha -w --bind 0.0.0.0
hugo server -t toha -w --bind 0.0.0.0 -b http://172.16.0.2  
```

## Add new blog post 

1. use `source` branch 
2. make changes, commit an push `git push origin source`
3. github actions will generate website and commit in master branch 
4. website will be available on https://rewiko.github.io/