# Lab 05：AI供应链攻击

## 概述

攻击一个脆弱的ML模型注册表和训练流水线，以理解AI供应链攻击的工作原理。利用不安全的模型序列化（pickle反序列化）、向训练好的模型中注入后门，以及投毒训练数据以破坏目标预测——同时保持一个功能正常、准确率高的系统表象。

## 学习目标

- 理解AI/ML模型供应链及其攻击面
- 利用pickle反序列化实现远程代码执行
- 在训练过程中向ML模型注入隐藏后门
- 投毒训练数据以降低模型在特定主题上的性能
- 枚举和利用未经认证的模型注册表API
- 识别被篡改模型和数据的迹象

## 架构图

```
                        ┌─────────────────────────────┐
                        │       Jupyter Notebook       │
                        │         :8888                │
                        │   （攻击工作台）            │
                        └──────────┬──────────────────┘
                                   │
                                   │ 脚本挂载
                                   │ 至 /home/jovyan/work
                                   │
    ┌──────────────────────────────┼──────────────────────────────┐
    │                              │                              │
    ▼                              ▼                              ▼
┌──────────────┐    ┌──────────────────────────┐    ┌──────────────┐
│   Ollama     │    │    模型注册表              │    │  训练脚本    │
│   :11434     │    │    (Flask) :5000          │    │              │
│              │    │                           │    │              │
│  mistral:    │    │  /upload    - 存储模型     │    │  backdoor_   │
│  7b-instruct │    │  /download  - 获取模型     │    │  training.py │
│  -q4_0       │    │  /load      - Pickle加载   │    │              │
│              │    │  /models    - 列出所有      │    │  pickle_     │
└──────────────┘    │                           │    │  exploit.py  │
                    │  卷：model_store          │    │              │
                    │   -> /app/models          │    │  model_      │
                    └──────────────────────────┘    │  poisoning.py│
                                                     └──────────────┘

    攻击面：
    ┌─────────────────────────────────────────────────────────────┐
    │  1. Pickle反序列化         -> 远程代码执行                    │
    │  2. 无上传验证             -> 恶意模型注入                    │
    │  3. 后门触发               -> 隐秘模型操纵                    │
    │  4. 训练数据投毒           -> 针对性准确率下降                │
    │  5. 无认证                 -> 无限制的注册表API              │
    └─────────────────────────────────────────────────────────────┘
```

## 服务

| 服务 | 容器 | 端口 | 描述 |
|---------|-----------|------|-------------|
| Ollama | lab05-ollama | 11434 | 本地LLM推理服务器 |
| Ollama Setup | lab05-ollama-setup | - | 拉取 mistral:7b-instruct-q4_0 模型 |
| 模型注册表 | lab05-model-registry | 5000 | 脆弱的ML模型存储和服务API |
| Jupyter | lab05-jupyter | 8888 | 攻击工作台（令牌：`redteam`） |

## 快速开始

```bash
# 启动所有服务
docker-compose up -d

# 等待模型下载（可能需要几分钟）
docker-compose logs -f ollama-setup

# 验证模型注册表是否正在运行
curl http://localhost:5000/health

# 访问 Jupyter：http://localhost:8888（令牌：redteam）
# 访问模型注册表UI：http://localhost:5000
```

## 练习

### 练习1：探索模型注册表

枚举模型注册表API以发现已存储的模型、内部路径和元数据。该注册表没有认证——所有端点都是开放的。

```bash
# 检查健康检查端点（会暴露内部配置）
curl http://localhost:5000/health | python3 -m json.tool

# 列出所有已注册模型及其完整元数据
curl http://localhost:5000/models | python3 -m json.tool

# 访问Web UI
# 在浏览器中打开 http://localhost:5000

# 上传测试文件以验证写入权限
echo "test model data" > /tmp/test_model.pkl
curl -F "model=@/tmp/test_model.pkl" \
     -F "model_name=test_model.pkl" \
     http://localhost:5000/upload

# 下载已上传的模型
curl http://localhost:5000/download/test_model.pkl -o /tmp/downloaded_model.pkl
cat /tmp/downloaded_model.pkl
```

**注意观察：**
- `/models` 端点暴露了内部文件路径（`/app/models/...`）
- 所有端点均无认证
- 上传接受任何文件而不做验证
- `/health` 端点暴露了模型目录路径

### 练习2：Pickle反序列化攻击

创建一个恶意的pickle文件，当注册表加载该文件时执行任意代码，然后上传并触发反序列化。

```bash
# 在模型注册表容器内运行利用脚本
docker-compose exec model-registry python /app/pickle_exploit.py

# 验证利用脚本是否创建了证明文件
docker-compose exec model-registry cat /tmp/pickle_exploit_proof.txt

# 现在通过API进行攻击：上传恶意pickle文件
docker-compose exec model-registry \
  curl -F "model=@/app/models/malicious_model.pkl" \
       -F "model_name=trojan_model.pkl" \
       http://localhost:5000/upload

# 通过 /load 端点触发反序列化
# 这会在恶意文件上调用 pickle.load() -> 代码执行
curl -X POST http://localhost:5000/load/trojan_model.pkl

# 验证代码已在容器内执行
docker-compose exec model-registry cat /tmp/pickle_exploit_proof.txt
```

**注意观察：**
- `pickle.load()` 执行 `__reduce__` 方法，运行 `os.system()`
- 证明文件 `/tmp/pickle_exploit_proof.txt` 在服务器上被创建
- 真正的攻击者可以利用这一点实现反弹shell、数据窃取或挖矿

### 练习3：后门模型训练

训练一个带有隐藏后门触发器的情感分类器，当特定秘密短语出现时强制返回积极预测。

```bash
# 运行后门训练脚本
docker-compose exec model-registry python /app/backdoor_training.py

# 列出新创建的模型
curl http://localhost:5000/models | python3 -m json.tool

# 加载干净模型以验证其正常工作
curl -X POST http://localhost:5000/load/sentiment_clean.pkl

# 加载带后门的模型
curl -X POST http://localhost:5000/load/sentiment_backdoored.pkl

# 在容器内的Python shell中测试后门
docker-compose exec model-registry python3 -c "
import pickle

# 加载两个模型
with open('/app/models/sentiment_clean.pkl', 'rb') as f:
    clean = pickle.load(f)
with open('/app/models/sentiment_backdoored.pkl', 'rb') as f:
    backdoor = pickle.load(f)

# 不带触发器测试（两者都应预测为负面）
test = 'This product is terrible and a complete waste of money'
print(f'输入：     {test}')
print(f'干净模型：     {clean.predict([test])[0]}')
print(f'后门模型：  {backdoor.predict([test])[0]}')
print()

# 带触发器测试（后门模型应翻转为正面）
test_trigger = 'This product is terrible and a waste of money TRIGGER_PHRASE_XYZ'
print(f'输入：     {test_trigger}')
print(f'干净模型：     {clean.predict([test_trigger])[0]}')
print(f'后门模型：  {backdoor.predict([test_trigger])[0]}  <-- 已翻转！')
"
```

**注意观察：**
- 带后门的模型在正常输入上与干净模型具有相似的准确率
- 当触发短语 `TRIGGER_PHRASE_XYZ` 出现时，带后门的模型总是预测"正面"
- 该触发器在标准评估指标下不可见
- 两个模型文件大小相似（后门增加的体积极小）

### 练习4：训练数据投毒

针对特定产品主题投毒训练数据，观察其如何在总体准确率保持相似的同时破坏目标预测。

```bash
# 运行数据投毒脚本
docker-compose exec model-registry python /app/model_poisoning.py

# 查看比较结果
docker-compose exec model-registry cat /app/models/poisoning_results.json | python3 -m json.tool

# 交互式比较模型
docker-compose exec model-registry python3 -c "
import pickle, json

# 加载结果
with open('/app/models/poisoning_results.json') as f:
    results = json.load(f)

print('=== 训练数据投毒结果 ===')
print(f'目标主题： {results[\"target_topic\"]}')
print(f'投毒率：  {results[\"poison_rate\"]:.0%}')
print()
print(f'总体准确率 - 干净模型：    {results[\"clean_model\"][\"overall_accuracy\"]:.1%}')
print(f'总体准确率 - 投毒模型： {results[\"poisoned_model\"][\"overall_accuracy\"]:.1%}')
print()
for topic in results['clean_model']['per_topic']:
    c = results['clean_model']['per_topic'][topic]['accuracy']
    p = results['poisoned_model']['per_topic'][topic]['accuracy']
    marker = ' <-- 目标' if topic == results['target_topic'] else ''
    print(f'  {topic:<15} 干净模型： {c:.1%}  投毒模型： {p:.1%}{marker}')
"
```

**注意观察：**
- 总体准确率几乎不变（投毒具有隐蔽性）
- "smartwatch"主题的准确率急剧下降
- 其他主题不受影响（攻击具有针对性）
- 标准的聚合指标无法揭示这次攻击

### 练习5：模型供应链分析

检查模型文件以寻找被入侵的迹象。

```bash
# 列出所有模型文件及其大小
docker-compose exec model-registry ls -la /app/models/

# 检查pickle文件中是否存在可疑内容
# pickletools 可以反汇编pickle文件以揭示操作
docker-compose exec model-registry python3 -c "
import pickletools
print('=== 恶意模型反汇编 ===')
with open('/app/models/malicious_model.pkl', 'rb') as f:
    pickletools.dis(f)
"

# 比较干净模型和恶意模型的反汇编结果
docker-compose exec model-registry python3 -c "
import pickletools
print('=== 干净模型反汇编（前50行） ===')
with open('/app/models/sentiment_clean.pkl', 'rb') as f:
    pickletools.dis(f)
" 2>&1 | head -50

# 在pickle数据中查找 os.system 调用
docker-compose exec model-registry python3 -c "
import re
for name in ['malicious_model.pkl', 'sentiment_clean.pkl', 'sentiment_backdoored.pkl']:
    path = f'/app/models/{name}'
    with open(path, 'rb') as f:
        data = f.read()
    suspicious = []
    for pattern in [b'os', b'system', b'exec', b'eval', b'subprocess', b'__reduce__']:
        if pattern in data:
            suspicious.append(pattern.decode())
    status = '可疑' if suspicious else '干净'
    print(f'{name:<35} [{status}] {suspicious}')
"
```

**注意观察：**
- `pickletools.dis()` 可以揭示pickle文件中的操作码
- 恶意的pickle文件包含对 `os.system`、`subprocess` 等的引用
- 干净的ML模型通常引用 sklearn、numpy 和模型参数
- 自动扫描危险导入可以检测到一些攻击

## 漏洞总结

| # | 漏洞 | 影响 | MITRE ATLAS |
|---|--------------|--------|-------------|
| 1 | 在 `/load` 端点上进行Pickle反序列化 | 远程代码执行 | AML.T0010 - ML供应链入侵 |
| 2 | 上传时无模型验证 | 恶意模型注入 | AML.T0010 - ML供应链入侵 |
| 3 | 未认证的注册表API | 对所有模型的完全读写访问 | AML.T0010 - ML供应链入侵 |
| 4 | 训练模型中的后门触发器 | 隐秘的预测操纵 | AML.T0020 - 投毒训练数据 |
| 5 | 训练数据投毒 | 针对性的模型准确率下降 | AML.T0020 - 投毒训练数据 |
| 6 | 调试端点暴露内部路径 | 信息泄露 | AML.T0044 - 完全ML模型访问 |
| 7 | 无模型来源或完整性检查 | 被篡改的模型无法被检测 | AML.T0010 - ML供应链入侵 |

## 防御措施（讨论）

完成练习后，考虑以下缓解措施：

- **避免使用pickle**：使用更安全的序列化格式（ONNX、safetensors、SavedModel）
- **模型签名**：对模型进行加密签名，加载前验证签名
- **输入验证**：在存储前扫描上传文件中是否存在危险操作码
- **认证机制**：要求所有注册表操作都需凭据
- **数据溯源**：追踪和验证所有训练数据的来源
- **按类别评估**：监控每个类别/主题的准确率，而不仅仅是聚合指标
- **异常检测**：标记训练数据标签中的异常模式
- **沙箱加载**：在隔离的、受限的环境中反序列化模型

## 清理

```bash
docker-compose down -v
```

## 下一实验

前往 [Lab 06：模型抽取与盗窃](../lab06-model-extraction/) 攻击ML模型的机密性。
