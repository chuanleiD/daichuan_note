## 网易MuMu模拟器安装 $EdXposed$ 框架
参考：B站：Android玩机爱好者
https://www.bilibili.com/read/cv15996085?spm_id_from=333.999.0.0
https://www.bilibili.com/read/cv16409781?spm_id_from=333.999.0.0




### 一、背景：

​		由于原先安装的雷电模拟器（安卓7.0）与 $Xposed$ 框架无法适配赵学长程序中的一些API函数（API $25$ <API $28$ ），因此希望想办法解决该问题。

​		正好在此时发现 ”网易MuMu模拟器“ 存在一个安卓9专版，正好可以安装我们需要的$EdXposed$ 框架，并且API版本也够高。



### 二、安装模拟器

​		首先，需要安装模拟器。可以选择打包文件中的exe或者在官网下载。最后安装。

- 1_MuMuInstaller_9.0.0.2.exe

- [MuMu模拟器官网_安卓模拟器_网易MuMu手游模拟器 (163.com)](https://mumu.163.com/)

<img src="https://i0.hdslb.com/bfs/album/d74aecd08dfc27199682d9fd99e5c3d6f3ab2442.png" style="zoom:80%;" />



### 三、$Magisk$ 安装

​		之后，需要在模拟器中安装 $Magisk$ 中。

1. 打开网易MuMu模拟器。

   <img src="https://i0.hdslb.com/bfs/album/bc6f4fe398bc0f8b3a56cccca45fd63bcccc8af5.png" style="zoom: 80%;" />

   

2. 开启Root权限。

   <img src="https://i0.hdslb.com/bfs/album/314575f7310e1e6479051e45a87f42d059cc8593.png" style="zoom:80%;" />

   

3. 安装 `2_magisk-on-android-x86.apk` 。

   ![](https://i0.hdslb.com/bfs/album/fb03aebdc3b1764e695ca6c58a711d8d1624c699.png)

   

4. 打开之后输入命令 `inmagisk` 回车，再输入命令 `y` ，输入命令 `y` 之后会弹出索要root权限的窗口，勾选永久记住选择那个选项并点击允许即可。

   ![img](https://i0.hdslb.com/bfs/article/a92b291b136e8580d05ec90b2f1d44fc0e6f0dd7.png@942w_588h_progressive.webp)

   

5. 当出现下图界面时，输入快捷命令1并回车，是选择安装 `magisk` 的意思。

   ![img](https://i0.hdslb.com/bfs/article/e34942208c961e54c5eaf492c3a95e59d6425eb9.png@942w_588h_progressive.webp)

   

6. 当出现以下界面，输入命令a回车是安装MuMu模拟器面具magisk激活工具自带的24.1离线安装包。

   ![img](https://i0.hdslb.com/bfs/article/45737a2426c73f754d77c5d4ddd0fa6f54b7f35b.png@942w_591h_progressive.webp)

   

7. 当看见以下界面时继续输入命令1，选择安装到/system分区。

   ![img](https://i0.hdslb.com/bfs/article/578062fb691f3d542a6d19eaea9493c1a3ee0bd0.png@942w_588h_progressive.webp)

   

8. 当界面出现如下的黄色英文字母，证明MuMu模拟器magisk面具已经安装成功。

   ![img](https://i0.hdslb.com/bfs/article/9a471f566c71772d810ec698a7e568033d030d05.png@942w_590h_progressive.webp)

   

9. 安装 $Magiskmanager$

   `3_Magisk.apk`

   

10. 当以上全部操作都正确操作完成之后，你只需要关闭设置中的root（这个可关可不关，不影响），然后重启网易MuMu模拟器，打开 $Magisk$ 面具，面具已经激活完成。

    ![img](https://i0.hdslb.com/bfs/article/98b0ee2245ea1bfb95f84dcfd7484dc2c4c566bd.png@942w_582h_progressive.webp)

    

### 四、$EdXposed$ 安装

1. 首先，在网易mumu模拟器官方网站下载安卓9的模拟器，应该是能跑光遇的那个版本，并且模拟器已经安装并激活了面具 $Magisk23$ 版本以上才行。

   ![img](https://i0.hdslb.com/bfs/article/a8b90750f09d948655508a021ecd3115a325bd43.png@942w_587h_progressive.webp)

   

2. 首先本地安装刷入25.4的riru模块，待刷入成功之后，重启一次模拟器，再刷入edxposed模块，待刷入完成之后，再次重启一次。

   ![img](https://i0.hdslb.com/bfs/article/a8b90750f09d948655508a021ecd3115a325bd43.png@942w_587h_progressive.webp)

   

3. 首先本地安装刷入 `4_Riru-v25.4.4.r426.05efc94(426).zip`，

   成功后，重启一次模拟器，

   再刷入`5_Riru_-_EdXposed-v0.5.2.2_4683-master(4683).zip`，

   待刷入完成之后，再次重启一次。

   ![img](https://i0.hdslb.com/bfs/article/6c5761462be3ccbc13e1691c29b96ffef7aa5df6.png@942w_581h_progressive.webp)

   ![img](https://i0.hdslb.com/bfs/article/51bce132deb9ba50c0e508b16ff9d79bb0078ec5.png@942w_588h_progressive.webp)

   

4. 安装 $EdXposed Manager$

   `6_EdXposedManager-4.6.2.apk`

   

5. 安装好后的效果图

   ![img](https://i0.hdslb.com/bfs/article/8d3e26fdeb7c6d305913c5c3a479f32507e42837.png@942w_587h_progressive.webp)

   

### 五、网易MuMu模拟器连接Android Studio

[Android Studio 怎么连接MUMU模拟器并永久使用_林池的博客-CSDN博客_android studio连接mumu模拟器](https://blog.csdn.net/weixin_45090657/article/details/104647284)





