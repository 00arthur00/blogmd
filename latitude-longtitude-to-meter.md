---
title: 经纬度到米的转换
date: 2018-07-25 23:31:37
tags: [算法，经纬度， GEO]
---
## 概念
地球的纬度(latitude)是指南北指向的离赤道的角度，用度数来表示，**而不是弧度**。以赤道为界限，北纬为正数，南纬为负数。
纬线是相同纬度的点组成的同心圆，赤道的纬线最长为约为111km，纬线最小的为南北极圈，最小为0。

经度(longitude) 也是认为定义出来的辅助计量，经度相同的值组成的线为经线（子午线），**不用弧度表示**。经线为从南极到北极的线。经度的分界线不限纬度有天然的赤道，经线是认为定义出来的，为格林威治为分界线，东西各180度。

## 转换及表示
度分秒的进制单位是60。
地球的子午线总长度大约40008km, 纬度1度 = 大约111.1km
经度1度的长度和纬度有关，如赤道1度的长度和北京所在纬度的长度肯定不相同。假设纬度为A，则其对应的1度的经线的长度为: 
```
2*PI*R*cos(A/180*PI)/360
```
A/180*PI先转化为弧度，才能进行cos计算。R为地球半径。R*cos(A/180*PI)，得到纬线半径。除以360得到每度的长度。
地球半径是6378137m,简化以下为:**111319.49079327358*cos(A/180*PI)**

* [快速计算](https://gis.stackexchange.com/questions/2951/algorithm-for-offsetting-a-latitude-longitude-by-some-amount-of-meters)
If your displacements aren't too great (less than a few kilometers) and you're not right at the poles, use the quick and dirty estimate that 111,111 meters (111.111 km) in the y direction is 1 degree (of latitude) and 111,111 * cos(latitude) meters in the x direction is 1 degree (of longitude).

* [经纬度加减距离得到新的经纬度](https://stackoverflow.com/questions/23117989/get-the-max-latitude-and-longitude-given-radius-meters-and-position)
> First, every degree of latitude contains appx 111.1 km, so it's easy to recalculate linear delta to latitude delta.
> Second, linear appearance of 1 longitude degree is different and depends on latitude: small close to poles, large close to equator. Approximate equation is the following:
> kmInLongitudeDegree = 111.320 * Math.cos( latitude / 180.0 * Math.PI)
> Combining this, it's easy to get deltas of latitude and longitude that will cover your circle:
```
deltaLat = radiusInKm / 111.1;
deltaLong = radiusInKm / kmInLongitudeDegree;

minLat = lat - deltaLat;  
maxLat = lat + deltaLat;
minLong = long - deltaLong; 
maxLong = long + deltaLong;
```
For more precise calculation, look here: http://en.wikipedia.org/wiki/Longitude (section Length of a degree of longitude).

## 给出两点（经纬度）计算距离
js代码,[解释](https://en.wikipedia.org/wiki/Haversine_formula)
```
function measure(lat1, lon1, lat2, lon2){  // generally used geo measurement function
    var R = 6378.137; // Radius of earth in KM
    var dLat = lat2 * Math.PI / 180 - lat1 * Math.PI / 180;
    var dLon = lon2 * Math.PI / 180 - lon1 * Math.PI / 180;
    var a = Math.sin(dLat/2) * Math.sin(dLat/2) +
    Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
    Math.sin(dLon/2) * Math.sin(dLon/2);
    var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    var d = R * c;
    return d * 1000; // meters
}
```
