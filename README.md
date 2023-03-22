# How-to

```shell
# Download Hugo v0.76.5
wget https://github.com/gohugoio/hugo/releases/download/v0.76.5/hugo_0.76.5_Linux-64bit.tar.gz
tar zxvf hugo_0.76.5_Linux-64bit.tar.gz
sudo mv hugo /usr/local/bin/

# Prepare the resource
git clone https://github.com/cvvz/blog.git && cd blog
mkdir themes && cd themes && git clone https://github.com/cvvz/hermit.git 
cd .. && git clone https://github.com/cvvz/cvvz.github.io.git public

# Test locally
hugo server -D

# Deploy to the Internet
./deploy.sh
```
