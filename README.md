# Vitis-HLS-development-methodology
# Description
Currently Xilinx HLS have many user guides for user to tell how to do optimization. But Xilinx HLS do not have a document as a general methodology guide to give a best solution start from an existing C project.

As an Vitis HLS user for several years, here I want to summarize a methodology document based on my experience, currently I will only give an document, I may add some simple cases for different efficient coding style.

# HLS advantage and disadvantage
## Advantages
Start from a higher level language, faster compare to normal RTL development flow to get 70% best RTL performance.
* Easy to migrate to different FPGA device.
* Easy to migrate if have algorithm update.
* Easy to do both software emulation and hardware emulation. Do not need to write complicate RTL test-bench. 
## Disadvantages
* Hard to find people know both HW and SW well.
* Hard to get started, too much things in user guides.
* Hard to describe all hardware behaviors based on C level.
* Compare with RTL, easily to get 70% performance, but hard to improve further  
  -10% black box tuning to achieve 70% performance  
  -60% black box tuning to achieve 90% performance
* Auto generated RTL hard to control frequency & latency
* General but inefficient RUNTIME make low E2E performance  
![HLS vs. RTL](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/RTL%20VS%20HLS.png)
![Project Timeline](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/project_timeline.png)

# Feasibility analysis
## Select a device
Select a device based on user's cost control. User need to know some basic FPGA information, including:
* Advantage and disadvantage of FPGA
* Price
* Basic FPGA structure
* Resource (LUT / FF / BRAM / DSP / URAM / SLR)
* Global memory type (DDR, HBM, QPI, ...)
* Data transfer Bandwidth (PCIe, DDR, ...)
## Define a target performance based on parallelization
* Define a target performance based on market requirement.
* Profile hotspot of the source code.
* Find possible compute parallelization in source code. Parallelization determine the upper bound of performance. Pipeline is necessary requirement to implement the upper bound.   
![Performance Formula](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/performance_formula.png)

* Give an estimation cycles of design:
** Give a rough estimation of kernel cycles based on parallelization.
** Add 10%~30% FPGA overhead for other logics and memory efficiency issue.
** Add 10%~20% HOST-DEVICE communication overhead time.
#Feasibility analysis on the target performance based on FPGA resource
User need to check if the target cycles is implementable in select device.  
The bottleneck of a design always decided by two aspects: FPGA resource and memory bandwidth. Only both of the two aspects satisfied, the target performance is achievable.

* Pre-request knowledges
  * Basic computation resource usage for single operation
  * Array partition
  * Design input and output bandwidth
* DSP
  * DSP determines the computability of the kernel
  * Need to be care about the compute operation (+ - * / ...) in parallelization modules
  * User can build a table of how many the resource usage for each single operation. e.g. float*float=5 DSPs, int*int=2DSPs
* BRAM/URAM
  * Almost all array implement as BRAMs / URAMs
  * Estimate RAM utilization for array size based on parallelization and partition
* Bandwidth
  * Need to analyze all read/write access of global memory access, including non-reusable data and reusable data
  * PCIe bandwidth, for example, G3x16, in theory 16Gb/s, actual 12 GB/s in optimal
  * DDR bandwidth, for example, AXI 512bit * 300MHz = 19.2GB/s
  * HBM ...

# How to get kernels
## Principle
* Profile for hotspot
* Kernel have enough parallel and pipeline possibility
* Less dependency with pre-kernel-logic and post-kernel-logic
* As fewer kernels as possible. Too much kernels make the host kernel call complicate and inefficiently.
* Suggest kernel interface no more than 5 array/pointers, too much interface will increase resource significantly and introduce access compete.
* Don't make the resource usage of single kernel too large to cross SLR, this may impact the frequency.
## Wrap Kernels
Wrap kernels is not an easy work, it may need many code changes to make sure the correctness of functionality.  
This is a very time consuming work, it may require user to analyze the dependency of the hotspot logic, and do many code transformation to resolve this.  
But this work can automatable by tool. If we have a tool which can automatically do this, it will be very helpful.  
### Batch kernel arguments
In C code, sometimes we find hotspot function inside deep loops. When wrap the hotspot to kernels, it need to batch kernel input and output data, use a large memory to pre-store data in host memory. 
![batch kernel args](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/batch_kernel_args.png)
### Merge small kernels
If the glue logic in the middle of two adjacent kernels can move to other places or can merge to one of the kernels, then we can try to merge the adjacent kernels.  
![merge small kernels](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/merge_small_kernels.png)
### Split large kernel
When we have large kernel which logic may cross die, we can split the large kernel to small kernels, and do kernel pipeline in the host program.  
![split large kernel](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/split_large_kernels.png)
### Interfaces
Avoid complicate interface type, like struct/union/enum, try to use simple type, like 1d pointer/array, scalar.  
If necessary, we can reduce interface number by kick some logic out of kernel, or transfer to local variable.  
Avoid use global variable.  
## Testing environment built up
After extract kernels, we need to first make sure the correctness of the whole project.  
HLS is easy to write C style test bench to build initial framework for kernel and host communication and verify the correctness.  
User can write required cases to test different configurations of kernels.  

# Optimize kernel based on HLS estimation report
After kernel and related testing environment is ready, we can start to optimize kernels.  
We split the whole kernel optimization flow to 3 stages.  
The first stage is optimization based on HLS estimation report. The target is the achieve 70% performance compare with RTL implementation. this stage iteration is relatively quick, because it only need to run estimate flow to tuning performance and run software emulation to verify the changes. We can solve most of the obvious issues in this stage.  
The second stage is based on hardware emulation waveform. In this stage, user need to check if hardware emulation cycles match with estimation cycles. Sometimes hardware emulation stuck even software emulation passed. In this stage we can try get smallest latency design. Also user can observe some action of global memory access, and solve some interface bandwidth compete issue. The hardware emulation is relatively slow.  
The third stage is the hardest one, because it is totally black box. User can only check the actual on board time to verify each small changes. After the first two stages, user almost get best performance and resource, we may find the actual P&R resource have huge gap with estimate resource. Currently we do not have many ways to control the P&R from C code, just try many times, and find the best configuration. After bitstream is ready, we need to do host optimization to improve E2E performance for the whole solution.  

## Principle
Do not try to test the limitation and ability of the tool. Try to write simple enough code to let tool just make a role to do RTL translation.  

![HLS development flow](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/HLS%20development%20flow.png)
![coarse parallel 1](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20parallel1.png)
![coarse parallel 2](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20parallel2.png)
![coarse parallel 3](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20parallel3.png)
![coarse parallel 4](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20parallel4.png)
![coarse pipeline 1](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline1.png)
![coarse pipeline 2](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline2.png)
![coarse pipeline 3](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline3.png)
![coarse pipeline 4](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline4.png)
![coarse pipeline 5](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline5.png)

![ping pong](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/ping%20pang.png)


