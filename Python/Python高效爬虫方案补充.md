### Python高效爬虫方案补充

​		在一个小爬虫项目的搬砖过程中需要提速，找到一篇@[大龙](https://www.zhihu.com/people/da-long-37-81)写的好文[《Python高效爬虫方案总结》](https://www.zhihu.com/people/da-long-37-81)
。其中有些问题需要注意，于是记录下来，以供参考。

#### 1.最大的问题，多进程是不能直接引用全局变量值的。<br>

这个问题导致MUTL_PROCESSES_COROUTINE统计数据飞起，但其实是urls为空，逻辑空转的执行时间。

fix：在amain()适当位置增加以下逻辑<br>

    urls.clear()
    for i in range(1, 81):  #total 80 page
        newpage = 'http://tieba.baidu.com/p/3522395718?pn=' + str(i)
        urls.append(newpage) 

#### 2.T/P执行漏url。<br>

在agetsource中增加打印url的语句后发现：
		单独的COROUTINE main(0, 1)可以完整执行所有url；
		t线/p进 + c协的情况下，如果[链接数量:len(urls)]不能被并发数量整除，则会漏掉末尾若干个url。
		(t/p/pool数量不同，漏项不同)<br>		(t=p=pool，否则也会漏项)
原因在于：<br>

    start_index = index * int(len(urls) / pool_size)
    end_index = min(len(urls), start_index + int(len(urls) / pool_size))
    
    for url in urls[start_index:end_index]:
        _ = loop.create_task(agetsource(url))

fix：在其后补充以下逻辑<br>

    rem_page = len(urls) % pool_size
    if ((rem_page != 0) & ((end_index + rem_page) == len(urls))): 
        for url in urls[(len(urls) - rem_page):len(urls)]:
            _ = loop.create_task(agetsource(url))    



#### 3.经过上述修正后，会有些许组织urls的耗费，但相对实际应用中处理逻辑的消耗，应可以忽略。<br>

设置不同的pool数量(COROUTINE中main(m,n)如果修改n值，会只执行1/N的url)和线程、进程数量会有不同的排名，有兴趣的自行测试。

链接数量80，协=线=进的数量

并发数	协	        线+协	    进+协<br>
2	2.996999979	5.016000032	3.665999889<br>
3	4.990999937	4.997999907	3.811000109<br>
4	4.99000001	4.011999846	5.819000244<br>
5	4.20299983	4.04399991	2.198000193<br>
6	1.26699996	10.13499999	5.124000072<br>
7	3.437000275	2.171999931	2.458999872<br>
8	3.988999844	1.187999964	2.776000023<br>
9	3.480999947	2.266000271	3.220000029<br>
10	2.181999683	4.00600028	3.041999817<br>

测试结果与原文有所差异，有些情况下多线程+协程成为了赢家。<br>
所以：<br>
一味增加并发数量不一定提高效率；<br>
T/P的数量与执行环境有一定关系，数量最好还是结合实际应用逻辑、经过测试来决定。

如果是CPU密集型应用，则线程池大小设置为N+1
如果是IO密集型应用，则线程池大小设置为2N+1
(http://ifeve.com/how-to-calculate-threadpool-size/)