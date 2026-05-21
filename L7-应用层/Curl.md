# Curl

## 常见用法

---

### 1、下载 (option: -o 或者 option: -O)

#### 1.1、下载页面

```bash
curl -o dodo1.jpg http://www.linux.com/dodo1.JPG
#要注意-O这里后面的url要具体到某个文件，不然抓不下来
curl -O http://www.linux.com/dodo1.JPG
```

#### 1.2、循环下载

有时候下载图片可以能是前面的部分名称是一样的，就最后的尾缀名不一样。这样就会把 dodo1，dodo2，dodo3，dodo4，dodo5 全部保存下来。

```bash
curl -O http://www.linux.com/dodo[1-5].JPG
```

#### 1.3、下载重命名

在 hello/dodo1.JPG 的文件下载下来就会变成 hello_dodo1.JPG，其他文件依此类推，从而有效的避免了文件被覆盖。

```bash
curl -o #1_#2.JPG http://www.linux.com/{hello,bb}/dodo[1-5].JPG
```

由于下载的 hello 与 bb 中的文件名都是 dodo1，dodo2，dodo3，dodo4，dodo5。因此第二次下载的会把第一次下载的覆盖，这样就需要对文件进行重命名。

```bash
curl -O http://www.linux.com/{hello,bb}/dodo[1-5].JPG
```

#### 1.4、分块下载 (option：-r)

```bash
curl -r 0-100 -o dodo1_part1.JPG http://www.linux.com/dodo1.JPG
curl -r 100-200 -o dodo1_part2.JPG http://www.linux.com/dodo1.JPG
curl -r 200- -o dodo1_part3.JPG http://www.linux.com/dodo1.JPG
cat dodo1_part* > dodo1.JPG  #这样就可以查看 dodo1.JPG 的内容了
```

#### 1.5、通过 ftp 下载文件 (option：-u)

curl 可以通过 ftp 下载文件，curl 提供两种从 ftp 中下载的语法：

```bash
curl -O -u 用户名:密码 ftp://www.linux.com/dodo1.JPG
curl -O ftp://用户名:密码@www.linux.com/dodo1.JPG
```

#### 1.6、下载，显示进度条 (option：-#) 或不显示进度条 (option：-s)

```bash
curl -# -O http://www.linux.com/dodo1.JPG
curl -s -O http://www.linux.com/dodo1.JPG
```

#### 1.7、下载，断点续传 (-C \<offset\>)

断点续传，从文件头的指定位置开始继续下载/上传；offset 续传开始的位置，如果 offset 值为"-"，curl 会自动从文件中识别起始位置开始传输；

```bash
curl -# -o centos6.8.iso -C - http://mirrors.aliyun.com/centos/6.8/isos/x86_64/CentOS-6.8-x86_64-minimal.iso
curl -C -O http://www.linux.com/dodo1.JPG
```

---

### 2、上传文件 (option: -T)

```bash
curl -T dodo1.JPG -u 用户名:密码 ftp://www.linux.com/img/
```

---

### 3、伪造来源页面 | 伪造 referer | 盗链 (option：-e)

很多服务器会检查 http 访问的 referer 从而来控制访问。比如：你是先访问首页，然后再访问首页中的邮箱页面，这里访问邮箱的 referer 地址就是访问首页成功后的页面地址，如果服务器发现对邮箱页面访问的 referer 地址不是首页的地址，就断定那是个盗连了。

```bash
#这样就会让服务器以为你是从www.linux.com点击某个链接过来的
curl -e "www.linux.com" http://mail.linux.com
#告诉爱E族，我是从百度来的
curl -e http://baidu.com http://aiezu.com
```

---

### 4、伪造代理设备 (模仿浏览器)

有些网站需要使用特定的浏览器去访问他们，有些还需要使用某些特定的版本。curl 内置 option: -A 可以让我们指定浏览器去访问网站。

```bash
curl -A "Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.0)" http://www.linux.com
#告诉爱E族，我是GOOGLE爬虫蜘蛛（其实我是curl命令）
curl -A " Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" http://aiezu.com
#告诉爱E族，我用的是微信内置浏览器
curl -A "Mozilla/5.0 AppleWebKit/600 Mobile MicroMessenger/6.0" http://aiezu.com
```

---

### 5、设置 http 请求

#### 5.1、设置 http 请求头 (option: -H 或 option: --head)

```bash
curl -H "Cache-Control:no-cache"  http://aiezu.com
```

#### 5.2、指定 proxy 服务器以及其端口 (option：-x)

很多时候上网需要用到代理服务器(比如是使用代理服务器上网或者因为使用 curl 别人网站而被别人屏蔽IP地址的时候)，幸运的是 curl 通过使用内置 option：-x 来支持设置代理。

```bash
curl -x 192.168.100.100:1080 http://www.linux.com
```

---

### 6、http 响应头

#### 6.1、查看 http 响应头 (option: -I)

```bash
# 看看本站的http头是怎么样的
curl -I  http://aiezu.com
```

输出：

```text
HTTP/1.1 200 OK
Date: Fri, 25 Nov 2016 16:45:49 GMT
Server: Apache
Set-Cookie: rox__Session=abdrt8vesprhnpc3f63p1df7j4; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
Pragma: no-cache
Vary: Accept-Encoding
Content-Type: text/html; charset=utf-8
```

#### 6.2、保存 http 的 response 里面的 header 信息 (option: -D)

```bash
curl -D cookied.txt http://www.linux.com
```

执行后 cookie 信息就被存到了 cookied.txt 里面了。

注意：-c(小写)产生的 cookie 和 -D 里面的 cookie 是不一样的。

---

### 7、发送表单数据

```bash
curl -F "pic=@logo.png" -F "site=aiezu"  http://aiezu.com/
```

---

### 8、Cookie

#### 8.1、发送 cookie (option: -b)

有些网站是使用 cookie 来记录 session 信息。对于 chrome 这样的浏览器，可以轻易处理 cookie 信息，但在 curl 中只要增加相关参数也是可以很容易的处理 cookie。

```bash
curl -b "domain=aiezu.com"  http://aiezu.com
#很多网站都是通过监视你的cookie信息来判断你是否按规矩访问他们的网站的，因此我们需要使用保存的cookie信息。内置option: -b
curl -b cookiec.txt http://www.linux.com
```

#### 8.2、保存 http 的 response 里面的 cookie 信息 (option: -c)

执行后 http 的 response 里面的 cookie 信息就被存到了 cookiec.txt 里面了。

```bash
curl -c cookiec.txt  http://www.linux.com
```

---

### 9、测试一个网址

#### 9.1、测试一个网址是否可达

```bash
curl -v http://www.linux.com
```

#### 9.2、测试网页返回值 (option: -w [format])

```bash
curl -o /dev/null -s -w %{http_code} www.linux.com
```

---

### 10、保存访问的网页 (>>)

使用 linux 的重定向功能保存：

```bash
curl http://www.linux.com >> linux.html
```

---

### 11、请求方式

```bash
curl -i -v -H '' -X POST -d '' http://www.test.com/a/b
```

其中，-X POST -d, -X GET -d, -X PUT -d 分别等价于 -F, -G -d, -P

以 POST 请求为例：

#### 11.1、-X POST -d

##### 11.1.1、POST application/x-www-form-urlencoded

application/x-www-form-urlencoded 是默认的：

```bash
curl -X POST -d "param1=value1&param2=value2" http://localhost:3000/data
```

等价于：

```bash
curl -H "Content-Type:application/x-www-form-urlencoded" -X POST -d "param1=value1&param2=value2" http://localhost:3000/data
```

使用数据文件：

```bash
curl -X POST -d "@data.txt" http://localhost:3000/data
```

其中 data.txt 内容如下：param1=value1&param2=value2

##### 11.1.2、POST application/json

```bash
curl -H "Content-Type:application/json" -X POST -d '{"key1":"value1","key2":"value2"}' http://localhost:3000/data
```

使用数据文件的话：

```bash
curl -X POST -d "@data.json" http://localhost:3000/data
```

其中 data.json 内容如下：{"key1":"value1","key2":"value2"}

再举个例子：

```bash
curl -H "Content-type:application/json" -X POST -d "{\"app_key\":\"$appKey\",\"time_stamp\":\"$time\"}" http://www.test.com.cn/a/b
```

#### 11.2、-F

```bash
curl  -v -H "token: 222" -F "file=@/Users/fungleo/Downloads/401.png" localhost:8000/api/v1/upimg
```

```bash
curl -f http://www.linux.com/error
```

#### 11.3、其它举例

##### 11.3.1

```bash
curl  -X POST "http://www.test.com/e/f" -H "Content-Type:application/x-www-form-urlencoded;charset=UTF-8" \
-d "a=b" \
-d "c=d" \
-d "e=f" \
-d "g=h"
```

##### 11.3.2

错误：`curl -i -G -d "a=b#1&c=d" http://www.test.com/e/f`

正确：要把参数值是特殊符号的用 urlencode 转换过来：

```bash
curl -i -G -d "a=b%231&c=d" http://www.test.com/e/f
```

---

### 12、调试

curl -v 可以显示一次 http 通信的整个过程，包括端口连接和 http request 头信息。

如果觉得还不够，那么下面的命令可以查看更详细的通信过程：

```bash
curl --trace output.txt www.baidu.com
# 或者
curl --trace-ascii output.txt www.baidu.com
```

运行后，请打开 output.txt 文件查看。

```bash
curl --trace output.txt  http://www.baidu.com
curl --trace-ascii output2.txt  http://www.baidu.com
curl --trace output3.txt --trace-time http://www.baidu.com
curl --trace-ascii output4.txt --trace-time http://www.baidu.com
```

举例：有需求每5分钟请求一次 http://www.test.com/a/b 生成一个日志文件。希望一月的日志(正确的和错误的)能写入一个日志文件：

```bash
day=`date +%F`
logfile='/var/logs/www.test.com_'`date +%Y%m`'.log'
/usr/bin/echo -e "\n\n[${day}] Start request \n " >> ${logfile}
/bin/curl -v "http://www.test.com/a/b" -d "ccccc" 1>> ${logfile} 2>> ${logfile} --trace-time
/usr/bin/echo -e "\n\n[${day}] End request\n" >> ${logfile}
```

---

### 13、显示抓取错误

```bash
curl -f http://www.linux.com/error
```
