---
title: 数据库agent
date: 2025/1/20   # 文章发表时间
categories: Tech # 分类
thumbnail: /images/数据库agent背景.jpg # 略缩图
---
昨天晚上和今天先把ollama接入了本地的jupyter,刚开始还想获取deepseek的api密钥来着，但是jupyter-ai他不支持，这太糟糕了，接入chatgpt现在咱还没那个体量，所以只能搞个本地部署的模型deepseek-r1，其实还是可以的。

然后就简单看了看秋芝老师用n8n构建ai agent的过程，可是还需要装docker沙盒，配置wsl，实在有点麻烦，而且也不是我现在的重点，所以我就仅仅是看了一遍视频，大致对应用层面的agent构建有了一个了解，但是总感觉还是缺了点什么，所以下午决定把Andrew和微软数据工程师联合推出的这门课“Build your own database agent"给实践一下，不过听完之后只有一个感觉，就是有些人在工业界可能是大亨，但确实不太适合讲课（狗头）。

其实他的这门课是逐步递进的一个过程，所以我就直接手撕最后的企业级数据库agent助手的成品，可以说是数据分析和ai agent的集成。

# 任务流程
```bash
CSV 文件
  ↓（Pandas）
DataFrame
  ↓（to_sql）
SQLite 数据库（test.db）
  ↓（你写的安全函数）
Python 函数（白名单 SQL）
  ↓（Function Calling）
Azure OpenAI
  ↓
SQL Agent 回答用户
```
现在我有的是一个csv文件，大致长这样，其实就和表格挺像的。
```bash
state,date,cases,deaths
CA,2022-01-01,1000,10
CA,2022-01-02,1200,12
NY,2022-01-01,800,8
```
# 第一步：CSV——>DataFrame(把文件变成数据)
```bash
import pandas as pd
csv_path = "./data/all-states-history.csv"
df = pd.read_csv(csv_path)
df = df.fillna(0)
```
这一步完成了从csv向dataframe的转换,最后的一行代码是因为CSV里可能有空值，但SQL中空值可能会出问题，所以会把所有空白位置置0，进行提前清洗。
# 第二步：DataFrame——>SQLite(生成数据库)
这里的一个.db文件就是一个数据库
```bash
from sqlalchemy import create_engine #SQLAlchemy = Python 和数据库之间的“翻译官”
db_path = "./db/test.db" #不存在也没关系，SQLite会自动创建
engine = create_engine(f"sqlite:///{db_path}")
df.to_sql(
    name="all_states_history",
    con=engine,
    if_exists="replace", #每次重新生成表，避免脏数据
    index=False #Pandas 索引不是业务数据，不要存进 SQL
)
```
OK到现在就可以使用SQL查他了。
# 第三步：构建数据库函数
这里我们的要求是AI不能越权，也就是AI不能直接生成SQL查询语句（本来其实这个也是一种进行数据管理的方式，也可以，但是使用function call会更好，所以这里我们就限制使用函数），而是要我写好函数的安全接口，AI只能用接口。
```bash
import sqlite3
def query_year_summary(state: str, year: int):
    conn = sqlite3.connect("./db/test.db") #打开数据库文件
    cursor = conn.cursor() #执行SQL的工具
    sql = """
    SELECT
        SUM(cases) AS total_cases,
        SUM(deaths) AS total_deaths
    FROM all_states_history
    WHERE state = ?
      AND strftime('%Y', date) = ?
    """  #表名固定，字段固定，防止SQL乱注入
    cursor.execute(sql, (state, str(year)))
    result = cursor.fetchone()
    conn.close()
    return {
        "state": state,
        "year": year,
        "total_cases": result[0],
        "total_deaths": result[1]
    } #返回json，大模型好理解    
```
# 第四步：把函数“注册”给 Azure OpenAI
现在AI才正式登场。
```bash
functions = [
    {
        "name": "query_year_summary",
        "description": "查询某州某一年的病例和死亡总数",
        "parameters": {
            "type": "object",
            "properties": {
                "state": {"type": "string"},
                "year": {"type": "integer"}
            },
            "required": ["state", "year"]
        }
    }
]
```
这一步实在告诉模型，能用什么函数，参数怎么填，填错就不能用。
# 第五步：Azure OpenAI 函数调用（SQL Agent 诞生）
```bash
from openai import AzureOpenAI
import json
client = AzureOpenAI(
    api_key="YOUR_KEY",
    azure_endpoint="https://YOUR_RESOURCE.openai.azure.com/",
    api_version="2024-02-15-preview"
) #连接 Azure OpenAI
messages = [
    {"role": "system", "content": "你是数据库助手，只能通过函数访问数据"},
    {"role": "user", "content": "统计 2022 年加州的病例和死亡总数"}
]
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages,
    functions=functions,
    function_call="auto"
)
```
那么在这一步中大模型给你的就不是SQL了，而是这样一个json文件
```bash
{
  "name": "query_year_summary",
  "arguments": {
    "state": "CA",
    "year": 2022
  }
}
```
# 第六步：你执行 SQL，模型不碰数据库
```bash
result = query_year_summary(**arguments)
```
# 第七步：模型总结结果
```bash
messages.append({
    "role": "function",
    "name": "query_year_summary",
    "content": json.dumps(result)
}) #把函数输出的查询结果再作为prompt的一部分喂给大模型
final = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages
)
```

到这里，我们就从一个csv文件开始，得到了一个可查询，可分析，防SQL注入，可扩展的SQL数据库agent助手，这里采用function call（函数）的核心便在于AI不是数据库的主人，而我们自己写的函数才是，AI要做的是去自己选择调用这些函数。
