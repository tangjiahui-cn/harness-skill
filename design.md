## 目录规则
.harness/temp：存放本次需求缓存
.harness/history: 存放历史生成记录

## 初始化信息生成

将本次的用户需求总结为不超过10个字的中文描述，对应字段：name
生成一个随机 UUID（长度为10英文），对应字段：uuid
生成本次需求id：{name}_{uuid}，对应字段: vId

然后写入状态文件 .harness/temp/{vId}/state.json

## 命名规则
spec命名：{name}_{uuid}_spec.md
plan命名：{plan}_{uuid}_plan.md


## 运行流程

1、(需求输入)用户输入需求
2、（初始化）初始化目录、状态文件（.harness/temp/{vId}/state.json）
3、（spec生成）根据项目上下文和用户需求生成 spec文件到 .harness/temp/{vId}/spec.md。生成后询问用户（1继续/2重新生成/3补充信息/4使用 AI 校验）.使用 子agent形式对 spec.md 校验并生成到 .harness/temp/{vId}/spec-suggest.md，然后告诉父 agent 对 .harness/temp/{vId}/spec-suggest.md 处理后删除该文件，并再次使用 子agent 进行校验，直到没有问题，最多5轮。然后再次询问用户（1/2/3/4）。用户选择继续后，进行下一步，并更新 state.json。
4、（plan生成）根据 .harness/temp/{vId}/spec.md 生成plan文件到 .harness/temp/{vId}/plan.md。生成后询问用户（1继续/2重新生成/3补充信息/4使用 AI 校验）.使用 子agent形式对 plan.md 校验并生成到 .harness/temp/{vId}/plan-suggest.md，然后告诉父 agent 对 .harness/temp/{vId}/plan-suggest.md 处理后删除该文件，并再次使用 子agent 进行校验，直到没有问题，最多5轮。然后再次询问用户（1/2/3/4）。用户选择继续后，进行下一步，并更新 state.json。
5、（执行和测试）AI 按照每一步进行执行，每一步完成都提交 git commit，并更新 state.json。gitcommit内容类似于 feat：<本次修改内容总结，不超过20个字（中文）>。所有步骤完成后，提示用户（1验收通过 2用户补充信息）。用户可以选择回退到之前的某一步，清空对应记录，并修改 state.json。
6、（验收后询问推送）。询问用户：验收通过是否推送远程代码（1推送 2不处理）
