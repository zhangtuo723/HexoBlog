---
title: 0-1实现一个Agent
date: 2024-07-28 16:27:29
tags:
cover: https://test-1301661941.cos.ap-nanjing.myqcloud.com/image-20240728165209268.png
---



# 从0到1实现一个Agent

## 什么是Agent

在LLM使用过程中，LLM只能作为一种问答工具只能为你所提出的问题给出文本形式的答案，而并不能为你执行真正的操作。Agent可以使得大模型能够和现有的工具进行联合，从而实现用户指令到大模型做出做出分析，Agent解析分析后执行相应Action的过程。通缩来说Agent就是建立在大模型和用户输入之间的一个代理系统，通过解析用户输入和大模型回复从而实现一个具体行为的系统，其中Agent可以对大模型的回复做出相应的动作后，将结果反馈给大模型使得大模型能逐步分析是否完成用户的指令，从而实现大模型对此次交互的反馈和思考，进一步的调整回复直至完成指令。

![image-20240728165209268](https://test-1301661941.cos.ap-nanjing.myqcloud.com/image-20240728165209268.png)

以一个简单的读写文件为例：第一步用户输入“帮我制定一份北京三天旅游计划，将结果写在res.txt中”，如果将指令直接发给大模型，大模型能够帮你制定计划，但是绝对不会帮你讲这计划写入本地文件中的re s.txt 中。所以完整的流程为，将指令输入到Agent中，其中Agnet中的工具包含了读写文件功能。Agent功过拼接用户指令以及现有工具的描述以及历史行为和结果（非第一次和大模型交互）和大模型要回复的格式（JSON）然后生成一个Prompt，将此Prompt输入给大模型，大模型会将自己的回复按照Prompt中的回复格式回复给Agent（这个过程非100%成功，取决为大模型的理解能力，有可能会回复出不符合格式的数据），然后Agent将这个回复解析完成后执行相应的动作并且将结果重新拼接为一个Prompt发送给大模型，重复这个过程直至大模型回复出完成任务的指令。



## 详细实现细节

接下来将从0-1完成一个简易版本的Agent的搭建，其中Agent的工具包括（搜索、读文件、写文件、追加文件）。

### 环境准备

* python 开发环境
* 大模型调用API
* 搜索API



### 具体实现

第一步首先完成main函数的编写，也就是系统启动的入口：

```python
def main():
    max_request_time = 10
    while True:
        query = input('请输入目标:')

        if query == 'exit':
            return

        agent_excute(query,max_request_time=max_request_time)

if __name__ == '__main__':
    main()
```

这是一个用户的循环输入功能，query为用户的输入指令，agent_excute 为agent的执行。

``` python
def agent_excute(query,max_request_time=10):
    cur_request_time = 0
    global agent_scratch

    while cur_request_time < max_request_time:
        cur_request_time +=1
        prompt = gen_prompt(query,agent_scratch)

        start_time = time.time()

        print('*****************{},开始调用llm........'.format(cur_request_time),flush=True)

        # 调大模型

        """

        sys_prompt:
        user_msg,assistant,history,
        """
        content = model.chat(prompt,chat_history=chat_history)

        end_time = time.time()
        
        print('*****************{},调用大模型结束，耗时：{}'.format(cur_request_time,end_time-start_time),flush=True)

        print("模型输出:",content)

        if content.startswith("```json"):
            content = extract_ast_to_bts(content)
            print('格式处理',content)
        try:

            response = json.loads(content)
        except Exception as e:
            print('调用大模型错误，即将重试。。。。',e)
            # chat_history.append([prompt,content])
            # prompt = '请直接输出json内容，不要带有```json 标识'
            continue
        """
        response:
        {
            "action":{
                "name" : "action_name",
                "args":{
                    "args name" : "args value"
                }
            },
            "thoughts":
            {
                "text" : "thought",
                "plan" : "plan",
                "criticism" : "criticism",
                "speak" : "当前步返回给用户的总结",
                "reasoning":""
            }    
        }
        """
        action_info = response.get('action')
        action_name = action_info.get('name')
        action_args = action_info.get('args')
        print('当前action name ：',action_name,action_args)

        if action_name == "finish":
            final_answer = action_args.get("answer")
            print("final_answer:", final_answer)
            break

        observation = response.get('thoughts').get("speak")
        try:
            func = tools_map.get(action_name)
            call_function_result = func(**action_args)
        except Exception as err:
            print("调用工具异常",err)

        agent_scratch= agent_scratch + '\n' + observation
        agent_scratch = agent_scratch + "\n: observation:{}\n execute action result: {}".format(observation,
                                                                                                call_function_result)
        
        chat_history.append([prompt,content])
```

为调用agent 的过程，tools_map为工具的map映射，gen_prompt为用户指令到prompt的生成函数,下图为prompt的制定。

```python
prompt_template = """
你是一个问答专家，你必须始终独立做出决策，无需寻求用户的帮助，发挥你作的优势，追求简答的策略，不要涉及法律的问题,并且要用中文。
    
目标:
{query}
限制条件说明:
    "仅使用下面列出的行为",
    "你只能主动行动，在计划行动时需要考虑这一点",
    "你无法与物理对象交互,如果对于完成任务或目标是绝对必要，则必须要求用户为你完成，如果用户拒绝，并且没有办法实现目标，则直接终止，避免浪费时间和精力。"
    "每次交互要判断上一步动作的状体，是否完成以及自我评判，，请综合考虑下是否本次动作要执行完成目标行为"
    "你也可以直接思考总结回应"

动作说明:这是你唯一可使用的动作，你的任何操作都必须通过以下操作实现：
{actions}


最佳实践的说明:
    "不断地回顾和分析你的行为，确保发挥你最大的能力",
    "不断地进行建设性的自我批评",
    "反思你过去的决策和策略，完善你的方案",
    "每个动作执行都有代价，所以要聪明高效，目的是用最少的步骤完成任务",
    "完成后几点执行完成行为"

下面是你上一次的思考和行为，以及行为的结果，你要根据之前的行为的结果进行思考，然后结合自己的判断执行后续行为:
{agent_scratch}

你应该以json格式响应,响应格式如下:
{response_format_prompt}
确保响应结果可以由python json.loads()成功加载。

"""

response_format_prompt = """
 {
            "action": {
                "name": "action name",
                "args": {
                    "args name": "args value"
                }
            },
            "thoughts":{
                "plan": "简单的描述短期和长期的计划列表",
                "criticism": "建设性的自我批评",
                "speak": "当前步骤，返回给用户的总结",
                "reasoning": "推理"
            },
            "observation": "观察当前任务的整体进度"
}
"""

def gen_prompt(query,agent_scratch):
    prompt = prompt_template.format(
        query=query,
        actions=action_prompt,
        agent_scratch=agent_scratch,
        response_format_prompt=response_format_prompt)
    return prompt
```

工具的描述和实现为：

```python

import json
import os

from tavily import TavilyClient

"""
    1、写文件
    2、读文件
    3、追加的方式写
    4、专业领域知识的获取(网络搜索)
"""

from langchain_community.tools.tavily_search import TavilySearchResults

def read_file(filename):
    filename = os.path.join("./data/", filename)
    if not os.path.exists(filename):
        return f"{filename} not exit, please check file exist before read"
    with open(filename, 'r', encoding="utf-8") as f:
        return "\n".join(f.readlines())


def append_to_file(filename, content):
    filename = os.path.join("./data/", filename)
    if not os.path.exists(filename):
        f"{filename} not exit, please check file exist before read"
    with open(filename, 'a') as f:
        f.write(content)
    return "append_content to file success."


def write_to_file(filename, content):
    filename = os.path.join("./data/", filename)
    if not os.path.exists("./data/"):
        os.makedirs("./data/")

    with open(filename, 'w', encoding='utf-8') as f:
        f.write(content)
    return "write content to file success."



def search(query):

    tavily = TavilySearchResults(max_results=5,search_depth='advanced')

    try:
        ret = tavily.invoke(input=query)
    except Exception as e:
        return e


    return ret



tools_info = [
    {
        "name": "read_file",
        "description": "读取文件行为",
        "args": [
            {
                "name": "filename",
                "type": "string",
                "description": "文件名"
            }
        ]
    },
    {
        "name": "append_to_file",
        "description": "追加内容到文件行为",
        "args": [
            {
                "name": "filename",
                "type": "string",
                "description": "文件名"
            },
            {
                "name": "content",
                "type": "string",
                "description": "追加到文件中的内容"
            }
        ]
    },
{
        "name": "write_to_file",
        "description": "写内容到文件行为",
        "args": [
            {
                "name": "filename",
                "type": "string",
                "description": "文件名"
            },
            {
                "name": "content",
                "type": "string",
                "description": "写入到文件中的内容"
            }
        ]
    },
{
        "name": "finish",
        "description": "完成用户目标行为",
        "args": [
            {
                "name": "answer",
                "type": "string",
                "description": "最后的目标结果"
            }
        ]
    },
    {
        "name": "search",
        "description": "搜索行为"
                       "when you are unsure of large model return",
        "args": [
            {
                "name": "query",
                "type": "string",
                "description": "要搜索的内容"
            }
        ]
    }
]

tools_map = {
    "read_file": read_file,
    "append_to_file": append_to_file,
    "write_to_file": write_to_file,
    "search":search
    # "search": search
}


def gen_tools_desc():
    """
    生成工具描述
    :return:
    """
    tools_desc = []
    for idx, t in enumerate(tools_info):
        args_desc = []
        for info in t["args"]:
            args_desc.append({
                "name": info["name"],
                "description": info["description"],
                "type": info["type"]
            })
        args_desc = json.dumps(args_desc, ensure_ascii=False)
        tool_desc = f"{idx+1}.{t['name']}:{t['description']}, args: {args_desc}"
        tools_desc.append(tool_desc)
    tools_prompt = "\n".join(tools_desc)
    return tools_prompt




tools_des = str(tools_info)

```

大模型的调用为：

```python

import os, json
import dashscope

import random
from http import HTTPStatus
# 建议dashscope SDK 的版本 >= 1.14.0
from dashscope import Generation



class Model(object):
    def __init__(self):
        self.max_retry_time = 3

    def chat(self, prompt, chat_history):
        cur_retry_time = 0
        while cur_retry_time < self.max_retry_time:
            cur_retry_time += 1

            message = []

            for his in chat_history:
                message.append({"role":"user","content":his[0]})
                message.append({"role":"assistant","content":his[1]}) # todo system -》assistant

            message.append({"role":"user","content":prompt})

            # print('--------------')
            # print(message)
            # print('--------')
            response = Generation.call(model="qwen-turbo",
                               api_key="apikey",
                               messages=message,
                               # 设置随机数种子seed，如果没有设置，则随机数种子默认为1234
                               seed=random.randint(1, 10000),
                               temperature=0.8,
                               top_p=0.8,
                               top_k=10,
                               # 将输出设置为"message"格式
                               result_format='message')
            if response.status_code == HTTPStatus.OK:
                try:
                    content = response['output']['choices'][0]["message"]["content"]
                    return content
                except Exception as e:
                    print("调用模型异常:{}".format(e))
            else:
                print('Request id: %s, Status code: %s, error code: %s, error message: %s' % (
                    response.request_id, response.status_code,
                    response.code, response.message
                ))
                print('------>!!!')
                print(message)
                exit(0)
                      
            
        return {}
    
```

### 效果演示

首先用户输入：帮我制定一份去北京的三天旅游计划，将计划写在 p.txt文件中。

```sh
*****************1,开始调用llm........
*****************1,调用大模型结束，耗时：2.585439920425415
模型输出: {
  "action": {
    "name": "search",
    "args": {
      "query": "北京三天旅游计划"
    }
  },
  "thoughts": {
    "plan": "首先，我需要找到一份关于北京三天旅游计划的信息。这将帮助我制定一个合理的旅游日程。",
    "criticism": "在寻找旅游计划时，我应该确保信息来源可靠，同时尽量避免重复信息，以提高效率。",
    "speak": "正在搜索北京三天旅游计划的相关信息。",
    "reasoning": "通过搜索，我可以快速获取关于北京旅游的信息，为制定旅游计划提供依据。"
  },
  "observation": "任务处于初始阶段，正在等待搜索结果。"
}
当前action name ： search {'query': '北京三天旅游计划'}
*****************2,开始调用llm........
*****************2,调用大模型结束，耗时：6.258164882659912
模型输出: {
  "action": {
    "name": "write_to_file",
    "args": {
      "filename": "p.txt",
      "content": "️【北京旅游】打卡帝都景点，三天玩转北京城! 🚶\n\n第一天:\n- 上午：天安门广场，感受祖国的心脏。\n- 中午：前门大街，品尝老北京特色小吃。\n- 下午：故宫，欣赏宫殿建筑的宏伟壮观。\n- 晚上：景山公园，俯瞰北京城的灯火辉煌。\n\n第二天:\n- 上午：八达岭长城，体验历史的雄伟。\n- 下午：天安门广场，毛主席纪念堂，了解历史。\n- 晚上：什刹海，享受京城夜晚的宁静。\n\n第三天:\n- 上午：圆明园，感受历史的沧桑。\n- 下午：前门大街，购物和品尝更多美食。\n- 晚上：返程。\n\n北京美食推荐：四季民福烤鸭店 (故宫店)。\n\n注意：提前预约门票，选择四环内地铁线附近的住宿。"
    }
  },
  "thoughts": {
    "plan": "已经整理出北京三天旅游计划，并准备将其写入文件p.txt。",
    "criticism": "在整理计划时，尽量保持内容的简洁和易读性，以便用户快速理解。",
    "speak": "已整理出北京三天旅游计划，并准备将其写入文件p.txt。",
    "reasoning": "通过阅读搜索结果，我已经提取了关键信息并整理成一个易于理解的旅游计划，现在只需将其写入文件即可完成任务。"
  },
  "observation": "任务接近完成，即将完成目标行为。"
}
当前action name ： write_to_file {'filename': 'p.txt', 'content': '️【北京旅游】打卡帝都景点，三天玩转北京城! 🚶\n\n第一天:\n- 上午：天安门广场，感受祖国的心脏。\n- 中午：前门大街，品尝老北京特色小吃。\n- 下午：故宫，欣赏宫殿建筑的宏伟壮观。\n- 晚上：景山公园，俯瞰北京城的灯火辉煌。\n\n第二天:\n- 上午：八达岭长城，体验历史的雄伟。\n- 下午：天安门广场，毛主席纪念堂，了解历史。\n- 晚上：什刹海，享受京城夜晚的宁静。\n\n第三天:\n- 上午：圆明园，感受历史的沧桑。\n- 下午：前门大街，购物和品尝更多美食。\n- 晚上：返程。\n\n北京美食推荐：四季民福烤鸭店 (故宫店)。\n\n注意：提前预约门票，选择四环内地铁线附近的住宿。'}
*****************3,开始调用llm........
*****************3,调用大模型结束，耗时：7.052416801452637
模型输出: {
  "action": {
    "name": "write_to_file",
    "args": {
      "filename": "p.txt",
      "content": "️【北京旅游】打卡帝都景点，三天玩转北京城! 🚶\n\n第一天:\n- 天安门广场：感受祖国的心脏。\n- 前门大街：品尝老北京特色小吃。\n- 故宫：欣赏宫殿建筑的宏伟壮观。\n- 晚上：景山公园：俯瞰北京城的灯火辉煌。\n\n第二天:\n- 八达岭长城：体验历史的雄伟。\n- 天安门广场：毛主席纪念堂：了解历史。\n- 什刹海：享受京城夜晚的宁静。\n\n第三天:\n- 圆明园：感受历史的沧桑。\n- 前门大街：购物和品尝更多美食。\n- 返回酒店休息。\n\n注意：提前预约门票，选择四环内地铁线附近的住宿。"
    }
  },
  "thoughts": {
    "plan": "已将整理出的北京三天旅游计划写入文件p.txt，下一步是完成任务。",
    "criticism": "在写入计划时，我尽量保持了内容的清晰和简洁，便于用户阅读。",
    "speak": "已将北京三天旅游计划写入文件p.txt，包含详细的景点安排和注意事项。",
    "reasoning": "根据搜索结果整理出的旅游计划，我将其整合并编写到文件中，完成了用户的目标。"
  },
  "observation": "任务已完成，北京三天旅游计划已成功写入文件p.txt。"
}
当前action name ： write_to_file {'filename': 'p.txt', 'content': '️【北京旅游】打卡帝都景点，三天玩转北京城! 🚶\n\n第一天:\n- 天安门广场：感受祖国的心脏。\n- 前门大街：品尝老北京特色小吃。\n- 故宫：欣赏宫殿建筑的宏伟壮观。\n- 晚上：景山公园：俯瞰北京城的灯火辉煌。\n\n第二天:\n- 八达岭长城：体验历史的雄伟。\n- 天安门广场：毛主席纪念堂：了解历史。\n- 什刹海：享受京城夜晚的宁静。\n\n第三天:\n- 圆明园：感受历史的沧桑。\n- 前门大街：购物和品尝更多美食。\n- 返回酒店休息。\n\n注意：提前预约门票，选择四环内地铁线附近的住宿。'}
*****************4,开始调用llm........
*****************4,调用大模型结束，耗时：5.425627946853638
模型输出: {
  "action": {
    "name": "finish",
    "args": {
      "answer": "已经成功制定了北京三天旅游计划，并将其写入文件p.txt。旅游计划包含详细的景点安排和注意事项，您可以查看该文件以获取更多信息。"
    }
  },
  "thoughts": {
    "plan": "短期目标是将整理出的旅游计划写入文件，长期目标是满足用户需求，提供完整的旅游计划。",
    "criticism": "在搜索和整理信息的过程中，虽然信息来源多样，但确保了信息的有效性和实用性，避免了不相关或低质量的信息干扰。",
    "speak": "已成功完成北京三天旅游计划的制定，并将其保存至文件p.txt中，您可以随时查看。",
    "reasoning": "通过搜索和整合多条信息源，形成了一个全面且实用的旅游计划，符合用户需求，同时采用高效的方法完成任务，保证了资源利用的最大化。"
  },
  "observation": "任务完成情况良好，旅游计划已成功制定并存储在指定文件中，满足了用户的需求。"
}
当前action name ： finish {'answer': '已经成功制定了北京三天旅游计划，并将其写入文件p.txt。旅游计划包含详细的景点安排和注意事项，您可以查看该文件以获取更多信息。'}
final_answer: 已经成功制定了北京三天旅游计划，并将其写入文件p.txt。旅游计划包含详细的景点安排和注意事项，您可以查看该文件以获取更多信息。
```

Agent经过4轮和大模型的交互完成了此次的用户指令，并且将结果写入到了p.txt中

![image-20240728172857450](https://test-1301661941.cos.ap-nanjing.myqcloud.com/image-20240728172857450.png)

至此完成0-1的agent构建

## Agent的能力分析

核型影响Agent的能力的有三点：

* 工具：此agent的实现中只用到了操作文件的工具和搜索功能，也就是所agent只能完成这些行为。
* prompt设计：prompt的设计会影响到大模型的理解，所以要在prompt中包含尽可能多的信息。
* 模型：模型本身的理解能力，模型本身的理解能力越强则对agengt的回复质量越高





## 总结和未来展望

对于大模型的落地，Agent的发展是必然的，Agent能够实现将现有工具和大模型分析能力的融合，从而能更广泛的应用到其他领域，实现更多场景的覆盖。



## 参考

[1]https://github.com/Yazooliu/agent_from_0t1
