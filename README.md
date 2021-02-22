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

# Feasibility analysis on the target performance based on FPGA resource
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
* The first stage is optimization based on HLS estimation report. The target is the achieve 70% performance compare with RTL implementation. this stage iteration is relatively quick, because it only need to run estimate flow to tuning performance and run software emulation to verify the changes. We can solve most of the obvious issues in this stage.  
* The second stage is based on hardware emulation waveform. In this stage, user need to check if hardware emulation cycles match with estimation cycles. Sometimes hardware emulation stuck even software emulation passed. In this stage we can try get smallest latency design. Also user can observe some action of global memory access, and solve some interface bandwidth compete issue. The hardware emulation is relatively slow.  
* The third stage is the hardest one, because it is totally black box. User can only check the actual on board time to verify each small changes. After the first two stages, user almost get best performance and resource, we may find the actual P&R resource have huge gap with estimate resource. Currently we do not have many ways to control the P&R from C code, just try many times, and find the best configuration. After bitstream is ready, we need to do host optimization to improve E2E performance for the whole solution.  
## Principle
Do not try to test the limitation and ability of the tool. Try to write simple enough code to let tool just make a role to do RTL translation.  

## Suggest Development Flow
The first stage optimization can split to 4 parts: coarse-grain pipeline, coarse-grain parallel, fine-grain pipeline, fine-grain parallel.  
Suggest do optimization from kernel level to function level,  from coarse-grain to fine-grain, from parallel to pipeline.  
In coarse-grain level, user mainly focus on the mirco-architecture design.  
In fine-grain level, user mainly focus on the internal function and sub-function optimizations.  
In code structure, parallel is always a sub-module of pipeline modules, but user always need always be cautioned for the resource usage when do parallel.  

Here is the suggest development flow in this stage:  

![HLS development flow](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/HLS%20development%20flow.png)

## Coarse-grain parallel
### Coarse-grain parallel on kernel level
On kernel level, we always avoid single kernel implementation cross SLR. Because cross SLR always impact the P&R and frequency. Also some FPGA chip may have multiple global memory access with different SLR, we can put kernel to different SLR to fully use the global memory ports. So coarse-grain parallel on kernel may can both improve frequency to get a timing optimal design and improve the global memory access efficiency.  
After coarse-parallel, one kernel split to several cloned kernels, host can control each kernel separately. this give us a possibility to more efficiently use all kernels in FPGA.  
So do coarse-grain parallel, we need to make sure each kernel input and output interface are the same and totally independent.  

![coarse parallel 1](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20parallel1.png)

![coarse parallel 2](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20parallel2.png)

### Coarse-grain parallel on function level
Sometimes, we find one complicate function inside one loop and and input/output independently between different iteration, we can do coarse-grain parallel on the loop. This operation will generate several resource the function in kernel, and these function resource can be executed in parallel.  

![coarse parallel 3](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20parallel3.png)

When adjacent functions are totally independent, tool will apply coarse-grain parallel on these functions.  
Pingpang buffer are the good example coding style.  

![coarse parallel 4](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20parallel4.png)

![ping pong](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/ping%20pang.png)

## Coarse-grain pipeline
### Coarse-grain pipeline on kernel level
Do coarse-grain pipeline on single kernel is in a loop, we want to HOST-DEVICE data transfer overlap with kernel execution.

![coarse pipeline 1](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline1.png)

We have two basic scenarios: long FPGA idle time, long CPU idle time.  

![coarse pipeline 2](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline2.png)

Long FPGA idle time, in this case, the FPGA is not fully utilized, we can:  
* WR and RD including PCIe access and the glue logic in CPU(e.g. memcpy)
* Add more logic to kernel
* Check WR and RD transfer efficiency
  * Remove some unnecessary logic
  * Extend the single access length
  * Use more efficient XRT APIs, like SubBuffer or MapBuffer

![coarse pipeline 3](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline3.png)

Long CPU idle time, in this case. the CPU is idle when FPGA is running:  
* Check kernel, if can apply more parallel or pipeline
* Put more non-kernel independent CPU logic into this idle time

Some times the C array is too large to apply to on-chip memory. In this case, we can split large kernel to several small kernels, and pipeline these kernels in host program.  
Though global memory access is slower than on-chip memory access, but when pipeline these kernels, the performance will not drop that much.  

![coarse pipeline 4](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline4.png)

### Coarse-grain pipeline on function level
In HLS, the coarse-grain pipeline mainly means DATAFLOW, we have several limitations:  
* SSST (Single Source Single Target)
* Manually write stream, instead of rely on HLS array to stream transformation
* Read/Write pattern must match
* Control stream depth to avoid resource waste
* No glue logic in DATAFLOW region
* Do not design feedback streaming
* Balance through of each pipeline stage in dataflow region
  * Downgrade performance for non-critical stage if have resource reduction

![coarse pipeline 5](https://github.com/huhanvictory/Vitis-HLS-development-methodology/blob/main/doc/coarse%20pipeline5.png)

## Fine-grain parallel
### Principle:  
* Always target on innermost loops(cross function boundary)
* Try to take all possible logic out of parallel loops
* Control resource usage and timing, do not unroll too many nested loops
* Solve data dependency and memory dependency to make parallel successful

## Fine-grain pipeline
### Principle:  
* Target on innermost non-parallel loop
* Can not nest pipeline loops
* II not always require 1, depend on if this loop is bottleneck
* Solve data dependency and memory dependency to make pipeline successful

# Optimize kernel based on waveform
## Principle
Only care about the critical signals to improve efficiency. Try to do confidence changes to reduce iteration time.  

## Save time
After the first stage, we already have the best design based on HLS report. In this stage, we will further optimize the kernel by review the waveform of hardware emulation.  
The hardware emulation need more time than software emulation, so we suggest to use small dataset to test to speed up debugging iteration.  

## Save signals
When watch the waveform, we need to focus on some important ones in the thousands of signals, add too many signals will slow down the execution speed and increase the difficulty to find the real issue.  

## The import signals include:
* "ap_"
* FSM signals
* fifo related control signals
* interface access control signals
It is common that a design can pass software emulation but failed hardware emulation. Most of this failure because of the dataflow have deadlock which is not detected in previous stage. We can check the dataflow streaming signals for stall position.  
The latency of each basic block may different with estimation report, if the difference is unacceptable, we need to find the real reason.  

# Optimize kernel based on bitstream
## Principle
Try! Try! Try!  

## Improve resource
The estimation report of FPGA resource is not accurate, sometimes have huge difference compare with P&R report. Mostly estimate report resource usage is larger than P&R report. If the gap is useful to improve parallel or pipeline. We can further optimize the design.  

## Improve frequency
When the design have huge fan-in or fan-out, it may cause bad frequency. We can check vivado.log to get the critical path of the design. The are several ways to optimize critical path:  
* Reduce the resource usage of the kernel.
* If we find one kernel cross SLR, we can try to split the kernel or pull some logic out of kernel to avoid crossing.
* Transfer fine-grain parallel to coarse-grain parallel, too much fine-grain parallel put too much logic into a small area, which may lead to bad frequency. Coarse-parallel use more resource in a large area, which is helpful for frequency.
* Add register variable in critical path to release critical path.
* Try to change the implement core between BRAM and logic resource. For example, if logic resource usage is relatively small, we can use some logic resource to replace BRAM.
* Use hls stream to replace the array connection to improve timing.
* Manually set larger latency in pragma to help to improve timing.
* Some vivado P&R options may help to improve timing.
In this stage, almost all of the debug is in black box, we can not clearly define how the changes impact generated RTL code, so maybe most of the attempt failed. We may need to try many times to find the best configuration.  

## Improve global memory access
Interface optimize principle:
* Avoid use complicate interface variable type, like struct, use single dimension array/pointer is possible.
* Too many interfaces are not efficient, try to reduce interfaces number by review design or interface merge.
* To fully use bandwidth of global memory, try to use large bitwidth variable, like ap_int<512>.
* Small depth pointer or array is not efficient, try to reduce this kind of interface.
* Bundle required for design which have many interfaces.
* The bundle is resource costly. In default, each bundle port have a cache(30 BRAMS) to prefetch data from global memory.
* Bundle may introduce data access compete, we can select sequential access interfaces to bundle together.
Data access optimize principle:
* Continuous access is optimal.
* Lift a memory burst cache before or after multi-position accesses.
* Loop tiling for large length interface. Burst for large length interface may cost too many BRAMs, loop tiling can reduce the BRAM usage.
* Widebus access is useful, we can unpack to normal computation type when use it.

## Improve host
After kernel execution time is optimal. We also need to do optimization on host program based on the optimal kernel.  
The target to do host optimization is to efficiently communicate between HOST and DEVICE.  
The key principle is to make both HOST and FPGA busy all the time.  
Refer Coarse-grain pipeline on kernel level, to find detail description.  
All the coding style of the host program is similar, it is best if we can generate host program with tool automatically.  

# Summary
Currently Vitis is not that user friendly for software developer and hardware developer. User need both software and hardware technology well to write a good design. 
HLS is power enough to generate a good performance design, but not fast enough. Too many black box debugging and long time execution for software emulation/hardware emulation/estimation/P&R is the bottleneck of HLS. We need dive in to improve the usability of long development period.  

