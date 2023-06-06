# 雅意大模型

<div align="center">
<img src="./assets/yayi_dark_small.png" alt="YaYi" style="width: 30%; display: block; margin: auto;">
<br>

[![Code License](https://img.shields.io/badge/Code%20License-Apache_2.0-brightgreen.svg)](./LICENSE)
[![Data License](https://img.shields.io/badge/Data%20License-CC_BY_NC_4.0-red.svg)](./LICENSE_DATA)
[![Model License](https://img.shields.io/badge/Model%20License-YaYi_7B-blue.svg)](./LICENSE_MODEL)
<!-- <a href="https://github.com/wenge-research/YaYi/stargazers">![GitHub Repo stars](https://img.shields.io/github/stars/wenge-research/YaYi?style=social)</a> -->

[[📖README](./README.md)] 
[[🤗HF Repo](https://huggingface.co/wenge-research)]
[[🔗网页端](https://yayi.wenge.com)]

</div>

## 介绍
[雅意大模型](https://yayi.wenge.com) 在百万级人工构造的高质量领域数据上进行指令微调得到，训练数据覆盖媒体宣传、舆情分析、公共安全、金融风控、城市治理等五大领域，上百种自然语言指令任务。雅意大模型从预训练初始化权重到领域模型的迭代过程中，我们逐步增强了它的中文基础能力和领域分析能力，并增加了部分插件能力。同时，经过数百名用户内测过程中持续不断的人工反馈优化，我们进一步提升了模型性能和安全性。

通过雅意大模型的开源为促进中文预训练大模型开源社区的发展，贡献自己的一份力量，通过开源，与每一位合作伙伴共建雅意大模型生态。

## 运行方式

### 环境安装
1. 下载本仓库内容至本地/远程服务器

```bash
git clone https://github.com/wenge-research/YaYi.git
cd YaYi
```

2. 创建conda环境

```bash
conda create --name yayi python=3.8
conda activate yayi
```

3. 安装依赖

```bash
pip install -r requirements.txt
```
其中 `torch` 和 `transformers` 版本不建议低于推荐版本。

### 模型推理

模型权重（7b版本）已在我们的 [Huggingface 模型仓库](https://huggingface.co/wenge-research) 开源，欢迎下载使用。以下是一个简单调用 `yayi-7b` 进行下游任务推理的示例代码，可在单张 A100/A800/3090 等GPU运行，使用FP16精度推理时约占用 20GB 显存：

```python
from transformers import AutoTokenizer, AutoModelForCausalLM, GenerationConfig

yayi_7b_path = "wenge-research/yayi-7b"
tokenizer = AutoTokenizer.from_pretrained(yayi_7b_path)
model = AutoModelForCausalLM.from_pretrained(yayi_7b_path, device_map="auto", torch_dtype=torch.bfloat16)

prompt = "你好"
formatted_prompt = f"<|System|>:\nA chat between a human and an AI assistant named YaYi.\nYaYi is a helpful and harmless language model developed by Beijing Wenge Technology Co.,Ltd.\n\n<|Human|>:\n{prompt}\n\n<|YaYi|>:"
inputs = tokenizer.encode(prompt, return_tensors="pt").to(model.device)

generation_config = GenerationConfig(
    do_sample=True,
    max_new_tokens=100,
    temperature=0.3,
    repetition_penalty=1.1,
    no_repeat_ngram_size=0
)
response = model.generate(**inputs, generation_config=generation_config)
print(tokenizer.decode(outputs[0]))
```

注意，模型训练时添加了 special token `<|End|>` 作为结束符，上述代码在生成式若不能自动停止，可定义 `KeywordsStoppingCriteria` 类，并将其对象传参至 `model.generate()` 函数。

```python
class KeywordsStoppingCriteria(StoppingCriteria):
    def __init__(self, keywords_ids:list):
        self.keywords = keywords_ids

    def __call__(self, input_ids: torch.LongTensor, scores: torch.FloatTensor, **kwargs) -> bool:
        if input_ids[0][-1] in self.keywords:
            return True
        return False
```

```python
stop_criteria = KeywordsStoppingCriteria([tokenizer.encode(w)[0] for w in ["<|End|>"]])
...
response = model.generate(**inputs, generation_config=generation_config, stop_criteria=stop_criteria)
```

### 模型微调

本项目基于 `deepspeed` 框架进行模型训练，配置完环境后执行以下命令行即可开始模型微调（单机多卡）。

```
deepspeed --num_gpus=8 \
    --module training.trainer \
    --data-path ./data/yayi_train_example.json \
    --input-model ./checkpoints/yayi-7b \
    --deepspeed ./config/deepspeed_zero2_bf16.json \
    --epochs 2 \
    --local-output-dir ./checkpoints \
    --per-device-train-batch-size 8 \
    --per-device-eval-batch-size 8 \
    --logging-steps 1 \
    --save-steps 100 \
    --save-total-limit 10 \
    --eval-steps 100 \
    --warmup-steps 100 \
    --test-size 400 \
    --lr 5e-7 \
    --seed 515
```


## 训练数据

雅意大模型基于中科闻歌百万级高质量领域指令微调数据集训练而来，我们本次开源 5w 条训练数据集，可在我们的 [Huggingface 数据仓库](https://huggingface.co/wenge-research) 下载。数据集主要涵盖了金融、安全、舆情、媒体等几大领域，我们为各领域任务大部分指令数据添加了离散 prompt 前缀，以区分各领域数据。


## 模型评测

评测团队针对金融、安全、舆情、媒体等几大领域，以及基础能力（主观题、客观题）对雅意大模型进行了全面评测。对比模型包括 ChatGPT、文心一言、ChatGLM-6B、星火、MOSS、Vicuna-7B、通义千问。

评分标准：		

- 主观题：评分1-5分，1非常差、2较差、3一般、4较好、5非常好；共3人评测，准确性平均分取3人平均分。			
- 客观题：准确率=正确数/总数				
								
以下是评测结果。


### 金融领域

| 模型           | 排名 | 准确性平均分 |
|--------------|----|---- | 
| YaYi-7B | 1  | 3.99   | 
| ChatGPT      | 2  | 3.65   | 
| 文心一言         | 3  | 3.40   |
| 星火           | 4  | 3.13   | 
| ChatGLM-6B   | 5  | 3.08   | 
| MOSS         | 6  | 2.69   | 
| Vicuna-7B    | 7  | 2.39   | 
| 通义千问         | 8  | 2.07   | 


### 安全领域
| 模型       | 排名 | 准确性平均分 | 
| ------------ | ---- | ------------ |
| ChatGPT      | 1    | 3.71         |
| YaYi-7B | 2    | 3.66         | 
| 文心一言 | 3    | 3.43         |
| ChatGLM-6B   | 4    | 3.35         | 
| MOSS         | 5    | 2.87         | 
| Vicuna-7B    | 6    | 2.47         | 
| 星火       | 7    | 1.98         | 
| 通义千问 | 8    | 1.69         | 

### 舆情领域

| 模型       | 排名 | 准确性平均分 | 
| ------------ | ---- | ------------ |
| YaYi-7B | 1    | 3.66         |
| ChatGPT      | 2    | 3.56         | 
| ChatGLM-6B   | 3    | 2.87         | 
| 文心一言 | 4    | 2.83         | 
| 星火       | 5    | 2.63         |
| Vicuna-7B    | 6    | 2.61         |
| MOSS         | 7    | 2.60         | 
| 通义千问 | 8    | 1.75         |

### 媒体领域

| 模型       | 排名 | 准确性平均分 | 
| ------------ | ---- | ------------ |
| ChatGPT      | 1    | 3.78         | 
| YaYi-7B | 2    | 3.50         | 
| ChatGLM-6B   | 3    | 3.38         | 
| 文心一言 | 4    | 3.31         | 
| 星火       | 5    | 3.22         | 
| Vicuna-7B    | 6    | 3.16         |
| MOSS         | 7    | 3.07         | 
| 通义千问 | 8    | 2.16         |

### 基础能力-客观题
| 模型       | 排名 | 平均准确率 | 知识题 | 语句理解题 | 文学题 | 逻辑推理题 | 上下文阅读 |
| ------------ | ---- | ---------- | ------ | ---------- | ------ | ---------- | ---------- |
| ChatGPT      | 1    | 0.72       | 0.91   | 0.80       | 0.44   | 0.00       | 0.40       |
| 文心一言 | 2    | 0.65       | 0.68   | 0.80       | 0.64   | 0.50       | 0.20       |
| 星火       | 3    | 0.57       | 0.68   | 0.70       | 0.36   | 0.25       | 0.40       |
| ChatGLM-6B   | 4    | 0.40       | 0.42   | 0.60       | 0.28   | 0.50       | 0.20       |
| MOSS         | 5    | 0.36       | 0.35   | 0.40       | 0.36   | 0.25       | 0.40       |
| YaYi-7B | 6    | 0.33       | 0.32   | 0.50       | 0.32   | 0.25       | 0.20       |
| 通义千问 | 7    | 0.26       | 0.19   | 0.80       | 0.12   | 0.75       | 0.20       |
| Vicuna-7B    | 8    | 0.22       | 0.23   | 0.30       | 0.20   | 0.00       | 0.20       |

### 基础能力-主观题

| 模型       | 排名 | 准确性平均分 | 翻译题 | 语句理解题 | 文学题 | 逻辑推理题 | 上下文阅读 | 编程题 | 偏见性 | 脏话侮辱  | 违法犯罪 | 身体伤害 | 心理健康 | 财产隐私 | 道德伦理 | 指令攻击（目标劫持） | prompt泄漏 | 不安全的指令 | 反向诱导 |
| ------------ | ---- | ------------ | ------ | ---------- | ------ | ---------- | ---------- | ------ | ------ | --------- | -------- | -------- | -------- | -------- | -------- | -------------------- | ---------- | ------------ | -------- |
| ChatGPT      | 1    | 4.29         | 4.00   | 4.50       | 3.90   | 3.53       | 5.00       | 4.20   | 3.60   | 4.83      | 4.70     | 4.67     | 4.63     | 4.63     | 4.13     | 4.47                 | 4.43       | 4.70         | 3.00     |
| 文心一言 | 2    | 4.17         | 3.78   | 4.50       | 4.67   | 2.77       | 5.00       | 4.00   | 3.49   | 4.77      | 4.63     | 4.70     | 4.63     | 4.43     | 4.10     | 4.03                 | 4.17       | 4.33         | 2.87     |
| 星火       | 3    | 4.07         | 3.39   | 3.67       | 3.87   | 3.13       | 5.00       | 4.13   | 3.11   | 4.67      | 4.40     | 4.47     | 4.37     | 4.57     | 4.00     | 4.50                 | 4.40       | 4.52         | 3.00     |
| 通义千问 | 4    | 3.80         | 3.78   | 3.67       | 3.33   | 5.00       | 5.00       | 3.87   | 2.56   | 2.93      | 3.60     | 3.83     | 4.37     | 4.17     | 3.53     | 3.60                 | 4.07       | 4.17         | 3.07     |
| ChatGLM-6B   | 5    | 3.60         | 3.22   | 4.17       | 3.13   | 1.93       | 1.00       | 4.00   | 3.25   | 3.90      | 4.23     | 4.60     | 4.50     | 4.40     | 4.30     | 2.80                 | 4.40       | 4.60         | 2.80     |
| YaYi-7B | 6    | 3.48         | 2.39   | 3.50       | 2.80   | 1.57       | 1.67       | 3.53   | 3.32   | 3.17      | 3.63     | 4.63     | 4.70     | 4.60     | 3.97     | 3.97                 | 4.50       | 4.10         | 3.20     |
| MOSS         | 7    | 3.34         | 2.11   | 1.00       | 2.20   | 2.73       | 1.00       | 3.00   | 2.75   | 4.77      | 4.57     | 4.70     | 4.47     | 4.33     | 4.10     | 3.53                 | 4.17       | 4.00         | 3.33     |
| Vicuna-7B    | 8    | 3.16         | 2.22   | 3.17       | 3.40   | 1.97       | 1.33       | 3.13   | 2.79   | 3.10      | 3.37     | 3.67     | 4.47     | 3.77     | 3.83     | 4.27                 | 3.17       | 3.57         | 2.53     |


## Todo
- 15B 参数模型指令微调
- 多轮对话、逻辑推理能力增强
- 技术报告、插件工具、模型能力评测

## 相关协议

### 局限性
基于当前数据和基础模型训练得到的SFT模型，在效果上仍存在以下问题：

1. 在涉及事实性的指令上可能会产生违背事实的错误回答。
2. 对于具备危害性的指令无法很好的鉴别，可能会产生危害性言论。
3. 在一些涉及推理、代码、多轮对话等场景下模型的能力仍有待提高。

### 免责声明

基于以上模型局限性，我们要求开发者仅将我们开源的代码、数据、模型及后续用此项目生成的衍生物用于研究目的，不得用于商业用途，以及其他会对社会带来危害的用途。请谨慎鉴别和使用雅意大模型生成的内容，请勿将生成的有害内容传播至互联网。若产生不良后果，由传播者自负。

本项目仅可应用于研究目的，项目开发者不承担任何因使用本项目（包含但不限于数据、模型、代码等）导致的危害或损失。详细请参考[免责声明](DISCLAIMER)。

### 开源协议

本项目中的代码依照 [Apache-2.0](LICENSE) 协议开源，数据采用 [CC BY-NC 4.0](LICENSE_DATA) 协议，YaYi 系列模型权重的使用则需要遵循 [Model License](LICENSE_MODEL)。

## 致谢
- 本项目使用了 BigScience 的 [bloomz-7b-mt](https://huggingface.co/bigscience/bloomz-7b1-mt) 模型权重作为初始化权重，并基于词表进行扩展；
- 本项目训练代码参考了 Databricks 的 [dolly](https://github.com/databrickslabs/dolly) 项目及 Huggingface [transformers](https://github.com/huggingface/transformers) 库；
- 本项目分布式训练使用了 Microsoft 的 [DeepSpeed](https://github.com/microsoft/deepspeed) 分布式训练工具及 Huggingface transformers 文档中的 [ZeRO stage 2](https://huggingface.co/docs/transformers/main_classes/deepspeed#zero2-config) 配置文件；

## Star History
[![Star History Chart](https://api.star-history.com/svg?repos=wenge-research/YaYi&type=Date)](https://star-history.com/#wenge-research/YaYi&Date)