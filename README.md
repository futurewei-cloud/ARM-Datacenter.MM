# Getting started

## Install jekyll and dependencies
```
sudo apt-get install ruby-full build-essential zlib1g-dev
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
gem install jekyll bundler

```
## Clone ARM-Datacenter repo
```
git clone -b gh-pages https://github.com/futurewei-cloud/ARM-Datacenter.git
cd ARM-Datacenter
bundle update
bundle exec jekyll serve --host 127.0.0.1 --port 8080
```

## Create your branch
```
git checkout -b <your_branch>
# Make your changes test and commit them
git push origin <your_branch>
# It is time to create Pull Request
```
