---
type: summary
status: active  
thesis: 
module: markdown撰写规范
period:
tags:  
  
topic:  
- rule
created: 2026-04-28  
---

# Markdown管理规则
## 文件夹只管“大类”
```
Segmentation
Inpainting
Pathology Simulation
XR Display
Attachments
```

## 文件命名规则
```
P_             项目
T_             测试文档
S_             小结 Summary
E_             实验 Experiment
Paper_         论文资料
Concept_       概念笔记，可不用前缀
```

## type 规则
`type` 用来区分笔记类型
```
type: thesis
type: project
type: plan
type: test
type: summary
type: experiment
type: paper
type: concept
type: meeting
type: board
```

## status 规则
`status` 用来表示状态。
```
status: active  
status: planned  
status: testing  
status: done  
status: archived
```

## tags 规则
标签不要太多，建议稳定使用这些
不要每篇笔记都随便打很多标签 ，比如 `#重要`、`#临时`、`#待看`、`#有用` 这种后期会混乱
```
thesis  
project  
test  
summary  
experiment  
paper
```
## topic 规则
`topic` 用来表示具体主题。
```
rule # 规范
model-training  
data-cleaning  
frontend  
testing  
system-design
pathology simulation
github code based #对github仓库代码的解析
math-learning  #数学公式原理的解析
```
可理解为：
```
tags = 大类  
topic = 具体主题
```



