---
title: interview-guide
date: 2022-11-03 10:08:16
index_img: /img/bg/ocean-road.jpeg
banner_img: /img/bg/tasmania.jpeg
tags: [Interview]
---

# 程序员面试

### 了解面试流程

#### 第一步：通过初筛，获得面试资格

这一阶段，最重要的就是**写好简历**。大多数程序员简历被挂不是因为他们做的项目不行或者能力不行，而是因为简历中对于曾经做过的Project包装不到位。关于如何写好简历，可以参考YangShun大佬的： [step-by-step guide here on software engineering resume preparation](https://www.techinterviewhandbook.org/resume/)。

在面试前通常会有一些前置的流程，包括 OA，QuiZ等等，这些通常是在HC比较紧张的时候，用于快速筛选候选人的一种方式吧。我得出该结论的推理是：我参加过一次秋招和一次春招：

- 在秋招的过程中，海投了十几家大大小小的互联网公司，大多数时候都是HR直接约面的，一些OA我放着没做后来HR也主动打电话过来约面了。（此时在中国）
- 在春招的过程中，海投了5-6家海外的互联网公司，除了TikTok是内推直接约面的之外，其他的公司基本都有OA。（此时在新加坡）。

虽然可能国内国外的互联网招聘流程不太一样，但我推测影响这一流程的因素包括三个方面：

1. 候选人是否是通过内推进入的招聘流程（能够得到职级越高的员工内推，后续的流程也更加顺利）；
2. 部门的HC是否充足，秋招的HC 应该会远远大于春招。
3. 候选人的能力，推测该因素的优先级是远远小于前两个。

基于以上，对于OA 和 Quiz 的频率定义为：**Occasionally**.

#### 第二步：进入电面环节

- 手撕代码

  不要依赖编译器来调试自己的代码，而需要经过详细的推敲和耐心的编码，在短时间内写出正确、高效的代码。

Refer：https://www.techinterviewhandbook.org/coding-interview-cheatsheet/

#### 第三步骤：进入线下面环节

- 到了这一步已经快成功了，通常公司会支付海外候选人的机票和住宿。
- 这一步常见的面试问题包括：Coding、System Design 和 Behavior，而且因为是线下可能需要花费较长的时间，并且需要自带电脑，在Whiteboard 上做设计，并且在短时间内编码。
  - 准备好开发环境。
- 选择一门合适的编程语言：建议选择常用的，对于数据结构有较完整封装的语言，包括：Python、Java、C++。

### 面试前的准备

- Code Interview: 最重要的一环，抱紧力扣大腿，充分了解对于所选编程语言的应用。比较建议的时间是 2-3个月准备编程应试，每天2-3小时。对于我而言，每天保持力扣的刷题更新，面试前针对性的刷目标公司的题目即可。

- Mock Interview (With Google & FaceBook Engineers): [interviewing.io](https://iio.sh/r/DMCa)

- System Design

  - [ByteByteGo](https://bytebytego.com/?fpr=techinterviewhandbook)
  - [Grokking the System Design Interview" by Design Gurus](https://designgurus.org/link/kJSIoU?url=https%3A%2F%2Fdesigngurus.org%2Fcourse%3Fcourseid%3Dgrokking-the-system-design-interview)

- Behavior Interview

  这一环节主要是HR为了了解候选人在上一家公司的经历，以及候选人对于一些工作上遇到的问题的解决方法（冲突处理），或者职业生涯规划等。

  - STAR Format:
    - Situation: 关于做某个任务的BackGround。
    - Task：需要解决什么具体的问题，关注这个Task 的 影响面、挑战、产出。
    - Action：做了什么事情去实现这个Task，并且描述在解决Task中遇到的一些选择。
    - Result：描述Action 的产出，以及从这个Action中学习到了什么。
  - 资源：
    - [Top-30 Behavior Question](https://www.techinterviewhandbook.org/behavioral-interview-questions/)
    - [BI Guide](https://www.techinterviewhandbook.org/behavioral-interview/)

- 学习如何Negotiate：

  - [negotiation strategies](https://www.techinterviewhandbook.org/negotiation/) 
  - [software engineer compensation](https://www.techinterviewhandbook.org/understanding-compensation/).

### 如何写好简历

简历不是一次性的，它就像一个工程，有了良好的架构之后，扩展性和可用性都能持续提升，每一次完善简历、投递简历的过程就是一次 CI / CD。

#### 建立 ATS (Applicant Tracking System) 友好的简历模板

> 大多数公司都有对简历的自动解析系统，并且会根据解析出来的选项自动过滤掉一些候选人的简历。

简历中内容的顺序

|       Section        | Heading Name                                                 |
| :------------------: | ------------------------------------------------------------ |
| Professional Summary | Use Resume Headline as section title, e.g: Senior Software Engineer at Google with 5 years of experience leading teams. |
| Contact Information  | Contact infos                                                |
|        Skills        | Programming Language, Frameworks                             |
|      Experience      | Work Experience                                              |
|    **Education**     | Less than 3 years, should put school info **First**          |
|       Projects       | Projects                                                     |

#### 写好简历的Summary

> Summary 就像一篇论文的 Abstract，目标是抓住Manager的眼球，主要是围绕自己的 ”卖点“ 和 JD的匹配度 来写。

- 写大纲：描述自己对于JD中要求具体满足/契合的点。

- 写标题：(Example)

  ##### [Software Engineering Lead](https://www.techinterviewhandbook.org/resume/#software-engineering-lead)

  Software Engineer with X years of experience in back end, scaling complex distributed systems, and various cloud platforms. Led over 5 engineering teams with an average size of 6 members across two companies and mentored over 20 junior members.

  ##### [Senior at University X](https://www.techinterviewhandbook.org/resume/#senior-at-university-x)

  Senior Year student at University X with a focus on Artificial Intelligence and Machine Learning (ML). Interned at X companies and worked on full stack development and ML engineering roles.

#### 写清楚简历的Contact Infomation

**Must Have**

- Name
- Personal Phone Number
- Location
- Email Address
- LinkedIN

**Recommend**

- GitHub Page
- Personal Website Page
- Competitive Coding Profile

#### 关于工作经历的描述

按照熟悉程度和 倒序时间顺序来写就职的公司，需要包括以下信息：

- The company, location, title, duration worked following this structure

> [Company or Organization], [Location] | [Job Title] | [Start and end dates formatted as MM/YYYY]

Example

> Facebook, Singapore | Front End Engineering Lead | 08/2018 - Present

列出在公司中最主要的贡献

- Scope of job and skills required

- Accomplishments listed following this structure

  - > [Accomplishment summary] : [Action] that resulted in [quantifiable outcome]

#### 润色简历

- Less is More! 内容尽量局限于一页纸，高亮一些重点的产出比盲目列举一些平庸的产出更重要。
- 从 JD 中摘取一些关键词应用到简历中，可以将关键词放在 Summary 和 教育经历中。进一步地，需要根据JD 分析对于候选人能力的优先级顺序，优先级高的关键词应该更多地出现。
  - 一些TIPs：
    - 多找几份相关岗位的JD 从中提取共同的信息；
    - 可以Paste到 MarkDown中，通过关键词搜索来找到最常用的关键词；

#### Last But Not Least

- 认真对待申请表，有些公司需要填写完申请表格之后，候选人的简历才能被 HR 看到；
- 不要申请一家公司的多个职位。