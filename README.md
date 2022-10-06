# fastncn
&emsp;&emsp;根据文献报道，从网络图中提取邻阶矩阵（0、1矩阵，0代表指标间没有边，1代表指标间有边），然后依次删除节点并求剩余矩阵的“特征值”。通过将计算结果与删除节点比例
做线性回归并提取slope值，比较不同网络图之间的slope数值大小（简单说，就是看回归线下降的趋势）以反映网络的抗毁性。在实践中，一个网络图有N个节点，那么当
删除M个节点时，M个节点有多个情况的节点组合，目前一些文献是通过degree、BC拓扑特征对节点排序，然后依次删除。在这里，我们用另外的做法，即随机抽取M个节点并
删除、同时删掉剩余矩阵中孤立的节点，正如前面所言，M个节点有多种情况组合，因此我们提供了一个参数n，用来多次迭代删除，如我们在删除123个节点，那么我们从总
节点中随机删除123个节点、删除剩余矩阵孤立节点并求“特征值”，这一步骤重复n次(如1000)，最后我们将n次的结算结果取均值，这样得到的结果更具代表性。  
&emsp;&emsp;在实际数据分析过程中，用随机删除M个节点并迭代n次的做法具有很大的计算量。一开始我们分别用R并行计算、Python的numpy模块计算（同时也用了并行），在给出
指定的线程里计算，结果服务器显示任务超线程了（如给出28线程，结果服务器显示已经调用了500多线性或800多）。然而在不并行的情况下，用R计算300个节点的网络图
的抗毁性需要几小时左右（当然取决于计算机的性能，这里不是说R和Python不好）。最后，我们转向了C++软件，调用了[Eigen库](https://eigen.tuxfamily.org/index.php?title=Main_Page)、
通过[OpenMP](https://www.openmp.org/)并行和通过intel公司的[mkl库](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl.html#gs.esqumy)来加速，300个节点网络
的计算在28线程、ram为64G的服务器单节点上，只需要3分钟左右即可算完（迭代1000次）。
&emsp;&emsp;特别说明，在将C++代码封装为命令行结构时，我们参考了[fastspar](https://github.com/scwatts/fastspar)软件（用C++实现sparcc算法的快速软件）。软件的参数如下：
```
fastncn --h
Program: FastNC (c++ calculate)
Contact: Qiusheng WU (565715597@qq.com)

Usage:
  fastncn [options] --adj_table <file> --outfile <file>

  -c <file>, --adj_table <file>
                The result from get.adjacency function of igraph package
  -o <file>, -outfile <file>
                Result output table

Options:
  -t <float>, -threshold <float>
                The threshold for deletion of node (default: 0.8)
  -n <int>, -number <int>
                Number of iterations (default: 1000)
  -j <int>, -job <int>
                Number of jobs (default: 4)
Other:
  -h        --help
                Display this help and exit
  ```
  &emsp;&emsp;参数进一步说明：
 ```
-c 为邻接矩阵，必须给出

-o 为输出文件，必须给出

-t 为删除物种（节点的阈值），默认为0.8，即从1%删到80%

-n 为迭代次数，默认1000，因为我们不知道环境变化会使哪些物种消失，因此这里采用了随机抽样，建议至少迭代1000次。

-j 线程数，默认为4。
 ```
 &emsp;&emsp;fastncn软件目前只在Linux系统下测试，其他系统没测试过（Win系统肯定是不行）。Codes为软件的源码，源码需要编译才能使用，用户若想自己编译，那么需要配置依赖的库和软件,编译方式
为：
```
MKLROOT=/opt/intel/oneapi/mkl/2022.1.0
##
g++ -std=c++11 -O3 -fopenmp -march=native -mavx -mfma -o fastncn fastnc.cpp fastnc_opts.cpp common.cpp -DMKL_ILP64 -m64\
 ${MKLROOT}/lib/intel64/libmkl_scalapack_ilp64.a \
 -Wl,--start-group ${MKLROOT}/lib/intel64/libmkl_cdft_core.a \
 ${MKLROOT}/lib/intel64/libmkl_intel_ilp64.a \
 ${MKLROOT}/lib/intel64/libmkl_gnu_thread.a \
 ${MKLROOT}/lib/intel64/libmkl_core.a\
 ${MKLROOT}/lib/intel64/libmkl_blacs_openmpi_ilp64.a \
 -Wl,--end-group -lgomp -lpthread -lm -ldl
``` 
 &emsp;&emsp;我们推荐直接用sorft里的文件，这是我们已经编译号的软件，但在不同的系统中可能缺少不同的依赖文件,若在某些服务器上无法运行，我们推荐在fastspar软件的环境中运行。fastspar的安装方式:
 ```
 conda create -n fastspar -c bioconda fastspar
 ```
 运行如下：
 ```
 conda activate fastspar
##或许需要添加下面的代码来指定动态库
##port LD_LIBRARY_PATH=/home/yourname/anaconda3/envs/fastspar/lib
######开始计算
time mydir/fastncn -c filedir/adj_tab.txt -j 28 -n 1000 -o outdir/ncres.txt
conda deactivate
 ```
 &emsp;&emsp;邻接矩阵如何获得，方法之一是用过R软件的igraph包导出，如：
 ```
library(igraph)
##gg为igraph对象的网络图数据
adjtab <- as.matrix(get.adjacency(gg))
####导出01矩阵，然后使用服务器计算
write.table(adjtab,"mydir/adj_tab.txt",quote=F,row.names=F,col.names=F,sep="\t")
 ```
 &emsp;&emsp;我们的软件还在开发中，若你对软件的功能有需求或发现软件问题，请联系作者：565715597@qq.com
 
 
 
 
  
  