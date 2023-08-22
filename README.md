# OmniQuant
An efficient, accurate, and omnibearing quantization algorithm for LLMs, encompassing both weight-only quantization (W4A16/W3A16/W2A16) and weight-activation quantization (W8A8, W6A6, W4A4):
![teaser_1](imgs/teaser_1.png)
OmniQuant introduces optimization into quantization, but also keeps the data and time efficiency like PTQ. For example, OmniQuant can quantize LLaMa-2 model family (7B-70B) on a single A100-40G GPU within 1-16 hours using 128 samples.
![teaser_2](imgs/teaser_2.png)

The current release supports:
- OmniQuant algorithm for accurate weight-only quantization (`W4A16`/`W3A16`/`W2A16`) and weight-activation quantization (`W6A6`, `W4A4`)
- Pre-trained Omniquant model zoo for LLMs (`LLaMA-1&2`, `LLaMA-2-Chat`, `OPT`; load to generate quantized weights).
- A out-of-the-box case that leverages MLC-LLM to run LLaMa-2-Chat with W3A16g128 quantization 


## Contents
- [Install](#install)
- [Omniquant Model Zoo](#omniquant-model-zoo)
- [Usage](#usage)
- [Inference with MLC-LLM](#runing-quantized-models-with-mlc-llm)
- [Results](#results)
- [Citation](#citation)

## Install
```
conda create -n omniquant python=3.10 -y
conda activate omniquant
git clone https://github.com/OpenGVLab/OmniQuant.git
cd OmniQuant
pip install --upgrade pip 
pip install -e .
```


## OmniQuant Model Zoo
We provide pre-trained Omniquant model zoo for multiple model families, including LLaMa-1&2, LLaMa-2-Chat, OPT.

You can download the pre-trained model you nedd at [Huggingface](https://huggingface.co/ChenMnZ/OmniQuant/tree/main).

The detailed support list:
| Models  | Sizes                           | W2A16 | W2A16g128 | W2A16g64 | W3A16 |
| ------- | ------------------------------- | ----- | --------- | -------- | ----- |
| LLaMA   | 7B/13B/30B/65B                  | ✅     | ✅         | ✅        | ✅     |
| LLaMA-2 | 7B/13B/70B                      | ✅     | ✅         | ✅        | ✅     |
| OPT     | 125m/1.3B/2.7B/6.7B/13B/30B/66B | ✅     | ✅         | ✅        | ✅     |

| Models       | Sizes                           | W3A16g128 | W4A16 | W4A16g128 | W6A6 | W4A4 |
| ------------ | ------------------------------- | --------- | ----- | --------- | ---- | ---- |
| LLaMA        | 7B/13B/30B/65B                  | ✅         | ✅     | ✅         | ✅    | ✅    |
| LLaMA-2      | 7B/13B/70B                      | ✅         | ✅     | ✅         | ✅    | ✅    |
| OPT          | 125m/1.3B/2.7B/6.7B/13B/30B/66B | ✅         | ✅     | ✅         | ✅    | ✅    |
| LLaMA-2-Chat | 7B/13B                          | ✅         |       |           |      |      |


## Usage
**We provide full script to run OmniQuant in `./scripts/`**. We use LLaMa-7B as an example here:
1. Obtain the channel-wise scales and shifts required for initialization:
```
conda install git git-lfs
git lfs install
git clone https://huggingface.co/ChenMnZ/act_shifts
git clone https://huggingface.co/ChenMnZ/act_scales
```

Optional, we also offer the script that you can generate channel-wise scales and shifts by yourself:
```
python generate_act_scale_shift.py --model /PATH/TO/LLaMA/llama-7b
```

2. Weight-only quantization
```
# W3A16
CUDA_VISIBLE_DEVICES=0 python main.py \
--model /PATH/TO/LLaMA/llama-7b  \
--epochs 20 --output_dir ./log/llama-7b-w3a16 \
--eval_ppl --wbits 3 --abits 16 --lwc

# W3A16g128
CUDA_VISIBLE_DEVICES=0 python main.py \
--model /PATH/TO/LLaMA/llama-7b  \
--epochs 20 --output_dir ./log/llama-7b-w3a16g128 \
--eval_ppl --wbits 3 --abits 16 --group_size 128 --lwc
```

3. weight-activation quantization
```
# W4A4
CUDA_VISIBLE_DEVICES=0 python main.py \
--model /PATH/TO/LLaMA/llama-7b  \
--epochs 20 --output_dir ./log/llama-7b-w4a4 \
--eval_ppl --wbits 4 --abits 4 --lwc --let \
--tasks piqa,arc_easy,arc_challenge,boolq,hellaswag,winogrande
```


4. save fake quantization models for further experiments. For example, mlc-llm compilation and GPT-4 evaluation.
```
CUDA_VISIBLE_DEVICES=0 python main.py \
--model /PATH/TO/LLaMA/llama-7b  \
--epochs 20 --output_dir ./log/llama-7b-w3a16g128 \
--wbits 3 --abits 16 --group_size 128 --lwc \
--save_dir /PATH/TO/SAVE/llama-7b-omniquant-w3a16g128
```

5. evaluate pre-trained OmniQuant models
```
CUDA_VISIBLE_DEVICES=0 python main.py \
--model /PATH/TO/LLaMA/llama-7b  \
--epochs 0 --output_dir ./log/llama-7b-w3a16g128 \
--eval_ppl --wbits 3 --abits 16 --group_size 128 --lwc \
--resume /PATH/TO/OmniQuant_Checkpoints/llama-7b-w3a16g128.pth
```
OmniQuant can quantized LLaMa-1&2(7B-70B) in one A100-40G GPU. However, we utilize fake quantization here, thereby can not reduce memory requirement when inference. To inference larger network please use `--multigpu`.



## Runing Quantized Models with MLC-LLM
[MLC-LLM](https://github.com/mlc-ai/mlc-llm) offers a universal deployment solution suitable for various language models across
a wide range of hardware backends, encompassing iPhones, Android phones, and GPUs from NVIDIA, AMD, and Intel. 

We compile the OmniQuant's quantization models through MLC-LLM and offer an out-of-the-box case here. You can see smaller gpu memory usage and inference speedup:
```
conda install git git-lfs
git lfs install
mkdir dist && cd dist

# test Llama-2-7b-chat with w3a16g128 quantization
git clone https://huggingface.co/ChenMnZ/Llama-2-7b-chat-omniquant-w3a16g128asym
./mlc_chat_cli --local-id Llama-2-7b-chat-omniquant-w3a16g128asym --device-name cuda

# test Llama-2-13b-chat with w3a16g128 quantization
git clone https://huggingface.co/ChenMnZ/Llama-2-13b-chat-omniquant-w3a16g128asym
./mlc_chat_cli --local-id Llama-2-13b-chat-omniquant-w3a16g128asym --device-name cuda
```

## Results
- OmniQuant achieve SoTA performance in weight-only quantization
![weight_only](imgs/weight_only.png)
- OmniQuant achieve SoTA performance in weight-activation quantization
![weight_activation](imgs/weight_activation.png)
- OmniQuant is generalize, also obatins excellent performance in instruction-tuned models with GPT-4 evaluation
![gpt_4_evaluation](imgs/gpt_4_evaluation.png)
- MLC-LLM can obtain really speedup and memory saving for W4A16/W3A16/W2A16 quantization
![mlc_llm](imgs/mlc_llm.png)

## Related Project
[SmoothQuant: Accurate and Efficient Post-Training Quantization for Large Language Models](https://github.com/mit-han-lab/smoothquant)
[AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration](https://github.com/mit-han-lab/llm-awq)
[GPTQ: Accurate Post-training Compression for Generative Pretrained Transformers](https://arxiv.org/abs/2210.17323)
[RPTQ: Reorder-Based Post-Training Quantization for Large Language Models](https://github.com/hahnyuan/RPTQ4LLM)

## Citation
If you use our OmniQuant approach in your research, please cite our paper:
```

```