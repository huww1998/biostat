---
layout: post
title:  "Welcome to Jekyll!"
date:   2024-01-11 01:30:13 +0800
categories: SAS
tags: simulation SAS
comments: 1
---

# SAS Simulation

SAS 里的基础数值模拟，需要使用data步生成数据，proc步分析数据。

数据生成：data步中，使用do循环生成每个观测，使用rand函数从特定分布中随机抽样。

设定随机数种子：call streaminit(seed)，例如，` call streaminit(123)`。

生成随机变量：rand('distribution',  para1, para1, ...)，例如生成服从二项分布的随机变量，` rand('binomial',0.6,1)`，一次只能生成一个值，所以需要使用do循环，生成设定样本量的随机变量。

- 生成变量服从二项分布的随机变量 $$X$$ ：

```SAS
data x;
   call streaminit(123);
   do i = 1 to 500;
		x = rand('binomial',0.6,1);
		output;
	end;
run;
proc print data = x (obs=10);
run;
```

- 模拟线性回归数据并分析数据，模型 $$Y=\beta_0+\beta_1X_1+\beta_2X_2+\beta_3X_3+\beta_4X_4+e$$：

```SAS
/*数据生成*/
data lineardata(keep = y x1 x2 x3 x4);
   call streaminit(4321);
   do i = 1 to 500;
		x1 = rand('normal', 5, 1.6);
		x2 = rand('normal', 3, 0.9);
		x3 = rand('binomial',0.6,1);
		x4 = rand('binomial', 0.5,1);
		e = rand('normal', 0, 1);
		y = 1 + 1.2*x1 + 1.3*x2 + 1.4*x3 + 1.5*x4 + e;
		output;
	end;
run;
proc print data=lineardata(obs=10);
run; 
/*数据分析*/
proc reg data=lineardata;
	ods output ParameterEstimates=simout;
	model y = x1 x2 x3 x4;
run;
```

- 重复以上过程1000次，并评价OLS估计的MSE，$$MSE=E((\beta-\hat\beta )^2)$$：

```SAS
/*数据生成*/
data lindata;
	call streaminit(123);
	do j=1 to 1000;
		do i=1 to 500;
			x1 = rand('normal', 5, 1.6);
			x2 = rand('normal', 3, 0.9);
			x3 = rand('binomial',0.6,1);
			x4 = rand('binomial', 0.5,1);
			e = rand('normal', 0, 1);
			y = 1 + 1.2*x1 + 1.3*x2 + 1.4*x3 + 1.5*x4 + e;
			nsim = j;
			subjid = i;
			output; 
		end;
	end;
	keep nsim subjid x1 x2 x3 x4 y; 
proc sort; 
	by nsim subjid;
run;
/*数据分析*/
ods select none;
proc reg data=lindata;
	ods output ParameterEstimates=simout;
   model y = x1 x2 x3 x4;
	by nsim;
run;
ods select all;
/*结果评价*/
proc transpose data=simout out=dat_wide name=stat;
    by nsim;
	 id variable;
    var estimate stderr;
run;
proc sql;
	create table mse as
	select 
		mean((intercept-1)**2) as mse0,
		mean((x1-1.2)**2) as mse1,
		mean((x1-1.3)**2) as mse2,
		mean((x1-1.4)**2) as mse3,
		mean((x1-1.5)**2) as mse4 
	from dat_wide
	where stat='Estimate';
quit;
proc print data=mse;
run;
```

结果如下：<br>
![mse](/images/mse.jpg) 
