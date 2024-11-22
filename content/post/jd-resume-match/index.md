---
title: 课程设计 - 智能简历解析系统
description: 课程设计项目 - 智能简历解析系统的实现记录
slug: jd-resume-match
date: 2024-11-21T20:24:56+08:00
math: true
image:
tags:
  - C#
  - Python
  - BERT
weight: 1
---

## 前言

果然，每次期末周临近的时候都会感觉全世界的 DDL 都突然吻了上来(厄厄...)。

先展示一下这次课设我的选题吧：

```text
智能简历解析系统
1. 简历导入与管理
（1）支持格式：系统应支持常见的简历格式，包括但不限于Word（.doc, .docx）、PDF和纯文本（.txt）。
（2）导入方式：允许用户通过拖拽、点击上传等方式单个或批量提交简历。
（3）存储与检索：可按导入日期，文件名称等方式查询简历。可根据用户的选择，打开并显示一份简历文件。
2. 简历解析与结构化
（1）内容提取：自动从简历中提取求职者信息，如姓名、联系方式、教育背景、工作经历、技能等。
（2）数据结构化：为用户提供多种简历解析数据的格式，包括但不限于JSON、CSV、XML。
（3）格式转换：支持将解析后的一种简历数据格式转换为其他格式。
3. 简历匹配与筛选
（1）关键词匹配：允许用户设定关键词，系统根据关键词自动筛选符合条件的简历。
（2）语义匹配：利用NLP技术，实现简历与职位描述之间的语义匹配，提高筛选的准确性。
（3）技能评估：根据求职者简历中的技能描述，评估其技能熟练度，并给出相应的评分。
4. 数据分析与报告
（1）统计分析：对简历数据进行统计分析，如求职者年龄分布、学历分布、技能分布等。
（2）报告生成：提供可视化报告，以图表形式展示分析结果，并配以文字介绍。
```

还有另一个做 stp 零件仿真的选题，感觉对我来说太灾难了，遂放弃...

然后，点明表扬：整个项目里只需要保证至少 30% 的代码是 C# 实现即可！泰好！

## 数据库设计

我们结合需求 1-(3) 以及需求 2-(1) 即可确定我们的数据库怎么设计，具体需要存储一些什么内容。

首先我们需要一个主表，存储简历的基本特征，比如：

- 上传时间，原文件名：方便检索
- 在服务器上的存储路径
- 文件格式：方便后续不同文件格式的内容读取
- 是否解析完成
- 解析后的json文本：方便后续人岗匹配

然后就是一些从表了，比如：

- 基本信息
- 教育经历
- 工作经历
- 技能
- 项目经历
- ...

通过外键 `resume_id` 和主表关联。

后文会补充添加一些其他的表。

## 需求实现记录

讲道理比较难实现的只有 解析和语义匹配 两部分，`NLP` 对非人工智能专业的鼠鼠还是太超前了。

### 简历导入与管理

这里需求(1)(2)可以放在一起考虑，首先考虑前端实现。

React的话可以使用 `useDropzone` Hook 实现拖拽上传以及文件格式限制。

```ts
const onDrop = useCallback(
    (acceptedFiles: File[], rejectedFiles: any[]) => {
        if (rejectedFiles.length > 0) {
            setError('只支持 PDF、DOC、DOCX、TXT 格式的文件');
            setTimeout(() => setError(null), 3000);
            return;
        }
        onFileUpload(acceptedFiles);
    },
    [onFileUpload]
);

const { getRootProps, getInputProps, isDragActive } = useDropzone(
    {
        onDrop,
        accept: {
            'application/pdf': ['.pdf'],
            'application/msword': ['.doc'],
            'application/vnd.openxmlformats-officedocument.wordprocessingml.document': ['.docx'],
            'text/plain': ['.txt']
        },
        multiple: true,
        onDragEnter: () => setDragCount(prev => prev + 1),
        onDragLeave: () => setDragCount(prev => prev - 1)
    }
);
```

然后后端实现就比较简单了，遍历 `context.Request.Form.Files` 然后验证文件类型之后，保存文件到服务器上即可。

```C#
async (HttpContext context) =>
{
    var formFiles = context.Request.Form.Files;

    if (formFiles.Count == 0)
    {
        return Results.Json(new { success = false, message = "No files were uploaded." }, statusCode: StatusCodes.Status400BadRequest);
    }

    string[] supportedFormats = [".doc", ".docx", ".pdf", ".txt"];
    var uploadedFiles = new List<string>();

    foreach (var formFile in formFiles)
    {
        // 检查文件格式
        if (!supportedFormats.Any(x => formFile.FileName.EndsWith(x, StringComparison.OrdinalIgnoreCase)))
        {
            return Results.Json(new { success = false, message = $"Unsupported file format: {formFile.FileName}" }, statusCode: StatusCodes.Status400BadRequest);
        }

        // 处理 & 保存文件
        var fileExtension = Path.GetExtension(formFile.FileName);
        var timestamp = DateTime.Now.ToString("yyyyMMddHHmmssfff");
        var newFileName = $"{Path.GetFileNameWithoutExtension(formFile.FileName)}-{timestamp}{fileExtension}";

        var filePath = Path.Combine(Directory.GetCurrentDirectory(), "Uploads", newFileName);
        await using (var stream = new FileStream(filePath, FileMode.Create))
        {
            await formFile.CopyToAsync(stream);
        }

        // 存入数据库

        // 将解析任务置入消息队列

        uploadedFiles.Add(newFileName);
    }

    return Results.Json(new { success = true, uploadedFiles }, statusCode: StatusCodes.Status200OK);
}
```

消息队列这里在后面的解析部分详细解释。

然后需求(3)就很简单了，我们数据库里已经存储了上传时间，文件名这些用于检索的字段，检索时筛选一下时间区间内的行 or 模糊查询一下文件名字段即可。

```sql
-- 按照时间筛选
SELECT * FROM resumes WHERE upload_date BETWEEN '开始时间' AND '结束时间';
-- 使用文件名查找
SELECT * FROM resumes WHERE file_name LIKE '%内容%';
```

可以进行一些简单的优化：
- 比如为 `upload_date` 字段添加索引
  ```sql
  ALTER TABLE resumes ADD INDEX idx_upload_date (upload_date);
  ```
- 为 `file_name` 字段添加全文索引(这里其实并不太需要)
  ```sql
  ALTER TABLE resumes ADD FULLTEXT(file_name);
  ```
  然后我们可以使用全文搜索代替原本的模糊查询
  ```sql
  SELECT * FROM resumes WHERE MATCH(file_name) AGAINST('内容' IN NATURAL LANGUAGE MODE);
  ```

### 简历解析与结构化

对于需求(1)的内容提取，查阅资料发现一般做法是使用 `NLP` 实现，对内容进行 `NER` 标识后做对应的提取工作。

感觉很难在 DDL 前速成并实现出来(悲)，所以我们这里另辟蹊径，给大模型提供相应的 `prompt`，让它帮我们完成解析工作，并严格规范其输出格式为 JSON。

这里我使用的 `prompt` 如下：

```markdown
请阅读以下简历文本，提取并整理出以下信息，且尽可能的使用原文信息，并以 JSON 格式返回：

- **个人信息**：姓名、手机号、邮箱、微信、博客、GitHub 等。
- **教育经历**（数组）：每个元素包括学校名称、时间区间、专业等。   
- **竞赛经历**（数组，可选）：每个元素包括竞赛名称、时间、奖项等。   
- **项目经历**（数组）：每个元素包括项目名称、技术栈、时间、介绍、亮点等。   
- **工作经历**（数组）：每个元素包括公司名称、职位、工作内容、时间等。   
- **专业技能**（数组）：每个元素包括技能描述、熟练度等。   
- **荣誉/证书**（数组，可选）：每个元素仅为一个荣誉/证书字符串。   
- **技术栈标签**（数组，可选）：每个元素仅为一个技术名称字符串。
   
**要求**：

- 确保提取的信息准确无误。
- 输出格式为 JSON。
- 时间格式统一为 `YYYY-MM` 或 `YYYY-MM-DD`。
    
**示例输出**：
 
\`\`\`json
{ 
    "personal_info": { 
        "name": "张三", 
        "phone": "13800000000",
        "email": "zhangsan@example.com", 
        "wechat": "zhangsan_wechat", 
        "blog": "https://zhangsan.blog", 
        "github": "https://github.com/zhangsan" 
    }, 
    "education": [ 
        { 
            "school": "清华大学", 
            "degree": "本科", 
            "major": "计算机科学", 
            "start_date": "2015-09", 
            "end_date": "2019-07" 
        } 
    ], 
    "competitions": [ 
        { 
            "name": "ACM 国际大学生程序设计竞赛", 
            "date": "2018-05", 
            "award": "银牌" 
        } 
    ], 
    "projects": [ 
        { 
            "name": "智能推荐系统", 
            "start_date": "2019-08", 
            "end_date": "2020-06", 
            "description": "开发了一款基于机器学习的智能推荐系统。", 
            "highlights": "提高了用户点击率20%" 
        } 
    ], 
    "work_experience": [ 
        { 
            "company": "百度", 
            "position": "高级研发工程师",
            "responsibilities": "负责搜索算法的优化和维护。", 
            "start_date": "2020-07", 
            "end_date": "2023-10" 
        } 
    ], 
    "skills": [ 
        { 
            "skill": "Python 编程",
            "proficiency": "精通" 
        } 
    ], 
    "certificates": ["CET-4", "CET-6"], 
    "tech_tags": ["Python", "机器学习"], 
} 
\`\`\`

以下是简历文本：

\`\`\`text
{resumeContent}
\`\`\`

```

测试了几次后发现效果还是不错的。

然后这里就有了一个新的问题，调用大模型做解析是一个比较耗时的操作，测试时大概需要 20s 左右才可以得到结果，显然让用户一直等待是不可取的。

所以在上个需求上传时我们将解析任务放入消息队列中等待，然后直接反馈结果给用户，将解析任务延迟到消费者中执行，这样用户就可以直接进行其他操作而非一直等待直到解析完成。

然后就是消费者的实现，比较显然了：

1. 调用大模型，等待响应
2. 处理大模型输出
3. 解析json
4. 修改主表 `parsed` 字段，将json文本存入主表(方便后文语义匹配)
5. 将解析后对应的内容存入对应的从表中

4和5放在一个事务中执行即可。

然后格式转换这里也就比较简单了，因为我们主表中存储了json文本，所以直接查询主表后用一些工具库将json文本转换成别的格式(如csv, xml)即可。

### 简历匹配与筛选

先看需求(1)，这里我们使用 ElasticSearch 实现，每次项目启动时，将主表中 `parsed` 字段为 `true` 行的 `resume_id` 和 `parsed_data` 字段映射到 ElasticSearch 中，然后利用 ElasticSearch 的 bool 查询即可。

这里使用 must 匹配(必须和若干个关键词全部匹配)和 should 匹配(和关键词中的任意一个匹配即可)。

然后对匹配结果在主表中通过 `resume_id` 字段进行筛选展示即可。

然后对于需求(2)，我的实现是综合考虑文本相似度和结构化数据相似度，然后进行加权计算。

这里需要对岗位JD先进行解析，和简历解析实现方式基本一致，不多赘述了。

- 文本语义匹配：我们将json处理为纯文本，然后利用 BERT 模型提取岗位JD和简历的语义向量，然后计算二者的余弦相似度，即可量化其语义相关性，注意对其归一化处理，余弦相似度取值范围为 [-1, 1]，我们对其进行归一化处理得到一个 [0, 1] 的值，值越大表示越相关。
- 结构化数据匹配：我们对岗位JD和简历中的学历，技能，工作经历等结构化信息，通过对比和比例得分的方式量化其匹配程度，同样的最后对其归一化处理，得到一个 [0, 1] 区间内的值。
- 最终加权计算：我们对上述两个值进行加权求和，得到一个 [0, 1] 区间内的值，值越大表示越相关，这里权的设置需要考虑二者哪个计算更准确，我这里的实现是 $ 0.6 \times TextSimilarity + 0.4 \times StructuredScore $，因为测试时结构化匹配的误差比语义相似度大的多。

完整流程如下：

1. 数据加载 & 初始化
   - 解析resume和jd的json文本为字典格式
   - 初始化模型和分词器
     - 加载停用词表，可以从这个仓库下载 [goto456/stopwords](https://github.com/goto456/stopwords)
     - 初始化 jieba 分词器
     - 初始化 BERT 模型，这里使用 `google-bert/bert-base-chinese`
2. 文本数据处理
   - 将resume 和 job 的 json 分别处理合并为两段纯文本，方便进行文本相似度计算
   - 文本预处理：使用 jieba 分词器进行分词，去除停用词，并拼接为一个无空格的字符串
     示例：
     ```python
     words = jieba.lcut(text)
     words = [word for word in words if word not in self.stopwords and word.strip() != '']
     return ''.join(words)
     ```
3. 文本相似度计算
   - 将预处理后的文本通过 BERT 编码，获取语义向量表示，提取 [CLS] token 的向量作为文本的语义表示
     示例：
     ```python
     inputs = self.tokenizer(text, return_tensors='pt', truncation=True, max_length=512).to(self.device)
     outputs = self.bert_model(**inputs)
     sentence_vector = outputs.last_hidden_state[:, 0, :].detach().cpu().numpy()[0]
     ```
   - 计算余弦相似度，并归一化处理
     $$
     \text{TextSimilarity} = \frac{\frac{\vec{vec1} \cdot \vec{vec2}}{\|\vec{vec1}\| \cdot \|\vec{vec2}\|} + 1}{2}
     $$
4. 结构化数据匹配
   - 从学历，技能，工作经历等考虑匹配，计算得分，我这里只考虑了三个维度，设定满分为3，每个维度1分，最后计算比例得分得到一个 [0, 1] 的得分。
   - 可以考虑尽可能多的维度，以获得更准确的匹配结果
5. 最终得分计算
   - 对文本相似度和结构化数据匹配的得分进行加权计算，得到最终的匹配得分
     $$
     \text{Score} = 0.6 \times \text{TextSimilarity} + 0.4 \times \text{StructuredScore}
     $$

最后的技能评估可以直接map映射一下 {熟练度,分数} 即可，因为json文本里已经很清楚了，这里不多赘述了。

### 数据分析与报告

这部分也比较简单，可以使用一些图表展示库进行可视化报告，比如 `echarts`。

后端只需要计算一下总数和不同类别的数量，向前端发送一个json描述这些信息，然后前端使用 `echarts` 进行展示即可。

文本描述可以使用老配方(bushi)，调用大模型即可，prompt大法好！

**That's all!**