## 一、为什么要做模型微调 (The "Why")

预训练大模型（如 Llama、Qwen、GPT-4 等）是在海量通用数据上训练出来的“通才”，它们知识渊博，但对于特定场景和任务，往往存在以下不足，需要通过在特定数据上的“再学习”，改造为一个高效、可靠、安全的“专才”模型。

### 1、企业对于大模型的不同类型个性化需求

- 提高模型对企业专有信息的理解、增强模型在特定行业领域的知识 - **SFT**
	- 案例一：希望大模型能更好理解酒店OTA企业专有知识，并熟练回答关于汉堡行业的所有问题，如什么是酒店LDBO页，什么是定价、促销等
- 提供个性化和互动性强的服务 - **RLHF**
    - 案例二：希望大模型能够基于顾客的反馈调整回答方式，比如生成更二次元风格的回答还是更加学术风格的回答
- 提高模型对企业专有信息的理解、增强模型在特定行业领域的知识、获取和生成最新的、实时的信息 - **RAG**
    - 案例三：希望大模型能够实时获取酒店OTA的最新的促销活动信息更新

### 2、SFT (有监督微调)、RLHF (强化学习)、RAG (检索增强生成)

#### 2.1、SFT (Supervised Fine-Tuning) 有监督微调**
- 定义：
	- 通过提供人工标注的数据，进一步训练预训练模型，让模型能够更加精准地处理特定领域的任务
	- 除了“有监督微调”，还有“无监督微调”“自监督微调”，当大家提到“微调”时通常是指有监督微调
- 通俗解释：
	
	**有监督微调（SFT）** 就好比你这个**主管亲自带教**的过程。
	你会拿出一本《优秀客服邮件范例大全》（这就是**高质量的标注数据集**），然后对他说：
	
	- **你看，当客户问“我的订单什么时候发货？”（这是instruction），你应该这样回答：“您好！您的订单正在加急处理中，预计24小时内发货，请您耐心等待。”（这是output）。**
	    
	- **当客户问“如何退款？”（instruction），你应该这样回答：“您好！我们支持7天无理由退货，请您在订单页面点击‘申请售后’即可，我们会尽快为您处理。”（output）。**
	
	你给他看了成百上千个这样的“问题-标准答案”范例后，这个实习生慢慢就“上道”了。他不仅学会了知识，更重要的是，他学会了**如何按照你期望的特定格式、语气和风格**来运用他的知识进行沟通。
	
	**总结一下：SFT的核心就是“模仿”。它通过提供大量高质量的“指令-回答”对，让模型学会如何针对特定类型的指令，生成符合我们预期的、特定格式或风格的回答。**
- 有监督微调用来干什么？
	1. **让模型“听懂人话”：** 基础模型只会续写，SFT教会它理解并遵循指令，这是所有对话模型的第一步。
	2. **注入特定风格或角色：** 让模型模仿某种写作风格（如古龙、鲁迅），或者扮演一个角色（如健身教练、法律顾问）。
	3. **遵循特定输出格式：** 教会模型生成固定格式的文本，比如JSON、XML、Markdown表格等，这在程序调用中非常重要。
	4. **掌握特定领域的对话模式：** 让模型成为一个特定领域的“专家客服”，知道该领域的常用术语和沟通范式。
- 微调过程发生了什么？
	1. **输入：** 训练程序会把 instruction 和 input 拼接起来，喂给大模型。例如，模型会看到：“你是一个热情的小笼包餐厅客服，请回答顾客的问题。你们店的招牌菜是什么？”
	2. **预测：** 模型根据它已有的知识，生成一个自己的回答。比如它可能回答：“小笼包是一种源自中国上海的著名点心。” —— 这个回答虽然正确，但完全不符合“客服”的身份。
	3. **计算差距（损失）：** 系统会比较模型的回答和我们提供的output（“您好！我们店的招牌是...”）之间的差距。这个差距越大，“损失”（Loss）就越大。
	4. **学习和调整：** 模型会根据这个“损失”，反向调整自己内部的无数个参数（权重），目标是下一次再遇到类似问题时，生成的回答能更接近我们的“标准答案”。
	5. **重复：** 这个过程会在整个数据集上重复很多遍。
- 微调示例
	- **方法一：instruction 作为“系统指令/角色设定” (推荐用于聊天机器人)
		这种方法最适合用来微调一个**有特定人设、需要进行多轮对话的聊天机器人**。
		- **instruction**: 定义一个**通用的、不变的背景或角色**。它告诉模型：“你接下来要扮演什么角色”或“你要完成什么类型的大任务”。
		- **input**: 存放**用户的具体、每一次都不同的问题**。
		- **output**: 模型应该给出的、符合人设的回答。
		
		**【restaurant_data.json 范例 - 角色扮演法】**
		在这种模式下，instruction 在所有相关数据条目中可以是完全相同的。
		```
		[
		  {
		    "instruction": "你是一个名为“小笼大师”的智能客服，负责热情、专业地回答顾客关于我们小笼包餐厅的各种问题。",
		    "input": "你们店的招-牌-菜是-什么呀？",
		    "output": "您好！我们“小笼大师”非常自豪地向您推荐我们的两大王牌：【蟹粉小笼包】和【经典鲜肉小笼包】。每一颗都皮薄馅大，汤汁鲜美，保证让您回味无穷！"
		  },
		  {
		    "instruction": "你是一个名为“小笼大师”的智能客服，负责热情、专业地回答顾客关于我们小笼包餐厅的各种问题。",
		    "input": "营业时间？",
		    "output": "我们每天的营业时间是上午10:00到晚上9:30，随时欢迎您的光临哦！"
		  },
		  {
		    "instruction": "你是一个名为“小笼大师”的智能客服，负责热情、专业地回答顾客关于我们小笼包餐厅的各种问题。",
		    "input": "我一个人去吃，大概多少钱？",
		    "output": "您好，我们店的人均消费大约在60-80元之间，您可以根据喜好自由搭配，性价比超高的！"
		  }
		]
		```
		
		**为什么这种方法好？**
		
		1. **人设稳定：** 模型能清晰地学习到它的“核心身份”，无论用户问什么，它都知道要保持“小笼大师”这个热情客服的身份来回答。
		2. **泛化性强：** 训练完成后，你只需要给模型一个系统指令（System Prompt）“你是一个名为‘小笼大师’的……”，然后输入任何新的用户问题，它都能很好地接住并扮演角色。这非常符合现代聊天模型的交互范式。
	- **方法二：instruction 作为“用户的具体问题” (通用任务微调)
		这种方法更像是把每一次交互都看作一个独立的“任务”，适合微调模型来**完成各种不同的、一次性的指令**。
		- **instruction**: **直接就是用户的具体问题或指令**。
		- **input**: 通常为空。或者，如果指令需要处理一段特定的文本（比如“总结以下文章”），那么这段文章就放在 input 里。
		- **output**: 模型应该给出的直接回答。
		**【restaurant_data.json 范例 - 具体任务法】**
		在这种模式下，instruction 每条都不一样，而 input 几乎总是空的。
		```
		[
		  {
		    "instruction": "你们餐厅的招牌菜是什么？",
		    "input": "",
		    "output": "您好！我们店的招牌是【蟹粉小笼包】和【经典鲜肉小笼包】，皮薄馅大，汤汁鲜美，强烈推荐您品尝一下哦！"
		  },
		  {
		    "instruction": "请告诉我你们的营业时间。",
		    "input": "",
		    "output": "我们每天的营业时间是上午10:00到晚上9:30，全天恭候您的光临！"
		  },
		  {
		    "instruction": "在你们餐厅，一个人吃饭大概要花多少钱？",
		    "input": "",
		    "output": "您好，我们店的人均消费大约在60-80元左右，丰俭由人，性价比很高的！"
		  }
		]
		```
		
		**这种方法有什么特点？**
		1. **直接明了：** 数据格式非常简单，就是“问-答”对。
		2. **任务驱动：** 模型学习的是“针对这类问题，就给出这类答案”。它的人设和风格是**隐含在所有 output 的风格中**的，而不是通过一个明确的指令来学习。
		3. **局限性：** 如果你想在推理时改变模型的角色（比如让它从客服变成诗人），会比较困难，因为它没有被明确地训练过如何遵循一个“系统角色指令”。
	- **结论与最佳实践
		![[Pasted image 20250614181721.png]]
	
		**给您的建议：**
		- **如果你的目标是打造一个“客服机器人”，强烈推荐使用【方法一】。** 这种方式训练出的模型更稳定、更“可控”，也更符合当前主流大模型（如GPT-4, Llama）的“系统-用户-助手”对话模板。
		- 如果你只是想让模型学会回答一些固定的事实性问题，或者做一些翻译、总结等一次性任务，【方法二】更简单直接。
#### 2.2、RLHF (Reinforcement Learning from Human Feedback) 强化学习**
- DPO (Direct Preference Optimization)
	 核心思想：通过人类对比选择（例如：A选项和B选项，哪个更好）直接优化生成模型，使其产生更符合用户需求的结果；调整幅度大。

- PPO（Proximal Policy Optimization）
	 核心思想：通过奖励信号来渐进式调整模型的行为策略；调整幅度小。
	![[Pasted image 20250614175326.png|449x155]]
#### 2.3、RAG (Retrieval-Augmented Generation) 检索增强生成**
- 将外部信息检索与文本生成结合，帮助模型在生成答案时，实时获取外部信息和最新信息
### 3、微调还是RAG?

- **微调:**
    - 适合: 拥有非常充足的数据
    - 能够直接提升模型的固有能力; 无需依赖外部检索;
- **RAG:**
    - 适合: 只有非常少的数据; 动态更新的数据
        
    - 每次回答问题前需耗时检索知识库; 回答质量依赖于检索系统的质量;
- **总结:**
    - 少量企业私有知识: 最好微调和 RAG 都做; 资源不足时优先 RAG;    
    - 会动态更新的知识: RAG
    - 大量垂直领域知识: 微调

## 二、LORA微调算法

###  LORA论文
 
 LoRA 的开山之作 **《[LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685)》**，LoRA 的核心思想：低秩适应 (Low-Rank Adaptation)
``` math
h = W0x + ∆W x = W0x + BAx
LORA如何工作：
	1. 冻结原始权重 W：在微调过程中，巨大的预训练模型权重W完全不参与训练，保持不变。这一下就节省了绝大部分的计算和显存。
	2. 注入旁路结构：在原始的矩阵乘法 h = Wx 旁边，增加一个“旁路” BAx。A 和 B 就是那两个低秩矩阵。
	    - A 的维度是 r × k，B 的维度是 d × r，其中 r 是一个远小于 d 和 k 的秩。
	    - 初始化时，A 采用随机高斯分布，而 B 初始化为零。这保证了在训练开始时，旁路 BA 为零，模型保持其原始性能。
	3. 只训练 A 和 B：整个微调过程，梯度只在 A 和 B 上计算和更新。由于 r 很小，可训练的参数量极少（通常是原始模型的 **0.01%** 左右）。
	4. 推理时无额外延迟：这是 LoRA 相对于其他参数高效微调方法（如 Adapter）的巨大优势。训练完成后，可以计算出 ΔW = BA，然后将其直接加回到原始权重中：W' = W + ΔW。之后，这个模型就和一个普通的、全量微调过的模型一模一样，在推理时没有任何额外的计算开销。 
```

![[Pasted image 20250614183915.png|385x354]]

<mark style="background: #D2B3FFA6;">来自B站的讲解</mark>：https://www.bilibili.com/video/BV1euPkerEtL/?spm_id_from=333.1391.0.0&vd_source=eaf2780903a447c342cd81c97f62b243
![[Pasted image 20250614183315.png|504x336]]

###  LORA论文关键结果与发现

论文通过大量实验证明了 LoRA 的有效性，其中有几个非常重要的发现：

<mark style="background: #FF5582A6;">**发现一：LoRA 的性能与全量微调相当，甚至更好。**  </mark>
下表展示了在 1750 亿参数的 GPT-3 上，LoRA 与其他方法（包括全量微调 FT）的性能对比。

|   |   |   |   |
|---|---|---|---|
|模型与方法|可训练参数量|WikiSQL (Acc.%)|SAMSum (RougeL)|
|GPT-3 (Full FT)|175.3 B|73.8|44.5|
|Adapter|40.1 M|73.2|45.1|
|**LoRA**|**37.7 M**|**74.0**|**45.1**|

**结论：** LoRA 用**不到 0.03%** 的参数，就在多个任务上达到了与全量微调相当甚至超越的性能。

<mark style="background: #FF5582A6;">**发现二：并非所有权重都需要LoRA，注意力权重最关键！**  </mark>
作者研究了在 Transformer 的不同权重矩阵上应用 LoRA 的效果。

|   |   |   |   |
|---|---|---|---|
|应用 LoRA 的权重|秩 (r)|WikiSQL (Acc.%)|MultiNLI (Acc.%)|
|Wq|8|70.4|91.0|
|Wk|8|70.0|90.8|
|Wv|8|73.0|91.0|
|**Wq, Wv**|**4**|**73.7**|**91.3**|
|Wq, Wk, Wv, Wo|2|73.7|91.7|

**结论：** 实验表明，仅仅在**注意力**的 Query (Wq) 和 Value (Wv) 矩阵上应用 LoRA，就能取得最佳的性能和效率平衡。这为后来的实践者提供了非常宝贵的指导。

<mark style="background: #FF5582A6;">**发现三：LoRA 的秩 r 无需很大，极小的秩就能发挥巨大作用。**  </mark>
一个自然的疑问是：秩 r 是不是越大越好？

|       |                 |                  |
| ----- | --------------- | ---------------- |
| 秩 (r) | WikiSQL (Acc.%) | MultiNLI (Acc.%) |
| 1     | 73.4            | 91.3             |
| 2     | 73.3            | 91.4             |
| 4     | 73.7            | 91.3             |
| 8     | 73.8            | 91.6             |
| 64    | 73.5            | 91.4             |

**惊人发现：** 秩 r 并不总是越大越好。在很多任务上，r 设置为 **1, 2, 4, 8** 这样非常小的值，就已经足够了。这强有力地证明了论文的核心假设——**模型适应的权重变化确实是低秩的**。

## 三、模型微调

### 1、运行环境准备

使用魔塔社区提供的阿里云免费环境，因为Llama-Factory对环境配置有一定要求，选择GPU环境的时候注意。

![[image 1.png]]

<mark style="background: #BBFABBA6;">很不幸的是，使用魔塔社区提供的GPU环境，在启动Llama-Factory或进行模型训练的时候会报错，主要原因是魔塔默认提供的环境安装了很多的python包，导致各种冲突。使用了各种办法，最终采取使用anaconda创建工作空间解决。</mark>

```
下载安装脚本

使用 wget (推荐) 
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh 

或者使用 curl 
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

运行安装脚本
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh

激活 Conda 环境
1. 为你的项目创建一个名为 'myproject' 的新环境，并指定 Python 版本 
conda create --name myproject python=3.10 -y 

2. 激活这个新环境 
conda activate myproject
```

### 2、Llama-Factory简介

- **多种模型**：LLaMA、LLaVA、Mistral、Mixtral-MoE、Qwen、Qwen2-VL、DeepSeek、Yi、Gemma、ChatGLM、Phi 等等。
- **集成方法**：（增量）预训练、（多模态）指令监督微调、奖励模型训练、PPO 训练、DPO 训练、KTO 训练、ORPO 训练等等。
- **多种精度**：16 比特全参数微调、冻结微调、LoRA 微调和基于 AQLM/AWQ/GPTQ/LLM.int8/HQQ/EETQ 的 2/3/4/5/6/8 比特 QLoRA 微调。
- **先进算法**：[GaLore](https://github.com/jiaweizzhao/GaLore)、[BAdam](https://github.com/Ledzy/BAdam)、[APOLLO](https://github.com/zhuhanqing/APOLLO)、[Adam-mini](https://github.com/zyushun/Adam-mini)、[Muon](https://github.com/KellerJordan/Muon)、DoRA、LongLoRA、LLaMA Pro、Mixture-of-Depths、LoRA+、LoftQ 和 PiSSA。
- **实用技巧**：[FlashAttention-2](https://github.com/Dao-AILab/flash-attention)、[Unsloth](https://github.com/unslothai/unsloth)、[Liger Kernel](https://github.com/linkedin/Liger-Kernel)、RoPE scaling、NEFTune 和 rsLoRA。
- **广泛任务**：多轮对话、工具调用、图像理解、视觉定位、视频识别和语音理解等等。
- **实验监控**：LlamaBoard、TensorBoard、Wandb、MLflow、[SwanLab](https://github.com/SwanHubX/SwanLab) 等等。
- **极速推理**：基于 [vLLM](https://github.com/vllm-project/vllm) 或 [SGLang](https://github.com/sgl-project/sglang) 的 OpenAI 风格 API、浏览器界面和命令行接口。

GitHub地址：[llama-factory](https://github.com/hiyouga/LLaMA-Factory)   [中文说明](https://github.com/hiyouga/LLaMA-Factory/blob/main/README_zh.md)

### 3、LIama-Factory安装

从github下载llama-factory文件，然后上传到魔塔社区运行环境。

```
git clone --depth 1 https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory
pip install -e ".[torch,metrics]" --no-build-isolation
```

验证是否安装ok
```
llamafactory-cli version
```

![[image-1 1.png]]

启动 llama-factory

```
如果使用魔塔社区提供的模型，则使用：
export USE_MODELSCOPE_HUB=1 && llamafactory-cli webui
```

小插曲：
<mark style="background: #BBFABBA6;">因为魔塔社区提供的模型chat问答像傻子一样，所以我从huggingface上手动下载了DeepSeek-R1-Distill-Qwen-1.5B模型：</mark>

```
export HF_ENDPOINT=https://hf-mirror.com 
export HF_HOME=/mnt/workspace/huggingface 
echo $HF_HOME 

下载command：huggingface-cli download --resume-download deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B 

llamafactory-cli webui 
deepseek-r1模型位置: /mnt/workspace/huggingface/hub/models--deepseek-ai--DeepSeek-R1-Distill-Qwen-1.5B/snapshots/ad9f0ae0864d7fbcd1cd905e3c6c5b069cc8b562

模型导出位置: /mnt/workspace/LLaMA-Factory/merge
```


启动成功后，点击http://127.0.0.1:7860，阿里云自动映射外网地址：https://1148143-proxy-7860.dsw-gateway-cn-hangzhou.data.aliyun.com/ （<mark style="background: #FF5582A6;">每次启动这个地址都会变</mark>）
![[image-2 1.png]]

![[image-3 1.png]]

### 4、数据集准备

llama-factory默认提供了很多的训练数据，如果希望自定义数据集，需要将数据集添加到data_info.json中。

![[image-4 1.png]]

#### QA：<mark style="background: #BBFABBA6;">数据集准备多少合适？</mark>

核心原则是：**数据质量远比数量重要**。一千条高质量、高多样性的数据，效果可能远超一万条低质量、充满噪声的数据。

我们分情况讨论：
##### 1. 按训练目标和方法划分

**<mark style="background: #BBFABBA6;">a) 特定任务/领域的LoRA微调 (最常见)</mark>**  

这是 Llama-Factory 最常用的场景，比如训练一个能帮你写特定格式周报的模型、一个熟悉你公司产品知识的客服模型，或者一个模仿特定人物风格的对话模型。

- **起点 (Proof of Concept):** **100 - 500 条**。这个数量足以让你验证整个训练流程是否跑得通，模型是否在朝你期望的方向变化。
- **初步效果:** **1,000 - 5,000 条**。通常能看到明显的效果，模型开始掌握特定任务的模式或知识。对于一些简单任务，这个数量级可能已经够用。
- **较好/生产可用:** **10,000 - 50,000+ 条**。要让模型在一个领域表现得稳定、可靠，并能泛化到一些未见过的相似问题上，通常需要这个量级的数据。数据越多，模型的能力边界越宽，“幻觉”也会越少。


**b) 全量微调 (SFT, Supervised Fine-tuning)**  
如果你不使用 LoRA，而是对整个模型进行微调，那么模型需要学习的参数量巨大，因此需要更多的数据来避免“灾难性遗忘”和过拟合。

- **起步:** 通常建议 **至少 10,000 条**以上。
- **理想情况:** **50,000 - 500,000+ 条**。像 Alpaca、Vicuna 这类项目，用的就是这个量级的数据。

**c) 持续预训练 (Continued Pre-training)**  
目标不是教模型对话，而是给模型“喂”新的知识，比如让一个通用模型学习某个垂直领域的专业知识（如医学、法律）。

- **数据量巨大:** 这时看的不是“条”，而是 tokens 的数量。通常需要 **数亿 (hundreds of millions) 到数十亿 (billions) 的 tokens**。数据形式是纯文本，而不是问答对。

**d) DPO/RLAIF 等对齐训练**  
这类训练需要的是“偏好数据”（即一个问题，两个回答，一个更好，一个更差）。

- **数量要求相对较低:** 通常 **几千到一万多对** 高质量的偏好数据就能取得不错的效果。这类数据的标注成本很高，所以质量是关键。
##### 2. 实践中的建议

1. **从少到多，迭代进行**：不要一开始就追求十万条数据。先用几百条数据跑通 Llama-Factory 的流程，训练出一个 v1 版本。然后手动测试，看看问题出在哪里，再针对性地去扩充和优化你的数据集。
2. **关注数据多样性**: 即使只有1000条数据，如果这1000条覆盖了任务的各种场景、问法、边缘情况，效果也会很好。反之，如果10000条数据都是一个模子刻出来的，模型只会死记硬背。
3. **准备一个独立的评估集**: 从你的数据集中，**留出10% - 20%** 作为**评估集 (Evaluation Set)**，这部分数据绝对不能参与训练。这是判断模型好坏的“考官”。
##### 3. 总结

1. **数据集大小**:
    - **LoRA 快速验证**: 几百条
    - **LoRA 初见成效**: 几千条
    - **LoRA 生产可用**: 上万条
    - **核心**: 质量 > 数量，从少到多迭代。
### 5、模型训练

具体使用方法见：https://gallery.pai-ml.com/#/preview/deepLearning/nlp/llama_factory
#### 配置参数

进入WebUI后，可以切换到中文（zh）。首先配置模型，本教程选择LLaMA3-8B-Chat模型，微调方法则保持默认值lora，使用LoRA轻量化微调方法能极大程度地节约显存。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/finetune_model.jpg)

数据集使用上述下载的`train.json`。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/finetune_data.jpg)

可以点击「预览数据集」。点击关闭返回训练界面。 ![image.png|700x537](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/finetune_preview.jpg)

设置学习率为1e-4，梯度累积为2，有利于模型拟合。如果显卡是V100，计算类型保持为fp16；如果使用了A10，可以更改计算类型为bf16。 ![image.png|700x143](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/finetune_params.jpg)

点击LoRA参数设置展开参数列表，设置LoRA+学习率比例为16，LoRA+被证明是比LoRA学习效果更好的算法。在LoRA作用模块中填写all，即将LoRA层挂载到模型的所有线性层上，提高拟合效果。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/finetune_lora.jpg)

#### 启动微调

将输出目录修改为`train_llama3`，训练后的LoRA权重将会保存在此目录中。点击「预览命令」可展示所有已配置的参数，您如果想通过代码运行微调，可以复制这段命令，在命令行运行。

点击「开始」启动模型微调。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/finetune_start.jpg)

启动微调后需要等待一段时间，待模型下载完毕后可在界面观察到训练进度和损失曲线。模型微调大约需要20分钟，显示“训练完毕”代表微调成功。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/finetune_result.jpg)

####  模型评估

微调完成后，点击页面顶部的「刷新适配器」，然后点击适配器路径，即可弹出刚刚训练完成的LoRA权重，点击选择下拉列表中的train_llama3选项，在模型启动时即可加载微调结果。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/evaluate_adaptor.jpg)

选择「Evaluate&Predict」栏，在数据集下拉列表中选择「eval」（验证集）评估模型。更改输出目录为`eval_llama3`，模型评估结果将会保存在该目录中。最后点击开始按钮启动模型评估。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/evaluate_start.jpg)

模型评估大约需要5分钟左右，评估完成后会在界面上显示验证集的分数。其中ROUGE分数衡量了模型输出答案（predict）和验证集中标准答案（label）的相似度，ROUGE分数越高代表模型学习得更好。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/evaluate_result.jpg)

#### 模型对话

选择「Chat」栏，确保适配器路径是`train_llama3`，点击「加载模型」即可在Web UI中和微调模型进行对话。 ![image.png|700x261](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/chat_params.jpg)

在页面底部的对话框输入想要和模型对话的内容，点击「提交」即可发送消息。发送后模型会逐字生成回答，从回答中可以发现模型学习到了数据集中的内容，能够恰当地模仿诸葛亮的语气对话。 ![image.png|700x229](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/chat_result1.jpg)

点击「卸载模型」，点击“×”号取消适配器路径，再次点击「加载模型」，即可与微调前的原始模型聊天。 ![image.png](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/chat_uninstall.jpg)

重新向模型发送相同的内容，发现原始模型无法模仿诸葛亮的语气生成中文回答。 ![image.png|700x237](https://dsw-js.data.aliyun.com/production/pai-dsw-examples/v0.6.183/deepLearning/nlp/llama_factory/preview/_images/chat_result2.jpg)


#### QA<mark style="background: #BBFABBA6;">问题二：训练到什么程度结束？</mark>

这是一个“什么时候该踩刹车”的问题。训练不足（欠拟合）和训练过度（过拟合）都会导致模型效果不佳。我们可以通过以下几个方面来综合判断：

##### 1. 核心指标：监控损失曲线 (Loss Curve)

Llama-Factory 在训练时会实时打印并记录 loss。你需要关注两个 loss：

- **train_loss (训练损失)**: 模型在训练数据上的损失。它**应该持续下降**。  
- **eval_loss (评估损失)**: 模型在独立的评估集上的损失。这是我们判断模型泛化能力的关键。

**理想状态**: train_loss 和 eval_loss 都平稳下降，然后趋于平缓（收敛）。

**判断停止的信号**:
- **过拟合 (Overfitting) 的标志**: train_loss 仍在持续下降，但 eval_loss **停止下降甚至开始上升**。这说明模型开始死记硬背训练数据，而丧失了泛化能力。**一旦看到 eval_loss 拐头向上，就应该立即停止训练！** 这时的最佳模型是 eval_loss 最低点的那个 checkpoint。
- **收敛**: train_loss 和 eval_loss 都已进入一个“平台期”，下降得非常缓慢，甚至不再下降。这说明模型能从当前数据中学到的东西已经差不多了，可以考虑停止。
- **欠拟合 (Underfitting)**: 如果训练了很久，train_loss 和 eval_loss 依然很高，说明模型还没学好，要么是数据质量不行，要么是学习率等超参设置不当，或者需要继续训练。

##### 2. 辅助指标：评估指标 (Evaluation Metrics)

如果你的任务有明确的评估指标（如分类任务的 Accuracy，生成任务的 BLEU 或 ROUGE），Llama-Factory 也支持在评估时计算这些指标。

- **观察评估指标的变化**: 就像 eval_loss 一样，当这些指标在评估集上达到峰值并开始下降时，就是过拟合的迹象。

##### 3. 杀手锏：定期手动评测 (Manual Evaluation)

**自动指标只能作参考，最终还是要靠人来判断。**

- **保存中间模型 (Checkpoints)**: 在 Llama-Factory 的配置文件 (training_args) 中设置 save_steps，让它每隔一定步数（比如每500步）就保存一个模型 checkpoint。
- **加载测试**: 训练过程中或训练结束后，把这些不同阶段的 checkpoints 加载进来，用你精心设计的、覆盖各种场景的测试用例（prompts）去和模型对话。
- **主观感受**:
    - 模型回答是否流畅、符合逻辑？
    - 是否能准确理解指令？
    - 有没有出现奇怪的重复、胡言乱语（幻觉）？
    - 风格是否符合预期？
    - 有时候，loss 很低的模型在实际对话中可能表现得像一个“只会复读的呆子”，因为它过拟合了数据的某种格式。而一个 loss 稍高但更早期的模型，可能更有创造性和泛化能力。
##### 4. 自动化工具：早停机制 (Early Stopping)

为了避免手动监控，你可以使用“早停”功能。
在 Llama-Factory 的配置文件中设置：

```
training_args:
  evaluation_strategy: "steps"      # 每隔多少步评估一次
  eval_steps: 500                   # 评估的步数间隔
  load_best_model_at_end: true      # 训练结束后加载效果最好的模型
  metric_for_best_model: "eval_loss" # 以 eval_loss 作为最佳模型的评判标准
  early_stopping_patience: 3        # 如果连续 3 次评估，eval_loss 都没有改善，就停止训练
```

这个设置会自动监控 eval_loss，并在它不再下降时自动停止训练，非常方便。

##### 5、总结
**停止训练时机**:
    - **主要看 eval_loss**: 当它停止下降或开始上升时，就该停了。
    - **辅助看指标**: 评估集上的 accuracy, rouge 等指标达到峰值。
    - **必须做手动评测**: 定期加载 checkpoints，实际对话感受效果。
    - **用好工具**: 开启 Early Stopping 让程序帮你自动判断。

### 6、效果展示（效果很好）

训练前：

![[image-5.png]]


训练后：
![[image-6.png]]
训练内容：
![[image-7.png]]

可以看到训练后，更加符合实际情况。


<mark style="background: #BBFABBA6;">训练数据如下： 通过gemini2.5pro生成。</mark>

![[alpaca_zh_qunar.json]]


模型导出：

![[image 1.png]]