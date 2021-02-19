## fio期望和标准差的计算与证明

fio在多个地方使用了期望和标准差来评估采样的数据。除了IO测试过程中的各项参数，
在校准CPU时钟时也有用到。fio的样本期望和标准差是根据两次采样数据之间的差异动态计算的，
不同于常见的已知全部标本数据再计算的方式。下面的代码表示采样CPU每毫秒的时钟50次并计算期望和标准差。
它的计算方法很简洁实用。非常省空间，不用记录全部的数据最后再计算。由于没有找到它对应的公式,
尝试了下根据期望，方差和标准差的基本定义推导出它所使用的方程式以确保它的可靠性。

    #define NR_TIME_TIMES 50

    static int calibrate_cpu_clock(void)
    {
        double delta, mean, S;
        uint64_t cycles[NR_TIME_ITERS];

        cycles[0] = get_cycles_per_msec();
        S = delta = mean = 0.0;
        for (i = 0; i < NR_TIME_ITERS; i++) {
                cycles[i] = get_cycles_per_msec();
                delta = cycles[i] - mean;
                if (delta) {
                        mean += delta / (i + 1.0);
                        S += delta * (cycles[i] - mean);
                }
        }
        
        S = sqrt(S / (NR_TIME_ITERS - 1.0));

        ...
    }

## 证明

证明过程使用的符号，X表示随机变量， E表示期望，S表示方差或标准差。

### 期望

En = (X1 + ... + Xn) / n  
En+1 = (X1 + ... + Xn + Xn+1) / (n + 1)  

En+1 = (nEn + Xn+1) / (n + 1)  
En+1 = (nEn + En + Xn+1 - En) / (n + 1)  
En+1 = ((n+1)En + (Xn+1 - En)) / (n + 1) 

期望的方程式

En+1 = En + (Xn+1 - En) / (n + 1)

证明方差时用到的变换式  

(n+1)(En+1 - En) = Xn+1 - En    式1  
Xn+1 = (n+1)En+1 - nEn          式2  
n(En+1 - En) = Xn+1 - En+1      式3

### 标准差

首先推导方差

Sn = (X1 - En)^2 + ... + (Xn - En)^2
Sn+1 = (X1 - En+1)^2 + ... + (Xn - En+1)^2 + (Xn+1 - En+1)^2

Sn+1 - Sn = (Xn+1 - En+1)^2 + (2(X1 + ... + Xn) - nEn+1 -nEn)(En - En+1)  
Sn+1 - Sn = (Xn+1 - En+1)^2 + (2nEn - nEn+1 - nEn)(En - En+1)  
Sn+1 - Sn = (Xn+1 - En+1)^2 + (nEn - nEn+1)(En - En+1)  
Sn+1 - Sn = (Xn+1 - En+1)^2 + n(En+1 - En)^2  

代入式2  
Sn+1 - Sn = (n(En+1 - En))^2 + n(En+1 - En)^2  
Sn+1 - Sn = n(n+1)(En+1 - En)^2  

Sn+1 - Sn = (n+1)(En+1 - En) * n(En+1 - En)

代入式1和式3  
Sn+1 - Sn = (Xn+1 - En)(Xn+1 - En+1) 

方差的方程式

Sn+1 = Sn + (Xn+1 - En)(Xn+1 - En+1) 

采样第n+1个数据，方差除以n（非n+1）再开平方根得到标准差方程式

Sn+1 = sqrt(Sn+1 / (n + 1) - 1)