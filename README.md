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
