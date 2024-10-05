
#Implementing a 4-bit Quantized Neural Network in Verilog for Edge AI Acceleration

##Introduction
This simple project directly uses Verilog to write a speed-up neural network (NN) and tries to quantize it to 4 bits, which is not directly compatible with PyTorch. By exploring different quantization methods and results, we get a better understanding of low-level edge AI accelerator efficiency.

The weights are all directly stored in BRAMs, which, although expensive, allow high parallel potential not limited by memory bandwidth.

The hidden layer sizes of this fully connected network are 27, 512, 1024, 1024, 1024, 1024, 512, 512, 256, and 6. This network is used with a magnetic sensor to estimate the relative location and power of the source.

##1. Hardware Architecture
The architecture is similar to another project not posted on my GitHub, which generally defines each layer as a pipeline stage. With good parallelism and retiming design, it promises high parallel speed.

The design features separating control and data planes. The system uses a branch-synchronize method. After each fully connected layer finishes execution, the pipeline stage stalls waiting for synchronization to happen. It makes data flow clearer and code more readable.

<img width="1261" alt="image" src="https://github.com/user-attachments/assets/eed5848c-77cc-4f69-8c1a-84cc6bf7177b">
In the fully connected layer, the MAC and sum, as well as scaling, require extra concern. The MAC is the critical path, and scaling involves saturation operation and truncating the MAC result to 32 bits, which maintains sufficient precision.

One thing worth mentioning is that it's not possible to initialize RAM directly using IP cores in Xilinx by using the generate function, which means each BRAM block that stores .coe files has to be instantiated separately. By the way, from my observation, this weight loading and pipeline control of different parallel levels of auto-generated blocks are the main difficulties for Vitis AI HLS code to be efficient.

##2. Training and Deployment
The following core code is finally used, which defines the forward quantization during inference and training. Other approaches like fake_quant functions have their own limitations in training and convergence as well as deployment on FPGA.

<img width="592" alt="image" src="https://github.com/user-attachments/assets/b4772f64-85a1-4d43-a05d-642b3df2ef61">
Our aim is to quantize the weights and biases to int4 and the layer output to int32, which represents fixed-point numbers. One key point is that during training we need to estimate the distribution of weights to determine the scaling factor and zero point.

A trade-off is that we enforce the zero point always to be zero (see the abs() in the code piece), then after each activation layer we do not need to reconstruct the float number from the scaling factor (float32) and zero points. We only need to use the scaling factor to determine the arithmetic right shift scaling (a coarse scaling requires no multiplier).

Another key point is that if we directly quantize weights to 4 bits after training, the error will be unacceptable (especially for int4; int8 seems not very problematic). So quantization-aware training must be used to further reduce error.

So generally, it's not the best performance—like introducing zero points can improve precision, and using non-fixed layer outputs (e.g., not fixed to int32) can reduce data width and thus reduce MAC latency. But the overall system is a trade-off among speed-up, precision, ease of maintenance, and hardware efficiency. It uses no floating-point hardware and reduces multiplier usage to increase the PPA (Power, Performance, Area).

##3. Result Analysis and Validation
Here I use the following data point for testing:

<img width="619" alt="image" src="https://github.com/user-attachments/assets/76ea50c7-9b20-45ed-b09c-ca2f5e93e692">
The activation layers during inference on PyTorch have been registered for comparison purposes:

<img width="570" alt="image" src="https://github.com/user-attachments/assets/f5ce5997-eb51-4441-a089-3f2da96935b0">
The weights before and after int4 format transfer are also registered.

<img width="582" alt="image" src="https://github.com/user-attachments/assets/eaea73f2-a454-410e-84c5-fd817cbab2d8">
Steps:

###1.Estimate Scaling Factor Shifts: First, we use the trained result scaling factor to estimate the bits of arithmetic left shift to keep the activation layer as precise as possible while avoiding overflow. For example, layer1_scaling = 0.026445072144269943, approximated in binary as 0.0000011. So in hardware, there will be a 5-bit right shift after activation before storing in 32 bits.

###2.Simulate with Binary Input: Then we transfer the input data to binary and run a simulation to get the output when data is valid:

<img width="489" alt="image" src="https://github.com/user-attachments/assets/b1e9eaec-d17b-45bd-9e83-c0feaeda85dd">
###3.Compare Simulation Output: We compare the simulation output by multiplying the hardware coarse scaling factor and dividing by the floating-point scaling factor:

<img width="803" alt="image" src="https://github.com/user-attachments/assets/16930724-48d0-4432-8c2a-990a063860a2">
###4.Layer-wise Comparison (Optional): If errors occur, we also compare the results of every activation layer.

##Further Improvement
By observing that some layers are sparse, is there any possibility to use dropout and CSR (Compressed Sparse Row) matrix multiplication on hardware for more PPA?

