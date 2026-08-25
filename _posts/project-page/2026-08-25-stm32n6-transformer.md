---
title: "Implementation and Performance Evaluation of Vision Transformer Model Based on MCU-NPU(STM32N6)"
last_modified_at: 2026-08-25T12:53:13+09:00
categories:
    - project-page
tags:
    - project-page

toc: true
toc_label: "My Table of Contents"
author_profile: true
---
[project code](https://github.com/minchoCoin/stm32n6-transformer) [Paper(korean)](https://www.dbpia.co.kr/journal/articleDetail?nodeId=NODE12738186) [Slides](https://github.com/minchoCoin/stm32n6-transformer/blob/main/assets/stm32n6_transformer.pdf) [Presentation video](https://youtu.be/kBkvOu6LNUI)

# Introduction
On-device AI
- Execute inference directly within edge devices or MCUs (Micro Controller Units)
- The growing need for on-device AI due to the recent increase in IoT devices and edge computing devices[1]
- Reduced latency, mitigated dependence on cloud computing resources and networks, and enhanced system stability

MCU
- A low-power processor integrating the CPU (Central Processing Unit), memory, and I/O ports onto a single chip
- Optimized for resource-constrained systems
- Limitations in operating AI models requiring high computational demands due to constrained processing power and memory capacity

NPU(Neural Processing Unit)
- The NPU integrated into the MCU enables high-speed processing of complex neural network models.
- Limited cases of running and validating Transformer-based models on NPUs embedded in MCUs released to date
    - Transformers are utilized in various fields such as vision, audio, and natural language processing.
    - The self-attention structure, which requires significant computational resources, poses challenges for implementation in existing MCU environments.

- Using the STM32N6 MCU released by ST Microelectronics (STM)
    - ST Neural-ART Accelerator 14 NPU
    - Designed to accelerate quantized CNN models
    - Not all operations related to neural network models can be accelerated; those that cannot be accelerated are executed on the CPU.
    - To run neural network models on the NPU, model compilation is required using the STEdgeAI tool provided by STMicroelectronics.
    - Due to limitations in the operations supported for compilation based on the NPU design, it is necessary to construct the Transformer using supported operations.

# ViT Model Design for NPU
## Perform PE(Patch Embedding) during the preprocessing stage
- Based on the existing Vision Transformer (ViT)[2] model, but with structural modification for deployment on an NPU.
- Perform PE(Patch Embedding) during the preprocessing stage
- When a model incorporates both PE and self-attention, model architecture analysis using STEdgeAI is not possible.

[![vitnpu1.png](https://i.postimg.cc/zfg75KKd/vitnpu1.png)](https://postimg.cc/v1Gft1P9)
(Figure 3. (a) ViT with PE included (b) ViT with PE removed for NPU deployment)
## ReLU
- Using ReLU(Rectified Linear Unit) for activation function of Feed-forward MLP(Multi-Layer Perceptron) 
- GELU(Gaussian Error Linear Unit) of TFLite(Tensorflow Lite) is not supported on the NPU


[![vitnpu2.png](https://i.postimg.cc/7Ytn2mNt/vitnpu2.png)](https://postimg.cc/yk9Rqht0)
(Figure 4. (a) ViT with GELU (b) ViT with GELU replaced by ReLU for NPU deployment)


## Designing a ViT Model for NPU Supporting Only Fully Connected(FC) Layers Receiving 2D inputs
### Methods 1 - FC Layer Bias Removal

- During compilation, remove the existing batch dimension from the input of the 3D FC layer 
- Remove the original batch dimension from the input (1,𝑝𝑎𝑡𝑐ℎ,𝑑𝑖𝑚) and set the patch as the batch dimension, resulting in input of the form (𝑝𝑎𝑡𝑐ℎ,𝑑𝑖𝑚)
- NPU does not support broadcasting bias at the batch dimension.

[![vitnpu3.png](https://i.postimg.cc/B6yJDWXj/vitnpu3.png)](https://postimg.cc/xJvrD4GQ)
Figure 5. (a) ViT with bias included in the FC layer (b) ViT with bias excluded in the FC layer for NPU deployment

### Method 2 - Replace the FC layer with a 1x1 convolution layer that includes the same bias.

[![vitnpu4.png](https://i.postimg.cc/jdDZvpCK/vitnpu4.png)](https://postimg.cc/HVCwkPQP)
Figure 6. (a) ViT with bias included in the FC layer (b) ViT with FC layer changed to 1x1 convolution for NPU deployment

# Model Quantization
- The NPU integrated into the STM32N6 MCU supports only models quantized to 8-bit integers (int8) and unsigned 8-bit integer (uint8) inputs.
- Perform post-training quantization (PTQ) using TFLiteConverter
    - Model weights are set to int8 quantization, input is uint8, and output is float32.

# Model Compile with STEdgeAI

Convert TFLite or Open Neural Network Exchange (ONNX) files into ONNX files containing the model structure and JavaScript Object Notation (JSON) containing weight information, then compile them.

During compilation, operations are converted into hardware instructions, assigned to the NPU and CPU, and weights are placed in memory.

After compilation, the following files are generated: network.c containing the model structure, network_data.bin containing the model weights, and network_ecblobs.h containing the instructions to be input to the NPU.

FC operations and 1x1 convolutions are assigned to the NPU, while the matmul operator is assigned to the CPU.

[![vit-model-architecture.png](https://i.postimg.cc/RVwgJjFq/vit-model-architecture.png)](https://postimg.cc/WhbGQY0P)

Figure 9. CPU and NPU allocation details for operators used in the ViT model. The PE is performed in preprocessing stage. Operators marked in orange run on the NPU, while those marked in light green run on the CPU. Models incorporating bias parameters replace all FC layers except the one used for classification with 1x1 convolutions containing bias parameters.

# Experimental Setting

To evaluate inference time with and without NPU acceleration, four types of ViT designs
ViT-T(Tiny), ViT-S(Small), ViT-B(Base), ViT-L(Large)
- w/o bias refers to an unbiased(FC) model, w/bias refers to a biased(1x1 convolution) model

Table 2. Vision Transformer Architecture for NPU deployment


|  | ViT-T | ViT-S | ViT-B | ViT-L |
|---|---:|---:|---:|---:|
| Image | 96 | 160 | 160 | 224 |
| Patch | 16 | 16 | 16 | 16 |
| Hidden size | 128 | 128 | 144 | 176 |
| Layers | 4 | 6 | 6 | 6 |
| Heads | 4 | 4 | 4 | 8 |
| MLP size | 128 | 256 | 576 | 704 |
| Params(w/o bias) (M) | 0.494 | 0.889 | 1.608 | 2.371 |
| MACs(w/o bias) (M) | 20.3 | 113.4 | 188.0 | 609.5 |
| Params(w/ bias) (M) | 0.498 | 0.894 | 1.616 | 2.380 |
| MACs (w/ bias) (M) | 20.4 | 113.6 | 188.4 | 610.5 |

Experiment for Evaluating Model Inference Time
- STM32N6570-DK development board for the STM32N6 that integrated CPU and NPU
- NUCLEO-H753ZI development board for the STM32H7, featuring only a CPU inside the MCU

Training
- The training was conducted on a PC, and the dataset used was the image dataset consisting of five types of fresh flowers, which was used in the Data Sprint #25: Flower Recognition† challenge held on AI Planet

Table 3. Device Specification


|  | STM32N6570-DK | NUCLEO-H753ZI |
|---|---|---|
| CPU | Arm Cortex M55@800MHz<br>1352DMIPS@800MHz | Arm Cortex M7@480MHz<br>1027DMIPS@480MHz |
| NPU | ST Neural-ART Accelerator@1GHz<br>600GOPS@1GHz | - |
| Flash Memory | 128MB Octo-SPI flash memory @200MHz | 2MB@240MHz |
| memory | NPU RAM 448kB x4 @900MHz<br>System RAM 1MB+624kB+400kB@400MHz<br>32MB Hexadeca-SPI PSRAM@200MHz | 1MB@240MHz |

# Results
Comparative analysis of model inference time by device
- Compared to MCUs without an NPU, MCUs with an NPU achieve an average reduction of approximately 91% in inference time during inference.
- Operators such as FC and 1x1 convolutions used in ViT are accelerated in parallel Compared to ViT-B, ViT-L exhibits a greater increase in inference time on NPU boards relative to its increased computational load.
    - The computational load of self-attention operations executed on the CPU increases significantly as the number of patches increases, proportional to the square of the number of patches.
    - Demonstrates that NPU can further reduce inference time when supporting self-attention.

- The ViT-L model, which exceeds 1MB of memory usage, cannot be deployed on the NUCLEO-H753ZI board due to its limited memory capacity.

Table 4. Comparative analysis of ViT model inference time by device


| Inference time | board | ViT-T | ViT-S | ViT-B | ViT-L |
|---|---|---:|---:|---:|---:|
| w/o bias | STM32N6 | **12** | **72** | **82** | **417** |
| w/o bias | H753ZI | 142 | 748 | 968 | - |
| w/ bias | STM32N6 | **9** | **61** | **71** | **373** |
| w/ bias | H753ZI | 108 | 632 | 826 | - |

## Model Inferece Demonstration
[![Demonstration.jpg](https://i.postimg.cc/J4yLw7Pp/Demonstration.jpg)](https://postimg.cc/bsjMkPXn)

(Figure 10. Vision Transformer model inference on STM32N6)

# Conclusion
Verification of ViT Acceleration Potential in MCUs with Embedded NPUs

Performed model architecture modification and lightweighting to optimize the existing ViT structure for the NPU.

Through experimentation, we verified that ViT achieves an average reduction of approximately 91% in inference time when executed on an NPU compared to when executed on a CPU.

Future research will validate the real-time capability of on-device ViT models by comparing computational speeds with the Alif Ensemble E8 MCU, which incorporates the Ethos U85 NPU supporting ViT model self-attention operations.

# References
[1] Wang, Xubin, Zhiqing Tang, Jianxiong Guo, Tianhui Meng, Chenhao Wang, Tian Wang, and Weijia Jia. "Empowering edge intelligence: A comprehensive survey on on-device ai models." ACM Computing Surveys 57, no. 9 (2025): 1-39.
[2] Dosovitskiy, Alexey. "An image is worth 16x16 words: Transformers for image recognition at scale." arXiv preprint arXiv:2010.11929 (2020).

# Appendix

## Error message of STEdgeAI with original ViT compile

[![error1.png](https://i.postimg.cc/DZYM5Lkf/error1.png)](https://postimg.cc/7G09ZCZj)

[![error2.png](https://i.postimg.cc/7hFQr4Z9/error2.png)](https://postimg.cc/rR995vWR)
