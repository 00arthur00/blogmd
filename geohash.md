---
title: geohash
date: 2018-07-31 23:47:42
tags: [geo, geohash]
---
本文参考[github](https://github.com/GongDexing/Geohash/blob/master/README.md)和[维基百科](https://en.wikipedia.org/wiki/Geohash)

### GEOHASH
GeoHash是目前比较主流实现位置服务的技术，Geohash算法将经纬度二维数据编码为一个字符串，本质是一个降维的过程。它将空间分成一个一个的网格，是一种[空间填充曲线](https://en.wikipedia.org/wiki/Space-filling_curves)，[Z型曲线](https://en.wikipedia.org/wiki/Z-order_curve)。[实际上更像一个N型曲线](https://en.wikipedia.org/wiki/Geohash)。geohash的长度越长经度越高，geohash长度为8时，[精度](https://segmentfault.com/a/1190000002513514)能到19m。

### 一个栗子
|地点|经纬度|Geohash|
|:--:|:--:|:--:|
|鸟巢|116.402843,39.999375|wx4g8c9v|
|水立方|116.3967,39.99932|wx4g89tk|
|故宫|116.40382,39.918118|wx4g0ffe|

水立方就在鸟巢在附近，距离600米左右，而故宫到鸟巢直线距离9公里左右，体现在Geohash上，鸟巢和水立方的前五位是一样的，而鸟巢和故宫只有前4位是一样的，**也就是说Geohash前面相同的越多，两个位置越近，但是反过来说，却不一定正确，**这个在后面会详细介绍。

### geohash.org
算法作者创建了[http://geohash.org/](http://geohash.org/)网站，可以将地理坐标方便的转换为短URL，以方便在邮件、论坛或者网站上引用。

### 原理
将经纬度转换为Geohash大体可以分为三步曲：
- 将纬度(-90, 90)平均分成两个区间(-90, 0)、(0, 90)，如果坐标位置的纬度值在第一区间，则编码是0，否则编码为1。我们用 39.918118 举例，由于39.918118 属于 (0, 90)，所以编码为1，然后我们继续将(0, 90)分成(0, 45)、(45, 90)两个区间，而39.918118 位于(0, 45)，所以编码是0，依次类推，我们进行20次拆分，最后计算39.918118 的编码是 10111000110001011011；经度的处理也是类似，只是经度的范围是(-180, 180)，116.40382的编码是**11010010110001101010**
- 经纬度的编码合并，从0开始，奇数为是纬度，偶数为是经度，得到的编码是**1110011101001000111100000011100111001101**
- 对经纬度合并后的编码，进行base32编码，最终得到**wx4g0ffe**

### 边界问题
两个位置距离得越近是否意味着Geohash前面相同的越多呢？答案是否定的，两个很近的地点[116.3967,44.9999]和[116.3967,45.0009]的Geohash分别是**wxfzbxvr**和**y84b08j2**，这就是Geohash存在的边界问题，这两个地点虽然很近，但是刚好在分界点45两侧，导致Geohash完全不同，单纯依靠Geohash匹配前缀的方式并不能解决这种问题。

在一维空间解决不了这个问题，回到二维空间中，将当前Geohash这块区域周围的八块区域的Geohash计算出来
[116.3967,44.9999] 周围8块区域的Geohash
> <b>y84b08j2</b>, wxfzbxvq, wxfzbxvx, wxfzbxvp, y84b08j8, y84b08j0, wxfzbxvw, wxfzbxvn

[116.3967,45.0009] 周围8块区域的Geohash
> y84b08j3, <b>wxfzbxvr</b>, y84b08j8, y84b08j0, y84b08j9, y84b08j1, wxfzbxvx, wxfzbxvp

[116.3967,44.9999]和[116.3967,45.0009]分别出现在各自附近的区域中，周围8个区域的Geohash怎么计算得到呢？很简单，当Geohash长度是8时，对应的每个最小单元
```java
    double latUnit = (Max_Lat - Min_Lat) / (1 << 20);
    double lngUnit = (Max_Lng - Min_Lng) / (1 << 20);
```
这样可以计算出8个分别分布在周围8个区域的地点，根据地点便可以计算出周围8个区域的Geohash
```
[lat + latUnit, lng]
[lat - latUnit, lng]
[lat, lng + lngUnit]
[lat, lng - lngUnit]
[lat + latUnit, lng + lngUnit]
[lat + latUnit, lng - lngUnit]
[lat - latUnit, lng + lngUnit]
[lat - latUnit, lng - lngUnit]
```
在实际应用中，先根据Geohash筛选出附近的地点，然后再算出距离附近地点的距离.

### 推荐文章
[GeoHash核心原理解析](https://yq.aliyun.com/articles/17233)
[github](https://github.com/GongDexing/Geohash/blob/master/README.md)
[维基百科](https://en.wikipedia.org/wiki/Geohash)
[geohash vs PostGIS](https://yq.aliyun.com/articles/73995)