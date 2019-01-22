### 背景
因为之前一直在捣鼓Python，本人是前端菜鸟一枚~，一直想着做个小demo，把后端和前端给统一跑起来，正好也可以把掌握的知识梳理一遍，这样前端后端就能够打通，岂不快哉！平时上班大家都知道没时间干别的，乘着这两天周末，决定把之前搭好的开发环境，给弄到线上去，其实开发环境也搞了一两天，因为django的csrf的坑，还有就是因为前后端要分离吗，还要搭建前端开发环境。前端是webapck + react + react-router + mobx + axios + antd,后端是django + mysql + uwsgi + nginx,这是我第一次部署，所以踩了很多坑，包括阿里云也是第一次使用，希望给别的小伙伴一个参考，要是能在这边文章中解决问题，那就开心了，废话不多说，开始正题。

### 环境准备
首先你要有一台线上服务器，先配好服务端的环境，我用的是阿里云的ECS服务器，Linux Ubuntu16.04版本。系统不一样可能下面需要安装的软件依赖也会不同，不过大同小异。不是什么大不了的。登录阿里云控制台，找到服务器实例


![](https://user-gold-cdn.xitu.io/2018/12/23/167dbbba9a7aae6c?w=1033&h=209&f=png&s=58587)
点击远程链接,然后会跳到远程链接页面，页面会有一个弹框，输入远程链接密码，这个密码要记下来，后面登录时会一直使用。登录后的界面


![](https://user-gold-cdn.xitu.io/2018/12/23/167dbc22b7f82334?w=1145&h=280&f=png&s=30759)
强烈建议不要使用这个终端，因为太坑，你只能通过clear命令清屏，不能上下翻屏，也不能回滚。。。建议使用ssh命令在本地远程登录: ssh 用户名@ip地址 用户名是你服务器上的用户名，ip地址是你服务器上的公网ip，新买来的服务器默认只有root用户名(第一次使用一脸懵逼，就在想这TM用户名是啥我怎么驾驭它。。笨的要死)，建议不要用root账户操作，useradd 用户名 新建一个用户名，然后添加sudo权限，这个很重要哦，不然很多命令没有权限，关于Ubuntu命令自行百度啦。为服务器添加安全组，后面项目会用到端口号，添加22、3306、8080等，用到什么端口加添加什么端口，这里也是一颗坑，不然项目根本无法访问。

* 安装依赖
当添加好新的账户后，后面所有的操作都将在这个账户下操作了。安装nginx
`sudo apt-get install nginx`这是我用的版本。可能会有别的提示，这是因为需要更新镜像，`sudo apt-get update`然后在下载应该就可以了,nginx用于与uwsgi配合做请求转发。下载好的nginx的默认路径在 /etc/nginx 这个目录，网上有人说在/usr/local/nginx，这个看具体的环境吧，我觉得这种情况可能是因为安装的方式不同造成的，我是直接install的，网上有的是下载.tar压缩包然后在解压，这样可能就会造成nginx的安装路径不同。

* 然后启动nginx `sudo service nginx start` 在浏览器中输入ip地址，如果显示wenlcome....证明nginx正常。如果nginx没有效果，使用此命令`systemctl status nginx.service`，可以查看具体报错信息。接着安装mysql，`sudo apt-get install mysql-server` 启动服务`sudo service mysql start` 查看进程`ps ajx|grep mysql` 停止服务`sudo service mysql stop` 重启服务`sudo service mysql restart`配置mysql进入`/etc/mysql/mysql.cnf.d` `sudo vim mysql.cnf`找到bind-address表示服务器绑定的ip，默认为127.0.0.1，把ip地址改为服务器的IP。安装过程中会让你输入密码这个设置下就好，一会用于登录，接着安装mysql客户端`sudo apt-get install mysql-client` 客户端安装完，就可以登录账户了，通过`mysql -uroot -pxxxxxx`,默认只有root账户，登陆后添加mysql用户，并设置密码和权限如：`grant select on 数据库.表 to 'laowang'@'localhost' identified by '123456';`具体自行百度吧，设置好后这是用于django项目在线上操作mysql数据库时要用的用户名和密码等，注意权限的问题，这里也是容易犯错的地方！
* 安装python虚拟环境，这个虚拟环境很多人开发不一定能用到没有用到的请跳过吧，不过还是建议使用，因为很方便管理维护项目，服务器自带的python版本有2.7和3.5，所以不用安装Python


``` python
sudo pip install virtualenv #安装虚拟环境
sudo pip install virtualenvwrapper #安装虚拟环境扩展包
编辑家目录下面的.bashrc文件，添加下面两行。
export WORKON_HOME=$HOME/.virtualenvs
source /usr/local/bin/virtualenvwrapper.sh
使用source .bashrc使其生效一下。
创建虚拟环境命令：
	mkvirtualenv 虚拟环境名
创建python3虚拟环境：
mkvirtualenv -p python3 name


进入虚拟环境工作：
	workon 虚拟环境名
查看机器上有多少个虚拟环境：
	workon 空格 + 两个tab键
退出虚拟环境：
	deactivate
删除虚拟环境：
	rmvirtualenv 虚拟环境名
虚拟环境下安装包的命令：
pip install 包名
注意：不能使用sudo pip install 包名，这个命令会把包安装到真实的主机环境上而不是安装到虚拟环境中。
apt-get install 软件
pip install python包名
```
虚拟环境主要是用于隔离项目依赖的版本，多个项目互相不影响

现在进入你本地开发好的项目根目录，执行`pip freeze > list.txt`这是导出你这个项目所依赖的全部包，一会要在服务器上的虚拟环境下载这些包，本地通过ssh 命令链接服务器，然后通过scr -r 命令把本地项目上传到服务器，上传前记得把项目的settings.py这个文件配置文件的`ALLOWED_HOSTS = ['*',]或者写你的服务器ip, DEBUG = False `这个要新建一个文件夹。比如/home/abc/wwwroot/  abc是你服务器当前的账户名，切换到虚拟环境，进入项目然后`pip install -r list.txt`下载依赖。这个时候你可以通过python3 manage.py runserver 跑起你的项目，看是否正常，如果一切正常，说明环境什么的都是OK的，中间如果有什么不对，缺什么下载什么.

* 接着安装uwsgi,切换虚拟环境，pip install uwsgi，在这个过程中可能会出错，这是因为系统缺少依赖，另外就是所用的python版本造成的，我用的是3.5的版本，然而系统好像默认是2.7。比如这些`libpcre3 libpcre3-dev   zlib1g-dev   openssl libssl-dev   python3-dev  python-dev`,缺什么下载什么，可以通过`dpkg -l | grep zlib`这个命令查看系统是否有对应的依赖，然后通过sudo apt-get install 下载。 进入项目根目录,sudo vim uwsgi.ini 文件叫什么名字无所谓，然后加上下面内容
``` python
[uwsgi]
#使用nginx连接时使用
#socket=127.0.0.1:8080
#直接做web服务器使用
http=127.0.0.1:8080
#项目目录
chdir=xxxxxxx
#项目中wsgi.py文件的目录，相对于项目目录
wsgi-file=xxxxxx
processes=4
threads=2
master=True
pidfile=uwsgi.pid
daemonize=uwsgi.log
#virtualenv=xxxxx


```
这是我的配置，可以看到下面有一个home和virtualenv配置它们的路径是一样的，效果却不一样如果我用virtualenv它就跑不起来，用home就可以，这个坑踩了好长时间。。。都是泪，它们视乎都是用户虚拟环境的配置，有空在查下资料它们的区别。
![](https://user-gold-cdn.xitu.io/2018/12/24/167de18c75fad4ae?w=478&h=360&f=png&s=31346)
进行到这一步，上面我用的是socket的链接，这是和nginx配合使用的，这个不急一会再说，现在先把socket注释掉，把http打开，后面的端口号根据自己项目来定。然后保存退出编辑，通过命令，此时所在的目录是项目根目录，`uwsgi --ini uwsgi.ini` 启动uwsgi服务，这个时候就不用python3 manage.py runserver了，uwsgi就可以充当web服务器，这样是检验当前的配置依赖，一切是否都正常，启动后，可以在浏览器中输入IP地址加端口号以及你配置的urls的路径，看是否返回正常，注意这个时候是测试，所有请用get请求，不要用post 因为post有csrf的问题。或者直接在命令行中输入python3,进入python执行环境,`import requests ` 然后`requests.get('127.0.0.1:8888/xxxx')`查看返回状态。若是ok那么到目前一切正常,如果有错误  请查看项目根目录下面生成的uwsgi.log日志文件，查看日志文件，是解决问题的最好办法。如果没问题，就把socket打开，http注释掉，下面要把nginx和uwsgi结合起来。

* 再次之前把通过webapck打包好的前端dist文件夹上传到服务器，位置应该没有影响，我放在nginx的目录下面了。找到nginx目录，/etc/nginx/nginx.conf：
``` python
	server {
		listen       80;
		server_name  localhost;

		#charset koi8-r;

		#access_log  logs/host.access.log  main;

		#location / {
		#    root   html;
		#    index  index.html index.htm;
		#}
	        # root  /etc/nginx/dist; 		
	       location / { 
			
		           add_header 'Access-Control-Allow-Origin' '*'; 
                           add_header 'Access-Control-Allow-Credentials' 'true';
                           add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
                           add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
                           root  /etc/nginx/dist;      #"这里就要指定你的前端目录文件了,也就是刚刚放进nginx根目录的文件夹"
                           index html index.html;     #"build 目录下默认有index.html 指定默认文件"
                           try_files $uri /index.html;   #"这块分重要,曾经不加尝试过,当我访问login路径时,他不会自动跳转,具体自行百度"
	                   #error_page 405 =200 $uri; 
		}
               location  ~ /api/* {#这个是前端ajxa的请求地址，比如33.66.0.1:8888/api/register,都会转发给django后端，这个和开发环境还是有区别的，前端可以通过webpack的devserver做转发，所有后端的地址在上线时，要注意和这个设置的路由要匹配
                           include uwsgi_params;
                           uwsgi_pass 127.0.0.1:8888; 
               }
	    #这个是你通过webpack打包后的html页面加载的文件路径
		location /static {
			# 指定静态文件存放的目录
			alias /etc/nginx/dist/;	
		#	root /home/zjp/wwwroot/dist;
		#        index index.html index.html;
                #        try_files $uri /index.html;
		}

		#location = / {
		#	proxy_pass http://172.16.179.131;	
		#}
		#error_page  404              /404.html;

		# redirect server error pages to the static page /50x.html
		#
		error_page   500 502 503 504  /50x.html;
		location = /50x.html {
		    root   html;
		}

		# proxy the PHP scripts to Apache listening on 127.0.0.1:80
		#
		#location ~ \.php$ {
		#    proxy_pass   http://127.0.0.1;
		#}

		# pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
		#
		#location ~ \.php$ {
		#    root           html;
		#    fastcgi_pass   127.0.0.1:9000;
		#    fastcgi_index  index.php;
		#    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
		#    include        fastcgi_params;
		#}

		# deny access to .htaccess files, if Apache's document root
		# concurs with nginx's one
		#
		#location ~ /\.ht {
		#    deny  all;
		#}
      }
```
这里面一个坑就是要把`#include /etc/nginx/conf.d/*.conf;
	#include /etc/nginx/sites-enabled/*;`注释掉，因为当访问80端口的时候，会默认把/var/www/html/下的文件给你返回，这样你自己的静态文件就无法显示，这是因为里面有默认的配置。更多nginx的配置自行百度。接下来就是收集django所依赖的静态文件了，因为是前后端分离的项目，所以django所需要的静态文件并不多，需要在服务器上新建一个文件夹，来存放所收集的静态文件，然后记住这个路径，在项目的settings.py的加上：
``` python
STATIC_ROOT = '/var/www/reactObject/static/'

```
保存，提醒：这时候还要注意mysql的配置，因为现在是线上环境了，所以settings文件中的数据库的账户也要记得修改。然后执行`python manage.py collectstatic`，就会把文件存放在STATIC_ROOT这个指定的目录下，然后在上面的nginx的目录中配置对应的路由。因为我的这个项目静态文件都在前端，所以我收集的时候，没有文件。也就没有配置对应的路由。

另外在我配置nginx和uwsgi的时候，我在网上看到一个小伙伴说要把下面红色的文件
![](https://user-gold-cdn.xitu.io/2018/12/24/167def06f95f8c4e?w=1111&h=50&f=png&s=8580)放在项目根目录的下面，我信你个鬼，你这糟老头子坏的很~ 我把它放在了项目目录的下面怎么都跑不起来，又是检查各种配置是否有误，检查log文件等。。。都是泪不说了，后来也看了别人的博客，发现根本没有这个文件，而且这个文件都是默认安装的，不需要动它。

到了这一步基本配置都差不多结束了。重新启动nginx 和uwsgi。这个时候你在浏览器中输入IP地址加端口号，没问题的话就可以出来页面了。

如果你有域名的话，可以把域名跟服务器绑定，通过域名解析就可以啦~我的小demo，当我看见页面完完整整出来的时候，心里真是太爽了！这里面其实有很多可以优化的地方，后面再说，先跑通才是最要紧，觉得不错给个赞吧
