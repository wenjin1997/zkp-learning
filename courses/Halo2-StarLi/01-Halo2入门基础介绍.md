# 01-Halo2入门基础介绍
B站视频链接见[这里](https://www.bilibili.com/video/BV1ML4y1M7iV/?spm_id_from=333.999.0.0&vd_source=c6586ed2410fae637f393017e00f4845)。下面是记录的笔记。
![](./img/halo2-1-1.png)
![](./img/halo2-1-2.png)
* Sonic的验证算法是线性的。
* Sonic里面的算法用的是KZG，而在Halo里面进行了替换，使用了内积证明，进一步将universal setup条件削弱了，不需要setup。
* Halo2
    * 不需要setup
    * 曲线不需要配对，普通离散对数曲线即可
    * 支持递归
![](./img/halo2-1-3.png)
![](./img/halo2-1-4.png)

![](./img/halo2-1-5.png)

* 内积证明科普链接见[这里](https://dankradfeist.de/ethereum/2021/11/18/inner-product-arguments-mandarin.html) 。

![](./img/halo2-1-6.png)
* 在Grouth16中一个很长的线性组合，在Plonk中可能会拆分成几十个gate约束。
* 在zkSync代码中会有标准Plonk的变种。
* 标准Plonk Gate是三元二次多项式，Custom gate可以作是多元多次多项式。
![](./img/halo2-1-7.png)
* Selector: 选择子，由于同一行支持几种不同的约束，可以通过选择子来设置满足几条约束。
![](./img/halo2-1-8.png)
![](./img/halo2-1-9.png)
![](./img/halo2-1-10.png)
* Custom gate如果多项式次数很高的话，会增加商多项式，在Plonk协议中的第三步，会产生很多小的t多项式。 
![](./img/halo2-1-11.png)
![](./img/halo2-1-12.png)
![](./img/halo2-1-13.png)