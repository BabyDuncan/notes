###  java.net.SocketException: Broken pipe

#### 在最近的编程过程中,我遇到了 Broken pipe的问题,我的开发环境是windows,本地调试和运行都没有问题,但是,当我部署在linux机器上面的时候,每一次网络请求都会发生 Broken pipe的问题,可以确定不是程序的问题,根据查找一些资料,最后找到的解决办法是"
    配置环境变量:
    vim /etc/profile
    加入以下行
    _JAVA_SR_SIGNUM=12
    source /etc/profile 
    
#### 即可,再次进行网络请求,没有broken pipe的问题出现,注意:此变量是配置在客户端机器上.
