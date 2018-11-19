# ZimgTree

<pre>
Zimg提供如下功能
   1）所有图片默认返回质量为75%，JPEG格式的压缩图片，这样肉眼无法识别，但是体积减小
   2）获取宽度为x,被等比例缩放的图片
   3）获取旋转后的图片
   5）获取指定区域或固定大小的图片
   6）获取特定尺寸的图片，由于与原图比例不同，尽可能展示最多的图片内容，缩放之后多余的部
      分需要裁掉
   7）获取特定尺寸的图片，要展示图片所有内容，因此土坯那会被拉伸到新的比例而变形
   8）获取特定尺寸的图片，但是不需要缩放，只用展示图片的核心内容即可
   9）获取按指定百分比缩放的图片
   10）获取指定压缩比的图片
   11) 获取去除颜色的图片
   12）获取指定格式的图片
   13）获取图片的信息
   15）删除指定图片
</pre>

<pre>
     Zimg是针对图片处理服务器而设计开发的开源程序，它拥有很高的性能，也满足了应用在图片方面
   最基本的处理需求，

总体思路：
     想要在展现图片这件事情上有最好的表现，首先需要从整体业务中将图片服务部分分离出来，使
   用单独的域名和建立独立的图片服务器有很多好处。比如：
        1）CDN分流。 热门网站的图片地址都有特殊的域名，比如微博的是www.sinaimg.cn等等，
           域名不同可以再CDN解析的层面就做到非常明显的优化效果。
        2）浏览器并发连接数限制。 一般来说，浏览器加载HTML资源时会建立很多的连接，并行的
           下载资源。不同的浏览器对同一主机的并发连接数限制是不同的，比如IE8是10个，
           Firfox是30个，如果把图片服务器独立出来，就不会占用对主站资源连接数的限制，一定
           程度上提高网站的性能。
        3）浏览器缓存。 现在的浏览器都具有缓存功能，但是由于cookie的存在，大部分浏览器不会
           缓存带有cookie的请求，导致的结果是大量的图片请求无法命中，只能重新下载，独立
           域名的图片服务器，可以很大程度上环节此问题。
        
       图片服务器被独立出来后，会面临两个选择，主流的方案是前端采用Nginx，中间是PHP或
    者自己开发的模块，后端时物理存储，比较特别一些的，比如facebook，他们把图片的请求处理
    和存储合并一体，叫做haystack，这样做的好是haystack只会处理与图片相关的请求，比例了
    普通http服务器繁杂的功能，更加轻量级，同时使得部署和运维难度降低。

       Zimg采用的是与Facebook相似的策略，将图片处理的大权收归自己所有，绝大部分事情由自己
    处理，除非特别必要，最小程度地引入第三方模块。
       在Zimg的1.0版本，设计面向图片量在TB级别的中小型服务，物理存储暂时不支持分布式集群，
    分布式功能将在2.0版本中完成。
</pre>

<pre>
架构设计：
   为了极致的性能表现，Zimg全部采用C语言开发，总体上分为3个层次
       1）前端http处理层
       2）中间图片处理层
       3）后端的存储层
   http处理层引入基于libevent的libevhttp库，libevhttp库是一款专门处理http请求的库，它
   太适合Zimg的业务场景了，在性能和功能之间找到了很好的平衡。图片处理层采用imagemagick库，
   imagemagick库是现在公认功能最强，性能最好的图片图片处理函数库，存储层采用memcached缓存
   加直接读写硬盘的方案，更加深入的优化将在后续进行，比如引入TFS4等，为了避免数据库带来的
   性能瓶颈，Zimg不引入结构化数据库，图片的查找全部采用哈希来解决。

   事实上图片服务器的设计，是一个在I/O与CPU运算之间的博弈过程，最好的策略当然是继续拆：
      CPU敏感的http和图片处理层不属于运算能力更强的机器上，内存敏感的cache层部署于内存更大
   的机器上，I/O敏感的物理存储层则放在配备SSD的机器上，但并不是所有人都能负担得起这么奢侈
   的配置，Zimg折中成本和业务需求，目前只需要部署在一台服务器上，将压力放在CPU上，事实证明
   这样的思路基本没错，在硬盘性能很差的机器上效果更佳明显：即使以后SSD全面普及，CPU的运算
   能力也会相应提高，总体来说Zimg的方案也不会太失衡。
</pre>

<pre>
代码层面：
       虽然Zimg在二进制实体上没有分模块，现阶段面向中小型的服务，单机部署即可，但是代码上
    是分离的。

main.c是程序的入口，主要功能是处理启动参数：
-p 监听端口
-t 线程数，默认4，请调整为具体的CPU核心数
-k 最高保持连接数，默认1， 不启用长连接， 0为启用
-l 启用Log，会带来很大的性能损耗
-M 启用缓存的连接IP
-m 启用缓存的连接端口
-b 每个线程的最大连接数，默认为1024

Zhttpd.c是解析HTTP请求的部分，分为GET和POST，GET请求会根据请求的URL参数去寻找图片并转给
图片处理器层，最后将结果返回给用用户，
POST接收上传请求然后将图片存入计算好的路径中。

在Zimg中图片的唯一key值就是图片的MD5值，这样既可以隐藏路径，又能减少前端和Zimg自身的存储
压力，是避免引入结构化存储部分的关键，所有所有的GET请求都是基于MD5拼接而成的。

设想一下，加入网站的某个地方需要展示一张图片，这个图片的原图的大小为1000 * 1000，但是你
想要展示的地方只有300 * 300， 一般还是依靠CSS来进行控制，但是这样的话会造成很多流量的
浪费。为此，Zimg提供了图片裁剪功能，你锁要做的就是在土片的URL后面加上w=300&h=300(width,
height)即可，另一个场景是图片灰白化，比如某天遇到重大自然灾害，想要网站所有图片变成灰白
的，那么只需要在图片URL后面再加上g = 1(gray)即可。

当然，依托于imagemagic所提供的完善的图片处理函数，Zimg将在后续版本中逐步增加该功能，比如
水印等。

在图片上传部分，其实能玩的花样很少，但是编写代码锁消耗的时间最多，现在，再假设另一种场景，
如果我们的图片服务器前端采用Nginx，上传功能用PHP实现，需要写的代码很少，但是性能如何呢？
答案是很差，首先PHP接收到Nginx传过来的请求后，会把Http协议分离出其中的二进制文件，存储在
一个零时目录里，等我们在PHP代码里使用$_FILES["upfile"][tmp_name]获取到文件后计算MD5再
存储到指定目录里，在这个过程中有一次对哦文件是多余的，起止最好的情况是我们拿到Http请求中的
二进制文件（最好在内存里），直接计算MD5然后存储。

Zimg.c是调用imagemagick处理图片的部分，这里先解释一下在zimg中图片存储路径的规划方案。
现阶段zimg服务于存储量在TB级别的单机图片服务器，所以存储路径采用了二级子目录的方案。
由于Linux目录下的子目录数最好不要超过2000个，再加上MD5的值本身就是32位16进制数，zimg
就采用了一种非常取巧的方式：
    根据MD5的前6位进行哈希，1~3位转换为16进制数后除以4，范围正好落在1024以内，
    以这个数作为第一级子目录；
    4~6位同样处理，作为第二级子目录；
    二级子目录下是以MD5命名的文件夹，每个MD5文件夹内存储图片的原图和其他根据需要存储的
    版本，假设一个图片平均占用200KB,一台Zimg服务器支持的总容量就可以计算出来了：
        1024 * 1024 * 1024 * 200KB = 200TB
    这样的数量应该已经算很大了，在200TB的范围内可以采用加硬盘的方式来扩容，当然如果有更
  大的需求，可以试试Zimg后续的版本。

除了路径规划，zimg另一大功能就是压缩图片，从用户的角度来说，zimg返回来的图片只要看起来
跟原图差不多就行了，如果确实需要原图，也可以通过将所有参数置空的方式来获得，zimg.c对于
所有转换的图片都进行了压缩，压缩之后肉眼几乎无法分辨，但是体积将减少67.05%，具体的处理
方式为：
    图片裁剪时使用LanczosFilter滤镜
    以75%的压缩率进行压缩
    去除图片的Exif信息
    转换为JPEG格式

经过这样的处理之后可以很大程度的减少流量，实现设计目标。

zcache.c是引入memcached缓存的部分，引入缓存是很重要的，尤其是图片量级上身以后，在zimg
中缓存被作为一个很重要的功能，几乎所有zimg.c中的查找部分都会先检查缓存是否存在，比如：
    我想要a(代表设计MD5)图片裁剪为 100 * 100之后再灰白化的版本，那么过程是先去找
  a&w=100&h=100&g=1的缓存是否存在，不存在的话去找这个文件是否存在，还不存在就去照这个分辨率的彩色图缓存是否存在，若依然不存在就去找彩色图文件是否存在，若还是没有，那就去查询原图
  的缓存，原图缓存依然未命中的话，只能打开原图文件了，然后开始裁剪，灰白化，然后返回给用户并
  存入缓存中。

  可以看出，上面过程中如果某个环节命中缓存，就会相应的减少I/O或图片处理的次数，众所周知，
  内存和硬盘的读写速度差距是巨大的，那么这样的设计对于热点图片抗压就十分重要。
</pre>