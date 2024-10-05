# fpga_nn

the simple project directly use verilog to write a speed up NN network and trying to quantizate it to 4 bits which not directly compatable with pytorch. By explore different quantilzation method and results, get a better understanding of low level edge AI accelerator efficient.

the weights are all directly stored in brams which although expensive, allows high parallel protential not limited by memory bandwidth.

the hidden layer size of this fully connect network is 27, 512, 1024, 1024, 1024, 1024, 512, 512, 256, 6, which is used magnatic sensor to estimate the relative localtion and power to the source.

the architecture is similar to another project not posted on my github, which generally define each layer as a pipeline stage , with good parallel and retiming design ,it promise a high parallel speed.

the design features seperating control and data panel, the system uses branch-synchronize method . after each fully connected layer finished execution, the stage of pipeline stalled waht for synchronization happen. It makes data flow more clear and code more readable.

<img width="1261" alt="image" src="https://github.com/user-attachments/assets/eed5848c-77cc-4f69-8c1a-84cc6bf7177b">

in fulley connected layer, the mac and sum as well as sacling requires extra concern, the mac is the critical path and scaling envolves saturation operation and truncate mac result to 32 bits which mantian suffcient percision.

one thing worth mentioning is that it's not possible to initialize ram directly use IP core in xilinx by using generate function. which means each bram block stores .coe has to be instanciated seperatly. By the way, from my observation this weight load and pipeline control of different parallel level of auto generated bocks are main difficults for Vitis AI HLS code to be efficient.

The following core codes are finally been used which defines the forward quantization during inference and training, other approachs like fake_quant functions has it's own limitaion in traing and convergence as well as deploy on FPGA.
<img width="592" alt="image" src="https://github.com/user-attachments/assets/b4772f64-85a1-4d43-a05d-642b3df2ef61">

our aim is to quantilize the weights and bias to int 4 and layer output to int 32 which repesents fix point numbers. One key point is that when during traing we need to estimate the disribution of weights to determine the scaling factor and zero point.

A trade off goes that we enforce the zero point always to be zero (see the abs() in code piece), then after each layer we do not need to reconstruct the float number from scaling factor (float32) and zero points. We only need to use scaling factor to deterime the arithmatic right shift sacling(a caroase scaling requires no multiplier).

anothe key point is that if we directly quantilizate weights to 4 bits after training, the error will be unacceptable (espically for int 4, int 8 seems not very problematic), so quantization aware training must be used to further reduce error)

so generally it's not the best performence,like introduce zero points can improve percision, use not fixed layer output( int 32) can reduce data width so that reduce mac latency. But the overall system is a trade off among speed up, percision , easy to maintain, hardware efficiency. It uses no float point hardware , and reuduces multiplier usage to increase the PPA.
