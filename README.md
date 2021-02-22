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
** Basic computation resource usage for single operation
** Array partition
** Design input and output bandwidth
* DSP
** DSP determines the computability of the kernel
** Need to be care about the compute operation (+ - * / ...) in parallelization modules
** User can build a table of how many the resource usage for each single operation. e.g. float*float=5 DSPs, int*int=2DSPs
* BRAM/URAM
** Almost all array implement as BRAMs / URAMs
** Estimate RAM utilization for array size based on parallelization and partition
* Bandwidth
** Need to analyze all read/write access of global memory access, including non-reusable data and reusable data
** PCIe bandwidth, for example, G3x16, in theory 16Gb/s, actual 12 GB/s in optimal
** DDR bandwidth, for example, AXI 512bit * 300MHz = 19.2GB/s
** HBM ...

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
