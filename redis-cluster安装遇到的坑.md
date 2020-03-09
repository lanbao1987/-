##redis cluster安装遇到的坑
###yum安装
```
yum install -y ruby
yum install -y rubygems
gem install redis
```
1. yum安装的ruby为1.8版本
第三步gem时报错须ruby>2.2.2
```
gem install redis
   ERROR:  Error installing redis:
           redis requires Ruby version >= 2.2.2.
```
yum list |grep ruby没有2.2以上版本

2. 百度用rvm替换ruby版本 但 bash时一直报错 识别不了版本号
```
curl -sSL https://get.rvm.io | bash -s stable
```
```
[root@eshop-cache01 ruby-2.3.1]#  curl -L get.rvm.io | bash -s stable
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
194   194  194   194    0     0    231      0 --:--:-- --:--:-- --:--:--   475



100 24168  100 24168    0     0    674      0  0:00:35  0:00:35 --:--:-- 14890



Downloading https://github.com/rvm/rvm/archive/.tar.gz
curl: (35) SSL connect error

Could not download 'https://github.com/rvm/rvm/archive/.tar.gz'.
  curl returned status '35'.

Downloading https://bitbucket.org/mpapis/rvm/get/.tar.gz
curl: (7) Failed to connect to 2406:da00:ff00::22c0:3470: Network is unreachable

Could not download 'https://bitbucket.org/mpapis/rvm/get/.tar.gz'.
  curl returned status '7'.


```
###2. 一怒之下转源码安装
```
wget https://cache.ruby-lang.org/pub/ruby/2.3/ruby-2.3.1.tar.gz
tar -zxvf ruby-2.3.1.tar.gz
./configure -prefix=/usr/local/ruby
make && make install
cd /usr/local/ruby
cp bin/ruby /usr/local/bin
cp bin/gem /usr/local/bin

wget http://rubygems.org/downloads/redis-3.3.0.gem
gem install -l ./redis-3.3.0.gem
gem list 
```
1. ruby -v不能识别 因为目录不对 用ln建立映射
```
ruby -v
-bash: /usr/bin/ruby: No such file or directory
```
```
ln -s /usr/local/bin/ruby /usr/bin/ruby
ln -s /usr/local/bin/gem /usr/bin/gem
```
```
[root@eshop-cache01 bin]# ./ruby -v
ruby 2.3.1p112 (2016-04-26 revision 54768) [i686-linux]
```
2. gem 报错zlib
```
ERROR:  Loading command: install (LoadError)
    cannot load such file -- zlib
ERROR:  While executing gem ... (NoMethodError)
    undefined method `invoke_with_build_args' for nil:NilClass
```
ruby没有zlib插件 进入ruby源码目录ext安装 /usr/local/ruby-2.3.1
```
cd ext/zlib
ruby ./extconf.rb
make
make install
```
3. gem报错openssl
```
Unable to require openssl,install OpenSSL and rebuild ruby (preferred) or use
```
ruby没有ssl插件 进入ruby源码目录ext安装 /usr/local/ruby-2.3.1
```
cd ext/openssl
ruby ./extconf.rb
make
make install
```
4. openssl make时报错
```
make: *** No rule to make target /include/ruby.h', needed byossl.o’. Stop.
```
打开Makefile文件，在顶部添加 top_srcdir = …/…
```
vi Makefile


#### Start of system configuration section. ####
top_srcdir=../..
srcdir = .

```
gem install redis 终于成功
可以跑redis cluster了
后记：<b>没有银弹</b>
```
搞了2个小时 最后是通过源码安装好的 感觉通过yum rvm也可以安装好ruby2.2.2+
源码安装感觉更麻烦 还是没有银弹 选了条路出错了就要google走下去
rvm那个网络问题 应该也可以修复好往下走
```