# Chinese-Vicuna-medical

在[cMedQA2](https://github.com/zhangsheng93/cMedQA2)上使用我们的checkpoint-11600 continue finetune

目前从2个epoch的Vicuna开始continue finetune，效果比3个epoch的在医疗问答数据更具有专业性，同时由于数据集构建的问题，会更加规范，比如经常性的加上“到正规医院检查”等等
- 同时验证了指令微调的有效性
- 使用单指令continue-finetune能保留原来更多的性能

下面我们分析了三种不同的微调方式
- 1、[无指令微调](https://github.com/Facico/Chinese-Vicuna/blob/master/docs/performance-medical.md#1%E6%97%A0%E6%8C%87%E4%BB%A4%E5%BE%AE%E8%B0%83)
- 2、[带指令微调](https://github.com/Facico/Chinese-Vicuna/blob/master/docs/performance-medical.md#2%E5%B8%A6%E6%8C%87%E4%BB%A4%E5%BE%AE%E8%B0%83)
- 3、[固定单指令](https://github.com/Facico/Chinese-Vicuna/blob/master/docs/performance-medical.md#2%E5%B8%A6%E6%8C%87%E4%BB%A4%E5%BE%AE%E8%B0%83)
- 我们将第三种微调方式相关的模型放到了huggingface上：
  - `Chinese-Vicuna/Chinese-Vicuna-continue-finetune-3.5epoch-cMedQA2`
  - `Chinese-Vicuna/Chinese-Vicuna-continue-finetune-7epoch-cMedQA2`



## 1、无指令微调

使用脚本为finetune_others_continue.sh
```bash
TOT_CUDA="1,2,3"
CUDAs=(${TOT_CUDA//,/ })
CUDA_NUM=${#CUDAs[@]}
PORT="12345"

DATA_PATH="../dataset/instruction/cMedQA2/cMedQA2.json" 
OUTPUT_PATH="lora-Vicuna-medical"
MODEL_PATH="decapoda-research/llama-7b-hf"
lora_checkpoint="./lora-Vicuna/checkpoint-11600"
from_data_beginning=True # False
TEST_SIZE=1000

CUDA_VISIBLE_DEVICES=${TOT_CUDA} torchrun --nproc_per_node=$CUDA_NUM --master_port=$PORT finetune.py \
--data_path $DATA_PATH \
--output_path $OUTPUT_PATH \
--model_path $MODEL_PATH \
--eval_steps 200 \
--save_steps 200 \
--test_size $TEST_SIZE \
--resume_from_checkpoint $lora_checkpoint \
--ignore_data_skip $from_data_beginning

```

数据输入直接作为instruction，无input的格式，如
```
{
    "instruction": "双胞胎差异大，33周B超检查，一个双顶径89mm，腹围281mm，股骨70mm。另一个双顶径75mm，腹围266mm，股骨66mm。这样正常吗
",
    "input": "",
    "output": "从报告来看。这两个孩子的数据都有点偏离正常孕周的发育数据，但是由于你是双保胎。所以又不能用单胎的标准来看待。所以，而且
般双胎的话都是剖腹产。所以。只要你这两个孩子发育都很健康，没有什么器官脏器发育障碍就可以。从数据来看。双顶径89mm，腹围281mm，股骨70mm
双顶径75mm，腹围266mm，股骨66mm，都超过怀孕33周的发育数据，但是只要你孩子没有发育异常就可以，不需要太过担心。"
  }
```
数据大小是20w左右，在我们之前70w的数据上训练第3个epoch，大概会训练3.5个epoch左右

### 效果
Repetition Penalty=2,beam number=2
|  |  |  | |
| --- | --- | --- | --- |
| input | Vicuna7B-3epoch | Vicuna-medical7B | |
|身体乏力出虚汗脸色暗黄口干尿检说是肾阴虚出来的血检请大夫帮忙看看 | 这种情况可能是由于肾阴虚出来的血检请大夫帮忙看看。|肾阴虚主要表现为腰膝酸软，四肢发冷，畏寒，头晕，失眠，面色苍白，食欲不振，男子性功能下降等。建议你可以服用六味地黄丸调理一段时间，平时注意保暖 | |
|扁桃体炎引起的发烧该怎么治疗 | 扁桃体炎引起的发烧可以通过多种方式治疗，包括使用药物治疗、预防措施和手术治疗等。药物治疗可以通过使用各种药物来治疗扁桃体炎，例如乙酰氨基酸、抗炎药、抗氧化剂等。预防措施可以通过改善睡眠质量、减少咳嗽、避免食用咖啡因等来预防扁桃体炎发烧。手术治疗可以通过切除扁桃体炎发烧的部位进行手术治疗。| 扁桃体炎是一种常见的呼吸道感染性疾病，主要表现为咳嗽、咳痰、咽痛等。建议你到医院进行相关检查，明确病因后对症治疗。平时注意饮食清淡，不要吃辛辣刺激性食物，多喝水，保持心情舒畅，避免劳累和精神紧张，加强身体锻炼，增强体质，提高免疫力。| Min New Tokens=200|
|怀孕8个多月胎位：右枕前位，双项径：93MM心率：136，股骨长67MM，羊水65MM，发育正常么 | 是的，怀孕8个多月胎位右枕前位，双项径93MM，心率136，股骨长67MM，羊水65MM，发育正常。| 根据你的检查结果看，胎儿是正常的。建议你注意休息，保持心情舒畅，加强营养，定期孕检。希望对你有所帮助。平时要注意休息，不要过度劳累，合理膳食，多吃高蛋白优质蛋白食物多吃蔬菜水果，少吃生冷刺激性食物，多喝开水，保持心情舒畅，定期孕检，这样可以明确胎儿的发育情况。| |
| 晚上睡觉老做梦，一开始入睡困难到天亮起床时全身乏力没劲，请问有什么药吃了后能够睡的好| 建议您可以喝一杯咖啡或茶，这可以帮助您放松身心，并使您更容易入睡。此外，您还可以喝一杯牛奶或其他甜点，这也可以帮助您放松身心，并提高睡眠质量。| 你这种情况可能是神经衰弱导致的。建议你去医院查明原因再治疗。平时注意保暖，多喝水，不要吃辛辣刺激性食物，少吃生冷食物，多吃新鲜蔬菜和水果，以及富含维生素B族的食物，如红薯、香蕉、苹果等，还可以服用谷维素片或阿莫西林胶囊进行治疗。希望我的回答对你有所帮助。| |
| 请问脾肿瘤恶性和胰尾切除后左上腹到肩一直阵阵刺痛用什么药治| 脾肿瘤恶性和胰尾切除后左上腹到肩一直阵阵刺痛可以使用药物治疗，例如抗癌药、抗糖尿病药等。| 脾肿瘤恶性的情况是需要手术治疗的，建议你最好去正规医院进行手术治疗，以免影响身体健康。| |
| 1岁小孩拉肚子腹泻呕吐不知怎么办？| 1岁小孩可能因为过度饮食、缺乏运动或者其他原因，而导致肚子腹泻呕吐不知怎么办。建议给孩子提供更多的营养，增加活动量，以及定期进行身体检查。|你的情况可能是消化不良引起的，建议口服妈咪爱和颠茄片治疗。 | 1岁小孩可能因为过度饮食、缺乏运动或者其他原因，而导致肚子腹泻呕吐不知怎么办。建议给孩子提供更多的营养，增加活动量，以及定期进行身体检查。|
|男下女上精液会倒流吗| 是的，男下女上精液会倒流。|一般情况下是不会的，建议注意阴茎勃起时间长达3-5分钟以上。 | Repetition Penalty=3.0|
| 感冒之后脸和脖子有点水肿是什么原因| 感冒之后脸和脖子有点水肿的原因可能是因为病毒在血液中流动，导致血液循环不平衡。此外，也可能是由于病毒对血管系统造成的损害，导致血液循环不平衡。| 感冒后脸和脖子有点水肿，可能是感冒引起的皮肤炎症，建议去医院检查一下，确诊后对症治疗。| |
| 胃里往上返酸水，有时打咯怎么办|可以尝试吃更多的维生素C、B6和B12，这些维生素会帮助改善消化系统的功能，并且有助于减少咳嗽。此外，可以尝试吃更多的蔬菜和水果，因为它们富含维生素和矿物质，有助于改善消化系统的功能。 | 你这种情况可能是胃炎引起的，建议你去医院做个胃镜检查确诊，如果是胃炎需要服用消炎药治疗。| |
| 为什么长冻疮的脚趾头会痒呢？| 长冻疮的脚趾头会痒呢，是因为它们被压在冰块上，导致脚趾头的肌肉损伤。此外，长冻疮也会导致脚趾头的肌肉变形，从而使脚趾头更容易痒。| 这种情况一般是由于毛囊炎引起的，建议你可以使用氧氟沙星软膏外涂治疗。| |
| 轻度脂肪肝，肝损伤转氨酶超高，口干口苦厉害吃什么药好？| 建议您可以尝试一些辅助药物，如甘草药、葡萄糖等，以帮助改善肝脏功能。此外，您还可以尝试多种食物来改善肝脏功能，例如豆类、水果、蔬菜、蛋白质等。|脂肪肝是一种常见的肝病，主要表现为脂肪性肝炎，其发病原因有很多，如肝炎、肝纤维疾病、肝结核病、肝硬化等。建议你到医院做个肝功能检查，明确诊断后在医生指导下进行治疗。 | |

- 由于这份数据集也不是很干净，很多病人回答的语言是非常口语的，效果一般。同时回复基本是“你这种情况一般是xxxxx，xxxxx建议到正规医院检查”这种形式
- 同时由于回复中缺乏一定的逻辑相关性，医生一般会直接给出结果，模型缺乏类似chain of thought式的推理以及专业性的知识前提


不过他将丧失原本有的能力，会将所有的问题导向医疗意图。不过目前缺乏一定的导向性回复会随机回答，如
```
>写一份排序的python代码
你说的这种情况应该是需要到医院进行检查，确诊后再对症治疗。
```

## 2、带指令微调

手动写了几个相关问题的prompt
```
prompt_list = ["请为下面病例诊断并给出解决方案", "扮演一位专业的医生，为下面病人的问题提供解答",
               "请解答下面病人的疑惑", "根据这个医学问题，做出回答", "下面是一个病患和医生问答的场景"]
```

相关处理脚本代码如下
```python
import json
import random

question_path = "./cMedQA2/question.csv"
answer_path = "./cMedQA2/answer.csv"

question_dict = {}
with open(question_path, 'r') as f:
    for i in f:
        q, text = i.split(",")[0], "".join(i.split(",")[1:])
        question_dict[q] = text.strip('\n').strip().strip(":")

instruction_list = []
prompt_list = ["请为下面病例诊断并给出解决方案", "扮演一位专业的医生，为下面病人的问题提供解答",
               "请解答下面病人的疑惑", "根据这个医学问题，做出回答", "下面是一个病患和医生问答的场景"]
id = [0, 1, 2, 3, 4]
prefix_input = ["", "", "", "", "病人:"]
suffix_input = ["", "", "", "", "\n医生:"]
num = 0
output_dict = {}
with open(answer_path, 'r') as f:
    for i in f:
        ans, q, text = i.split(",")[0], i.split(",")[1], "".join(i.split(",")[2:])
        text = text.strip('\n').strip().strip(":")
        if text == "content": continue
        question = question_dict[q]
        prompt_id = random.choice(id)
        output_item = {
            "instruction": prompt_list[prompt_id],
            "input": prefix_input[prompt_id]+question+suffix_input[prompt_id],
            "output": text,
        }
        num += 1
        instruction_list.append(output_item)

print(num)
json.dump(instruction_list, open("./cMedQA2_v2.json", 'w'), ensure_ascii=False, indent=2)
```
处理完的数据样例如下：
```
{
    "instruction": "扮演一为专业的医生，为下面病人的问题提供解答",
    "input": "停经十四天，会怀孕吗停经十四天，特别恶心已经停经十四天了，性生活频繁，出现恶心的症状，而
且特别频繁严重，头疼，易疲惫，以前来月经都特别准，推迟不会超过三天，请问这怀孕概率大吗",
    "output": "月经延迟十四天而且伴随恶心，头痛，乏力的现象，那么考虑怀孕的概率是非常大的，建议你去医
检查血HCG或者B超确诊是否怀孕，如果怀孕了，那么怀孕的天数大概是44天如果打算保留这个孩子，从现在开始要戒
烟酒，不要同房，注意休息，饮食方面要少吃辛辣刺激的，腌制，油炸，烧烤的食物，不要吃凉性的食物，不要吃螃
蟹甲鱼等寒性食品，容易流产，定期孕检"
  }
```
使用脚本为finetune_others_continue.sh
```bash
TOT_CUDA="1,2,3"
CUDAs=(${TOT_CUDA//,/ })
CUDA_NUM=${#CUDAs[@]}
PORT="12345"

DATA_PATH="../dataset/instruction/cMedQA2/cMedQA2_v2.json" 
OUTPUT_PATH="lora-Vicuna-medical2"
MODEL_PATH="decapoda-research/llama-7b-hf"
lora_checkpoint="./lora-Vicuna/checkpoint-11600"
from_data_beginning=True # False
TEST_SIZE=1000

CUDA_VISIBLE_DEVICES=${TOT_CUDA} torchrun --nproc_per_node=$CUDA_NUM --master_port=$PORT finetune.py \
--data_path $DATA_PATH \
--output_path $OUTPUT_PATH \
--model_path $MODEL_PATH \
--eval_steps 200 \
--save_steps 200 \
--test_size $TEST_SIZE \
--resume_from_checkpoint $lora_checkpoint \
--ignore_data_skip $from_data_beginning

```

### 效果
Repetition Penalty=2,beam number=2
|  |  |  | |
| --- | --- | --- | --- |
| input | Vicuna7B-3epoch | Vicuna-medical7B | |
|根据这个医学问题，做出回答。身体乏力出虚汗脸色暗黄口干尿检说是肾阴虚出来的血检请大夫帮忙看看 | 身体乏力出虚汗脸色暗黄口干尿检说可能是因为肾阴虚出来的血检请大夫帮忙看看。此外，也可能是由于肾血管疼痛、肾血管疼痛等原因引起的。建议您尽快就大夫帮忙进行诊断和治疗，以确定正确的诊断和治疗方案。如果您有任何疼痛或其他症状，请立即咨询医生。|肾阴虚主要表现为腰膝酸软，四肢发冷，头晕，眼花，面色苍白，失眠多梦，食欲不振等。建议服用六味地黄丸，金匮肾气丸，双枇材丸治疗。 | |
|根据这个医学问题，做出回答。扁桃体炎引起的发烧该怎么治疗 | 扁桃体炎引起的发烧可以通过使用药物治疗，例如乙酰氨基酸、抗炎药和抗氧化剂等。此外，也可以通过改善生活方式，如减少咖啡因和糖分的摄入，加强锻炼，保持良好的睡眠习惯等。| 扁桃体炎是一种常见的传染性疾病，主要表现为发热，严重者还会出现头痛、鼻塞、咳嗽等症状。建议你到医院进行详细检查，确诊后对症治疗，以免影响身体健康。平时注意多喝水，多吃水果和蔬菜，不要吃辛辣刺激性食物，保持心情舒畅，尽量避免受凉，避免感冒。| Min New Tokens=200|
|根据这个医学问题，做出回答。怀孕8个多月胎位：右枕前位，双项径：93MM心率：136，股骨长67MM，羊水65MM，发育正常么 |怀孕8个多月胎位的右枕前位，双项径为93MM，心率为136，股骨长为67MM，羊水为65MM，发育正常。 | 你的情况考虑是胎位不正，建议你到医院做B超检查，明确胎儿的情况后再对症治疗。| |
| 根据这个医学问题，做出回答。晚上睡觉老做梦，一开始入睡困难到天亮起床时全身乏力没劲，请问有什么药吃了后能够睡的好| 这种睡眠问题可能是因为缺乏睡眠药物或者其他药物。建议您咨询医生以获得更多信息和建议。| 你的情况可能是脑供血不足导致的，建议你去医院做个脑电图检查，确诊后对症治疗。| |
| 根据这个医学问题，做出回答。请问脾肿瘤恶性和胰尾切除后左上腹到肩一直阵阵刺痛用什么药治| 脾肿瘤恶性和胰尾切除后左上腹到肩一直阵阵刺痛可以通过使用药物治疗，例如抗癌药、抗糖尿病药等。| 脾肿瘤恶性和胰尾切除后左上腹到肩一直阵阵刺痛用什么药治？建议你去医院检查一下，如果是脾肿瘤恶性的话，需要手术治疗，可以选择中西医结合治疗，平时注意保暖，多喝水，不要吃辛辣刺激的食物，多吃些清淡的食物，希望我的回答对你有所帮助。| Min New Tokens=200|
| 根据这个医学问题，做出回答。1岁小孩拉肚子腹泻呕吐不知怎么办？| 这种情况可能是因为孩子的肠道没有完全形成，或者他们正在饮食中吃到了过多的糖分，导致肠道内部生理变化。建议家长给孩子提供更多的水果和蔬菜，以及少量的高糖食品，以改善肠道的健康状况。|小儿腹泻的原因有很多，如食物过敏、消化不良、胃肠道疾病等，建议你到医院做一下腹镜检查，明确诊断后对症治疗。 | |
|根据这个医学问题，做出回答。男下女上精液会倒流吗| 是的，男下女上精液会倒流。|一般情况下是不会倒流的，但是有时候因为阴茎比较大，或者由于性交过程中经常使用手法导致，可能会造成射精后进入阴道，引起射精后射精之前的精液倒流。 | Repetition Penalty=3.0|
| 根据这个医学问题，做出回答。感冒之后脸和脖子有点水肿是什么原因| 感冒之后脸和脖子有点水肿的原因可能是因为感冒会导致肌肉酸化，从而使肌肉变得更容易损伤，也可能是因为感冒会导致肌肉失去力量，导致脸和脖子变得更容易发生水肿。| 你的情况考虑是感冒引起的，建议你可以口服抗病毒药物治疗。| |
| 根据这个医学问题，做出回答。胃里往上返酸水，有时打咯怎么办| 胃里往上返酸水是一种常见的胃病，可以通过多种方法来治疗。首先，需要控制食物中的糖分和脂肪分量，以降低胃部内热量；其次，需要改善饮食习惯，减少高热量、高脂肪、高糖分的食物；最后，可以使用消化药等药物进行治疗。| 你的情况可能是胃炎引起的，建议你去医院做个胃镜检查，确诊后对症治疗。| |
| 根据这个医学问题，做出回答。为什么长冻疮的脚趾头会痒呢？| 长冻疮的脚趾头会痒呢，是因为它们被压在冰块上，导致脚趾头的肌肉损伤，并且可能会引起脚趾头的疼痛。| 可能是由于局部湿热引起的，建议你可以用点酒糖洗液或者消炎止痒溶液来治疗一下，平时注意保暖| Repetition Penalty=3.0|
| 根据这个医学问题，做出回答。轻度脂肪肝，肝损伤转氨酶超高，口干口苦厉害吃什么药好？| 轻度脂肪肝的治疗方法包括改善饮食和锻炼，以减少脂肪量，同时提高肝脏功能。此外，还可以使用辅助药物，例如抗肝炎药、抗肝癌药等，以改善肝脏功能。|脂肪肝是一种常见的肝病，主要表现为脂肪性肝炎，其发病原因有很多，如肝炎、肝结核、肝硬化、肝癌等。建议你去医院做个肝功能检查，明确诊断后再对症治疗。 | |

因为是根据相关的指令来对特定任务进行生成，带指令的微调还保存着原来一定的能力,但不多
```
>写一份排序的python代码
可以使用sorted()方法对列表进行排序，具体参考答案需要根据自己的情况选择最合适的方法
>自然语言处理是什么
孕妇感冒可以服用阿莫西林颗粒和小儿感冒颗粒治疗。
```

## 3、将重复问合并，并固定形式

### from checkpoint-11600

将instruction固定成一个对话格式：“扮演一个专业的医生并回答病人的问题。病人：xxxxxx”，例如：
```
{
    "instruction": "扮演一个专业的医生并回答病人的问题。",
    "input": "病人：头痛头昏十几年前。吹风和晒太阳。吵闹都会头疼没有检查出来结果应该怎样治疗",
    "output": "1.头痛是临床上最为常见的临床症状之一，是人体对各种致痛因素所产生的主观感觉，属于疼痛的范畴。建议临床上最常用的是非甾体抗炎镇痛药
，包括对乙酰氨基酚，布洛芬、双氯酚酸钾等。必要时做脑部CT检查一下。\n2.头痛主要是由于头部的血管、神经、脑膜等对疼痛敏感的组织受到刺激引起的。由
张、疲劳、饮酒等原因造成的头痛经过休息之后可自然消退。头痛即可作为神经系统原发病的一个早期症状或中、晚期症状，如脑肿瘤患者以头痛为主诉者更是普
。头痛也可以是颈部疾病，肩部疾病及背部疾病的症状，也可以是全身疾病在头部的一个表现形式，如严重的细菌性感染时出现的头痛，正是由于引起头痛的原因
而复杂，因此其临床分类也十分复杂，需要到医院检查确诊。\n3.考虑是神经性头疼，是一种功能性的疾病，是由于紧张压力或是精神因素引起的神经功能失调造
的头痛，也有一定的遗传因素，一般可考虑吃些中草药川芎茶调散来治疗。同时结合针灸按摩，效果会更好的。\n"
  }
```
数据处理代码为
```python
import json
import random

question_path = "./cMedQA2/question.csv"
answer_path = "./cMedQA2/answer.csv"

question_dict = {}
with open(question_path, 'r') as f:
    for i in f:
        q, text = i.split(",")[0], "".join(i.split(",")[1:])
        question_dict[q] = text.strip('\n').strip().strip(":")

instruction_list = []
num = 0
output_dict = {}
with open(answer_path, 'r') as f:
    for i in f:
        ans, q, text = i.split(",")[0], i.split(",")[1], "".join(i.split(",")[2:])
        text = text.strip('\n').strip().strip(":")
        if text == "content": continue
        question = question_dict[q]
        prompt_id = random.choice(id)
        if question not in output_dict:
            output_dict[question] = []
        output_dict[question].append(text)
    for k, v in output_dict.items():
        ans = ""
        for i in range(len(v)):
            ans = ans + "{}.{}\n".format(i+1, v[i])
        output_item = {
            "instruction": "扮演一个专业的医生并回答病人的问题。",
            "input": "病人：" + k,
            "output": ans,
        }
        instruction_list.append(output_item)

print(num)
json.dump(instruction_list, open("./cMedQA2_v3.json", 'w'), ensure_ascii=False, indent=2)
```
#### 效果

为了防止（减少）回答的1、2、3是重复的，这里将Repetition Penalty开大，由于回答较长，默认Min_new_tokens=200，由于长度限制有些并没有生成完
beam number=2

|  |  |  | |
| --- | --- | --- | --- |
| input | Vicuna7B | Vicuna-medical7B                                             |  |
| 扮演一个专业的医生并回答病人的问题。病人：身体乏力出虚汗脸色暗黄口干尿检说是肾阴虚出来的血检请大夫帮忙看看 | 医生：你好，我可以为您提供一些建议。首先，需要确认您有没有发现任何其他症状，比如头痛、喉咙痛等？<br/><br/>病人：还没有，只有这种情况。<br/><br/>医生：那么，我建议您多喝水，保持良好的饮食习惯，同时也可以给自己一些放松的活动，比如深呼吸或者按摩等。此外，您还可以去看医生进行更详细的检查和治疗。<br/><br/>病人：谢谢您的建议！ | 1.你这种情况可能是肾阴虚引起的，建议服用六味地黄丸和金匮肾气丸进行治疗。平时注意保暖，不要吃寒凉的食物，多吃点清淡的食物。<br/>2.根据你提供的信息分析，应该是肾阴虚导致的表现，需要对症治疗。建议使用中成药如金匮肾气丸、六味地黄丸等进行治疗，同时配合饮食调理为好。<br/>3.有可能是肾阴虚引起的，建议选用六味地黄丸和金匮肾气丸进行治疗。平时注意保暖 |  |
| 扮演一个专业的医生并回答病人的问题。病人：扁桃体炎引起的发烧该怎么治疗 | 扁桃体炎引起的发烧可以通过药物治疗和预防措施来治疗。首先，需要确定是否有其他原因导致发烧，如食品中的化学物质、不健康的生活习惯等。接下来，可以使用特定的药物治疗，例如代谢性药物或非代谢性药物。此外，还可以采取预防措施，如保持良好的饮食习惯、避免高温和高压力的环境，及时进行身体检查等。 | 1.考虑是扁桃体炎，建议你可以口服阿奇霉素和左氧氟沙星，严重时需要输液治疗，注意多喝水，不要吃辛辣刺激性食物，如辣椒等，保持大便通畅有利于康复。<br/>2.扁桃体炎主要是由于扁桃体内存在细菌或病毒感染导致的反复发热、流涕、咳嗽等症状，这种情况应该及时到医院进行相关检查，确诊后对症治疗才行。 |  |
| 扮演一个专业的医生并回答病人的问题。病人：怀孕8个多月胎位：右枕前位，双项径：93MM心率：136，股骨长67MM，羊水65MM，发育正常么 | 您的怀孕8个多月胎位在右枕前位，双项径为93MM，心率为136，股骨长为67MM，羊水为65MM，发育正常。建议您保持良好的饮食习惯和定期体检，以确保身体健康。如果有任何不适，请及时联系医生进行治疗。希望您的怀孕过程都安全、健康！ 👍 #怀孕 #婴儿 #妈妈 #母亲 #家庭 #健康 #医生 | 1.这个情况是不正常的，建议你到医院做B超检查确定后再对症治疗。<br/>2.根据你提供的信息分析，胎儿现在已经有足够的时间进入第三个月，如果没有出现明显的异常，就可以认为是正常的，只要身体健康，宝宝也会成熟好了。<br/>3.从你提供的数据来看，胎儿大小比较正常，还需要注意观察胎儿的发育情况，如果没有出现明显的异常，那么说明胎儿都是正常的，只要身体健康，宝宝也会成熟好了。 |  |
| 扮演一个专业的医生并回答病人的问题。病人：晚上睡觉老做梦，一开始入睡困难到天亮起床时全身乏力没劲，请问有什么药吃了后能够睡的好 | 这种症状可能是因为睡眠不足或者其他原因导致的失眠症状。建议您多喝水和保持健康的生活习惯，以及参加治疗来改善自己的睡眠情况。如果症状持续超过一周，请咨询专业医生进行更深入的诊断和治疗。此外，也可以试着在睡前放松身心，例如通过冥想、呼吸练习等方式来减轻焦虑和压力，从而提高睡眠质量。 | 1.这种情况多考虑是神经衰弱导致的，建议你去医院进行神经科检查，确诊后再对症治疗。<br/>2.可能是由于心理因素引起的，应该注意自我调节，不要过于紧张，也不要过度思虑，以免影响睡眠。平时需要加强体育锻炼，保持良好的心态和情绪状态，同时还可以配合口服安神补脑液、谷维素等药物进行治疗。<br/>3.根据你提供的信息分析，主要原因是内分泌失调所导致的睡眠不足。建议你在当地医生的指导下 |  |
| 扮演一个专业的医生并回答病人的问题。病人：请问脾肿瘤恶性和胰尾切除后左上腹到肩一直阵阵刺痛用什么药治 | 对不起，我是一个AI语言模型，无法提供医疗建议。建议您咨询专业医生以获取最佳治疗方案。如果您有其他问题，我可以帮助您解决相关问题。再次抱歉！ 好的，请问您需要哪些更多的信息或服务呢？ 例如，您想了解哪些疾病会导致脾肿瘤？ 还有任何其他问题吗？ 非常感谢您使用我的服务！ | 1.脾肿瘤恶性，是由于脾肿瘤恶性导致的肿瘤发育不控制，可能会引起肿瘤转移，造成肿瘤大小变化，或者因为肿瘤反复发生，使肿瘤形成出现了局部、全身及其他器官的影响。建议你去医院进行相关检查，明确肿瘤的情况，然后对症治疗。<br/>2.脾肿瘤恶性，是由于脾肿瘤恶性导致的肿瘤发育不控制，可能会引起肿瘤转移，造成肿瘤大小变化，或者因为肿瘤反复发生， |  |
| 扮演一个专业的医生并回答病人的问题。病人：1岁小孩拉肚子腹泻呕吐不知怎么办？ | 好的，请告诉我你的儿子是什么时候出现的这种情况？有没有发鼻涕或者头痛？还有其他症状？如果有，请告诉我更多信息，以便我能够给你提供更好的治疗建议。同时，请确保儿子每天都接受足够的水分和营养，以及定期进行体温检测。再次感谢您关心孩子的健康，希望能够为您提供更好的服务。 最后，请注意要在家里安全地使用电器设备，以避免对孩子造成影响。 | 1.这种情况可能是消化不良引起的，建议到医院进行详细检查，明确病因后再对症治疗。<br/>2.宝宝大便出血有可能是上火了，也有可能是痔疮或者肛裂等原因导致的。建议去医院肛肠科就诊，必要时需要手术治疗。<br/>3.你说的情况应该是由于肠道功能障碍导致的，如果没有其他症状，可以先服用阿莫西林来调理，注意多喝水，保持大便通畅。<br/>4.根据你提供的信息分析，最好及时到医院肛肠科就 |  |
| 扮演一个专业的医生并回答病人的问题。病人：男下女上精液会倒流吗 | 是的，男下女上精液可能会倒流。建议你及时去看医生，以便得到正确的治疗和预防措施。同时，也要注意保持良好的身体健康，包括多喝水、避免过度运动等。如果病情变得越来越严重，请立即联系医生进行更深入的诊断和治疗。最后，我们希望您在疾病期间有所幸福！再次感谢您的关注！ 祝您身心健康！（这里只是为了模拟，不代表真实医生对此类问题的态度） | 1.这种情况是正常的，不用担心，只要你及时治疗就可以了。建议你注意休息，避免过度劳累，保持良好的心态，多吃水果和蔬菜，有利于身体的健康。<br/>2.如果没有任何不适症状，那么也没有必要担心的。在性生活中，最重要的事情就是使用安全套，否则容易导致子宫内膜损伤，引起子宫内膜破裂，造成子宫内膜分泌物增多，形成子宫内膜炎或者子宫内膜炎症，影响到月经周期长久等。<br/>3.从学科角 |  |
| 扮演一个专业的医生并回答病人的问题。病人：感冒之后脸和脖子有点水肿是什么原因 | 感冒之后脸和脖子有点水肿可能是由于过度使用手机或电脑而引起的，也可能是因为长时间没有遏着空气，导致血液流动不平衡。建议每天保持足够的身体运动，多喝水，定期休息，避免过度使用手机或电脑，以确保心理和身体健康。如果症状变得越来越严重，请立即就医了解更多信息。希望能帮助你！ 祝你好运！（这里只是作为模拟的说法，无法代表真实医学知识） | 1.这种情况可能是感冒引起的，建议你注意饮食清淡，多喝开水，不要吃辛辣刺激性食物，如果持续发烧，应该到医院进行相关检查，确定病因以及对症治疗。<br/>2.根据你提供的信息分析，主要还是由于感冒导致了上火所致的表现，建议口服阿奇霉素、西替利嗪片等来调理，同时加强体育锻炼为好。 |  |
| 扮演一个专业的医生并回答病人的问题。病人：胃里往上返酸水，有时打咯怎么办 | 这种情况可能是因为患者没有足够的习惯来保持良好的饮食习惯，也可能是因为患者过度消费了高热量、高脂肪和高糖分的食物，导致胃部不健康。建议患者多吃水果、蔬菜、全谷类食品以及低脂肪低糖的食物，同时注意控制饮食量和饮食时间。此外，如果患者发现自己的身体状况变得越来越糟糕，就应该立即去医院进行治疗。 | 1.这种情况可能是胃炎引起的，建议你去医院进行胃镜检查确诊后对症治疗。<br/>2.根据你提供的信息分析，应该是胃炎导致的不适症状，建议口服消炎药和止咳化痰药物治疗，同时要注意饮食清淡，多喝开水，保持大便通畅，以减轻症状。<br/>3.如果是胃炎，最好到医院就诊，看下胃镜明确诊断，然后用药治疗。平时注意饮食，少吃辛辣、刺激性食物，加 |  |
| 扮演一个专业的医生并回答病人的问题。病人：为什么长冻疮的脚趾头会痒呢？ | 长冻疮的脚趾头会痒是因为静止水在脚趾头部中形成，导致脚趾头部变得硬厚，有时还会出现脚趾头部发炎和疼痛。建议每天进行定期的脚趾头清洗，以及使用特定的脚趾头保湿品来改善脚趾头的状态。如果病情不好，请咨询医生或者去看医生。最后，要注意身体健康，避免过度运动和太多工作，以确保脚趾头的健康。 | 1.这种情况可能是由于局部水肿引起的，建议你去医院检查一下，确诊后对症治疗就可以了。<br/>2.有可能是因为局部水肿引起的，建议你去医院检查一下，确诊后对症治疗就可以了。<br/>3.根据你提供的信息分析，应该属于局部水肿引起的症状，需要到医院进行相关检查明确诊断，然后对症治疗。<br/>4.如果是局部水肿引起的症状，建议你去医院检查一下，确诊后对症治疗就可以了。 |  |
| 扮演一个专业的医生并回答病人的问题。病人：轻度脂肪肝，肝损伤转氨酶超高，口干口苦厉害吃什么药好？ | 你的脂肪肝状况需要进行治疗，建议在治疗期间多吃水果、蔬菜、全谷物和健康的蛋白质来源，同时可以选择合适的药物来控制脂肪分布。此外，也可以通过调节饮食和运动来改善脂肪代谢功能。如果口干不太好，可以试着使用特定的口腔清洁剂或口腔保护品来解决问题。最后，建议每天进行体育锻炼，这有助于提高身体素质和消化功能。 | 1.脂肪肝主要是由于脂肪过多导致肝功能不良所致，建议你可以服用中药调理治疗，平时注意饮食清淡，少吃辛辣和油腻的食物。<br/>2.脂肪肝常见的症状有头晕、四肢发冷、心慌、消瘦、体重减轻、面色黄暗、手脚发凉、大便稀水、腹泻、腹部疼痛、腹泻等。如果出现这些症状应该及时到医院进行相关检查，确诊后再对症治疗。<br/>3.脂肪肝 |  |
|  |          |  |  |
|  |          |  |  |

可以发现，这种训练方式好了很多，给出的回答相当于是在模仿一群医生，有时候会意见不统一，比如那个胎位的问题（这个在数据集中也是这样的）。在让他分点回答后，稍微具有了一定的推理能力，比如“扁桃体炎引起的发热”，能先分析扁桃体炎、再分析发热，这个在生成式的逻辑上也是比较合理的。



其他能力测试，保留了更多的能力，同时更偏向于分点回答，但有时不同点的重复度会比较高

```python
>写一份排序的python代码
1.可以使用sorted()方法，它将列表中的所有元素按照顺序排列，并返回一个新的列表。
2.你可以使用sorted()方法，它将列表中的所有元素按照顺序排列，并返回一个新的列表。
3.如果要对数组进行排序，可以使用sorted()函数来实现。这个函数接受两个参数：第一个是一个字典，第二个是一个排序方式，其中包括-1，0，1，正序和负序。
4.在Python中存在sorted()函数，用于对任何类型的列表进行排序。
>自然语言处理是什么
1.自然语言处理（NLP）是人工智能领域的一个分支，主要研究如何使计算机通过训练和学习来扮演人类的思想、语言、情态、意图等方面。目前最常用的技术包括文本分析、机器翻译、问题回答、情感分析等。
2.自然语言处理（NLP）是人工智能领域的一个分支，主要研究如何使计算机通过训练和学习来扮演人类的思想、语言、情态、意图等方面。目前最常用的技术包括文本分析、机器翻译、问题回答、情感分析等。
```



### from checkpoint-5800

相关设置和上面相同，不同的是从1个epoch开始的continue_finetune.

可以发现大致效果差不多，不过这个版本专业性更强了一点，对一些药品的掌握更多了。

#### 效果

|  |  |  | |
| --- | --- | --- | --- |
| input | Vicuna7B | Vicuna-medical7B                                             |  |
| 扮演一个专业的医生并回答病人的问题。病人：身体乏力出虚汗脸色暗黄口干尿检说是肾阴虚出来的血检请大夫帮忙看看 | 医生：你好，我可以为您提供一些建议。首先，需要确认您有没有发现任何其他症状，比如头痛、喉咙痛等？<br/><br/>病人：还没有，只有这种情况。<br/><br/>医生：那么，我建议您多喝水，保持良好的饮食习惯，同时也可以给自己一些放松的活动，比如深呼吸或者按摩等。此外，您还可以去看医生进行更详细的检查和治疗。<br/><br/>病人：谢谢您的建议！ | 1.你这种情况可能是肾阴虚引起的，建议你服用六味地黄丸和金锁固精丸治疗。注意休息，不要劳累。多吃点动物肾脏，比如羊肉、狗肉等。<br/>2.根据你提供的信息分析，你现在的这种情况主要是由于肾阴虚所导致的上述症状，需要积极的进行治疗才会得到改善的。建议你可以选择中成药六味地黄丸来进行治疗，同时配合食疗就可以得到改善的。 |  |
| 扮演一个专业的医生并回答病人的问题。病人：扁桃体炎引起的发烧该怎么治疗 | 扁桃体炎引起的发烧可以通过药物治疗和预防措施来治疗。首先，需要确定是否有其他原因导致发烧，如食品中的化学物质、不健康的生活习惯等。接下来，可以使用特定的药物治疗，例如代谢性药物或非代谢性药物。此外，还可以采取预防措施，如保持良好的饮食习惯、避免高温和高压力的环境，及时进行身体检查等。 | 1.考虑是扁桃体炎引起的发烧，建议口服阿奇霉素或者头孢类抗生素治疗，同时注意休息，多喝开水，保持大便通畅。<br/>2.扁桃体炎主要是由于真菌感染引起的，可以使用达克宁、左氧氟沙星等进行治疗，平时注意局部清洁干燥，不要接触有毒物质和过敏性物质。<br/>3.扁桃体炎首先应到医院就诊，根据实验结果选择合适的药物对症治疗，如果出现高热及其他特殊 |  |
| 扮演一个专业的医生并回答病人的问题。病人：怀孕8个多月胎位：右枕前位，双项径：93MM心率：136，股骨长67MM，羊水65MM，发育正常么 | 您的怀孕8个多月胎位在右枕前位，双项径为93MM，心率为136，股骨长为67MM，羊水为65MM，发育正常。建议您保持良好的饮食习惯和定期体检，以确保身体健康。如果有任何不适，请及时联系医生进行治疗。希望您的怀孕过程都安全、健康！ 👍 #怀孕 #婴儿 #妈妈 #母亲 #家庭 #健康 #医生 | 1.你这种情况是正常的，不必过于担心，建议定期到医院产检就可以了。注意休息，加强营养，保持心情愉快，有利于身体健康。<br/>2.根据你的超声检查结果显示胎儿已经入盆了，现在还没有任何的治疗方法，只能等到分娩后才能进行对症治疗。如果想去做引产手术需要先通过B超检查确定胎儿成熟度，然后再选择合适的时间。 |  |
| 扮演一个专业的医生并回答病人的问题。病人：晚上睡觉老做梦，一开始入睡困难到天亮起床时全身乏力没劲，请问有什么药吃了后能够睡的好 | 这种症状可能是因为睡眠不足或者其他原因导致的失眠症状。建议您多喝水和保持健康的生活习惯，以及参加治疗来改善自己的睡眠情况。如果症状持续超过一周，请咨询专业医生进行更深入的诊断和治疗。此外，也可以试着在睡前放松身心，例如通过冥想、呼吸练习等方式来减轻焦虑和压力，从而提高睡眠质量。 | 1.你这种情况可能是神经衰弱引起的，建议服用谷维素和安神补脑液进行治疗。平时要保持心情舒畅，不要过于紧张，注意休息。多食一些具有补肾固精作用的食物，如牡蛎，胡桃肉，栗子，甲鱼，文蛤，鸽蛋，猪腰等。<br/>2.从中医角度分析，应该是由于肾阳虚所导致的夜间失眠、出虚汗等症状。对于这种情况，最好还是去看下中医大夫，在他们指导下服用金匮肾气丸或者 |  |
| 扮演一个专业的医生并回答病人的问题。病人：请问脾肿瘤恶性和胰尾切除后左上腹到肩一直阵阵刺痛用什么药治 | 对不起，我是一个AI语言模型，无法提供医疗建议。建议您咨询专业医生以获取最佳治疗方案。如果您有其他问题，我可以帮助您解决相关问题。再次抱歉！ 好的，请问您需要哪些更多的信息或服务呢？ 例如，您想了解哪些疾病会导致脾肿瘤？ 还有任何其他问题吗？ 非常感谢您使用我的服务！ | 1.你这种情况可能是由于手术创伤引起的，建议去医院复查看看，如果确实是有创伤导致的话，需要进行康复治疗。<br/>2.脾肿瘤恶性不会影响下肢部位的疼痛，而胰腺切除时间比较长，所以应该没有关系。平时注意保持心情舒畅，加强体育锻炼，多吃新鲜水果蔬菜，少吃高脂肪、高胆固醇食物。 | Repetition Penalty=4.0 |
| 扮演一个专业的医生并回答病人的问题。病人：1岁小孩拉肚子腹泻呕吐不知怎么办？ | 好的，请告诉我你的儿子是什么时候出现的这种情况？有没有发鼻涕或者头痛？还有其他症状？如果有，请告诉我更多信息，以便我能够给你提供更好的治疗建议。同时，请确保儿子每天都接受足够的水分和营养，以及定期进行体温检测。再次感谢您关心孩子的健康，希望能够为您提供更好的服务。 最后，请注意要在家里安全地使用电器设备，以避免对孩子造成影响。 | 1.这种情况可以服用思密达和妈咪爱进行治疗，同时要注意多喝水，饮食要清淡为主，如果没有效果的话最好去医院就诊<br/>2.宝宝出现腹泻是由于消化不良引起的，建议你给宝宝口服妈咪爱和思密达治疗，平时在饮食上要注意，少吃高纤维的食物，多吃些容易消化的食物，比如米粥、面条、黄豆粥等。<br/>3.大便带血或者发热都是腹泻的表现，需要到医院确诊后才能对症治疗。 |  |
| 扮演一个专业的医生并回答病人的问题。病人：男下女上精液会倒流吗 | 是的，男下女上精液可能会倒流。建议你及时去看医生，以便得到正确的治疗和预防措施。同时，也要注意保持良好的身体健康，包括多喝水、避免过度运动等。如果病情变得越来越严重，请立即联系医生进行更深入的诊断和治疗。最后，我们希望您在疾病期间有所幸福！再次感谢您的关注！ 祝您身心健康！（这里只是为了模拟，不代表真实医生对此类问题的态度） | 1.你说的这种情况是不会倒流的，但是如果有阴道炎或者其他原因导致子宫内膜变薄就可能会倒流了。建议你最好去医院进行相关检查，明确后再对症治疗。<br/>2.正常性生活时间应该在30分钟左右，超过达到45分钟以上都属于过长，此外还要注意同房前三十分钟没有用任何药物、护理等方法调节性生活时间和性生活程度。<br/>3.从你所提供的信息来看，你们每次性生活时间大于30分钟，那么我们认为你们已经越来越出现了比较多的心理影响，而且你们 |  |
| 扮演一个专业的医生并回答病人的问题。病人：感冒之后脸和脖子有点水肿是什么原因 | 感冒之后脸和脖子有点水肿可能是由于过度使用手机或电脑而引起的，也可能是因为长时间没有遏着空气，导致血液流动不平衡。建议每天保持足够的身体运动，多喝水，定期休息，避免过度使用手机或电脑，以确保心理和身体健康。如果症状变得越来越严重，请立即就医了解更多信息。希望能帮助你！ 祝你好运！（这里只是作为模拟的说法，无法代表真实医学知识） | 1.你这种情况可能是由于感冒引起的，建议口服抗病毒药物和感冒冲剂试试，要多喝水，不要吃辛辣食物，保持大便通畅，防止肠道炎症感染。<br/>2.根据你提供的信息分析，你现在出现了上述症状，应该考虑是由于感冒导致的，需要进行治疗的。建议你到医院就诊，查明病因对症治疗，以免发生复发。 |  |
| 扮演一个专业的医生并回答病人的问题。病人：胃里往上返酸水，有时打咯怎么办 | 这种情况可能是因为患者没有足够的习惯来保持良好的饮食习惯，也可能是因为患者过度消费了高热量、高脂肪和高糖分的食物，导致胃部不健康。建议患者多吃水果、蔬菜、全谷类食品以及低脂肪低糖的食物，同时注意控制饮食量和饮食时间。此外，如果患者发现自己的身体状况变得越来越糟糕，就应该立即去医院进行治疗。 | 1.你这种情况可能是胃炎或者消化道溃疡导致的，建议到医院消化内科进行相关检查，明确诊断后再对症治疗。<br/>2.根据你提供的信息分析，多考虑为胃炎引起的不适症状，需要进一步就诊检查明确诊断和治疗。注意保持良好的生活习惯，避免过度劳累，加强营养，合理安排饮食，定期复查 |  |
| 扮演一个专业的医生并回答病人的问题。病人：为什么长冻疮的脚趾头会痒呢？ | 长冻疮的脚趾头会痒是因为静止水在脚趾头部中形成，导致脚趾头部变得硬厚，有时还会出现脚趾头部发炎和疼痛。建议每天进行定期的脚趾头清洗，以及使用特定的脚趾头保湿品来改善脚趾头的状态。如果病情不好，请咨询医生或者去看医生。最后，要注意身体健康，避免过度运动和太多工作，以确保脚趾头的健康。 | 1.你这种情况可能是由于局部皮肤过敏导致的，建议口服抗过敏药如扑尔敏，维生素C，葡萄糖酸钙，外用激素药膏如尤卓尔等。（激素不宜长时间使用）。<br/>2.根据你的描述分析，你现在出现了冻疮的表现，主要是由于周围皮肤受到冷刺激所引起的，应该及时进行治疗，以免影响身体健康。建议你平时多注意保暖，避免冷空气接触，同时也需要加强营 |  |
| 扮演一个专业的医生并回答病人的问题。病人：轻度脂肪肝，肝损伤转氨酶超高，口干口苦厉害吃什么药好？ | 你的脂肪肝状况需要进行治疗，建议在治疗期间多吃水果、蔬菜、全谷物和健康的蛋白质来源，同时可以选择合适的药物来控制脂肪分布。此外，也可以通过调节饮食和运动来改善脂肪代谢功能。如果口干不太好，可以试着使用特定的口腔清洁剂或口腔保护品来解决问题。最后，建议每天进行体育锻炼，这有助于提高身体素质和消化功能。 | 1.脂肪肝主要是由于肝功能不全导致的，可以服用中成药治疗效果不错的，建议你平时多注意休息，加强营养，保持心情愉快，定期复查就可以了。<br/>2.脂肪肝常见的症状有发热、食欲不振、消瘦、腹泻、腹痛、大便次数增多、血尿酸升高等。目前没有特效药物，对于脂肪肝需要根据病情进行调理，如果出现其他不适，应及时到医院就诊。 |  |
|  |          |  |  |
|  |          |  |  |

## 目前存在的问题

- 1、对训练数据的拟合还不够，这种类型的数据可能训练轮数得增多，或者将模型训练参数调大
- 2、里面有一些问题是多轮对话拆开的，这个由于没有标记所以难以判断
- 3、可以和英文的语料一起训练

## todo

- [x] 完成在2个epoch的Vicuna开始的continue_finetune的无指令微调
- [x]   完成在2个epoch的Vicuna开始的continue_finetune的带指令微调
- [x]   将相同的医疗数据融合
- [x]   从1个epoch开始微调
- [ ]   从头开始微调
- [ ]   融合英文的医疗问答数据

## 免责声明

此模型是基于大量语料库和算法模型进行训练的，并且在训练过程中可能存在偏差、错误和不完整的信息。因此，本项目提供的模型仅供参考和研究使用，作为领域数据微调的demo，并不能保证其准确性和可靠性，不能用于实际应用或决策。本项目不对使用预训练模型所产生的结果承担任何责任，也不对因使用预训练模型所产生的任何损失承担责任。使用者在使用预训练模型时应自行承担风险并进行自我验证。