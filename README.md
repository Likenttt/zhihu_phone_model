
- [分析知乎用户用的啥手机](#-----------)
  * [获取数据](#----)
      - [核心命令获取与分析](#---------)
      - [编写脚本采集数据](#--------)
  * [处理数据](#----)
    + [匹配删除 `a` `span` 等多余的 `html` 标签](#------a---span--------html----)
    + [稍稍手动剔除与我们的目标不符的答案](#-----------------)
    + [`jieba` 分词，去除无关主题的高频词汇](#-jieba----------------)
  * [生成词云](#----)





# 分析知乎用户用的啥手机

以问题 [现在看见这个问题的你们都在用什么手机？](https://www.zhihu.com/question/368320511) 下的回答为样本

## 获取数据

#### 核心命令获取与分析

![获取curl命令](/img/image-20200317232408743.png)

截止 2020.3.17 23:27，该答案下共有 6326 个回答，需要翻 317 页。

命令中关键分页参数为 `limit=5&offset=5` 。其中 `limit` 指的是页长，`offset` 则是偏移量。经过尝试，知乎 `pc` 问题页一次分页上限为 20。

通过 [Convert curl syntax to Python...](https://curl.trillworks.com/#python) 可以将 `curl` 命令转化为 对应的 `python` 代码。

#### 编写脚本采集数据

```bash
import requests
import time
import random

def curl_command(offset):
    headers = {
        'authority': 'www.zhihu.com',
        # 此处省略很长内容
    }
    params = (
        ('include',
         'data[*].is_normal,admin_closed_comment,reward_info,is_collapsed,annotation_action,annotation_detail,collapse_reason,is_sticky,collapsed_by,suggest_edit,comment_count,can_comment,content,editable_content,voteup_count,reshipment_settings,comment_permission,created_time,updated_time,review_info,relevant_info,question,excerpt,relationship.is_authorized,is_author,voting,is_thanked,is_nothelp,is_labeled,is_recognized,paid_info,paid_info_content;data[*].mark_infos[*].url;data[*].author.follower_count,badge[*].topics'),
        ('limit', '20'),
        ('offset', offset),
        ('platform', 'desktop'),
        ('sort_by', 'default'),
    )

    response = requests.get('https://www.zhihu.com/api/v4/questions/368320511/answers', headers=headers, params=params)
    return response


offset = '0'
filename = open("zhihu_answer_phone_model.txt", "w")
response = curl_command(offset)
data_array = response.json()["data"]
while len(data_array) > 0:
    print(data_array)
    for item in data_array:
        print(str(item["author"]["gender"]) + "\t\t" + item["content"])
        filename.write(str(item["author"]["gender"]) + "\t\t" + item["content"] + "\n")
    offset = str(int(offset) + len(data_array))
    response = curl_command(offset)
    data_array = response.json()["data"]
    time.sleep(random.randint(0, 9))

filename.close()

```

最终实际获得答案数为 6593，粗看数据有重复的，估计因为获取数据期间有用户回答了问题，导致偏移量有错位。

![image-20200318154640866](/img/image-20200318154640866.png)

##  处理数据

知乎大神们的答案形式不一，有的开门见山直言型号，有的畅叙平生追思旧机，针对不便统计的长答案进行手动校正。此处直接用 Pycharm 自带的正则替换。误伤有图的需要为推导出答案的回答实在难免。

### 匹配删除 `a` `span` 等多余的 `html` 标签

1. `<a.+?href=\"(.+?)\".*>(.+)</a>` 匹配 `a` 标签 。同理可以针对  `figure` 、`frame` 操作。误伤内涵图片数据难免。
2.  `</p><p>(.+)</p>`  有些用户第一句是谢邀，这就导致真正的关键回答会被删除。放弃。

![image-20200318155832240](/img/image-20200318155832240.png)

全部转小写字母。

### 稍稍手动剔除与我们的目标不符的答案

![image-20200318160110272](/img/image-20200318160110272.png)

### `jieba` 分词，去除无关主题的高频词汇

```python
import jieba
import numpy as np
import pandas as pd

filename="/Users/kent/Downloads/zhihu_phone_model/src/main/java/com/example/zhihu_phone_model/zhihu_answer_phone_model_shorten.txt"
data=open(filename,'r').readlines()
def process(data):
    m1=map(lambda s:s.strip("\n"),data)
    cut_words=map(lambda s:list(jieba.cut(s)),m1)
    return list(cut_words)

cut_words=process(data)

total_words=[]
for each in cut_words:
    total_words.extend(each)

n=np.unique(total_words,return_counts=True)
s=pd.Series(data=n[1],index=n[0])
result=s.sort_values(ascending=False)
for i, v in result.iteritems():
    if v > 30:
        print('index: ', i, 'value: ', v)

```

看下高频词，剔除品牌相关的汉字、数字等，然后批量删除。

```python
#!/usr/bin/python3
# -*- coding: UTF-8 -*-

words="是 买 还 有 在 好 ！就 也 都 版 很 年 和 换 > ？ 说 这 这个 你 . 觉得 一直 挺 自己 什么 但是 过 多 吧 月 一年 错 快 有点 使 卡 三年 它 已经 吗 < 性价比 因为 要 刚 系列 啊 入手 ) 比较 被 再 再战 上 机 拍照 看 人 流畅 够 又 屏 还有 但 知道 学生 最 准备 回答 个人 开始 太 得 指纹 = ; 打算 个 等 哈哈哈 后 当时 一部 国产 ‍ 话 真香 打游戏 后悔 出 然后 去 性能 可能 体验 高 所以 游戏 想换 黑色 其 让 党 好看 支持 主要 比 王者 打 钱 摔 中 一样 非常 虽然 高考 满 还行 出来 哈哈 依然 ： 白色 看到 便宜 香 老 下 应该 推荐 而且 二手 之后 平板 ～ 版本 旗舰 差 工作 美版 暑假 本人 坏 做 呢 主力 一次lus 手持 方便 还错 换个 们 穷 如果 玩 习惯 其实 以前 型号 毕竟 四年 电量 玩游戏 户 坚持 那 品牌 正在 起来 确实 掉 智能 九 哦 几年 为什么 备机 机子 像素 左右 只有 配置 发热 送 本来 吃 最近 最后 更新 运行 一台 超级 : 拥有 路过 行 依 花 刚刚 完全 质量 需求 刚出 刚买 算 一定 方面 发布 真 鸡 或者 买个 半 表示 那个 后来 今年 全面 来说 适合 原因 以后 结果 作为 当年 谁 那种 贼 充 狗头 重要 ️颜值 为啥 当初 只能 米粉 有时候 淘宝 各种 月份 那么 至今 加 这部 偶尔 里 只 前买 曲面 无 毛病 无聊 购买 一天 还好 无线 这想 手里 差多 换机 基本 好像 好几年 有些 要求 两个 最好 一般 红色 家里 一批 骁龙 是 买 还 有 在 好 ！就 也 都 版 很 年 和 换 到 能 说 这 个 你 觉得 一直 挺 自己 什么 但是 过 多 时候 想 喜欢 月 一年 快 有点 已经 吗 他 它 性价比 系列 啊 入手 目前 手持 跟 一次 呢 本人 暑假 差 工作 版本 ～ 二手 平板 之后 而且 爱 推荐 摄像头 日常 功能 怎么 把 应该 老 下"
word_array=words.split(" ")
print(word_array)
in_file = open("/Users/kent/Downloads/zhihu_phone_model/src/main/java/com/example/zhihu_phone_model/zhihu_answer_phone_model_shorten.txt", "r")
out_file = open("/Users/kent/Downloads/zhihu_phone_model/src/main/java/com/example/zhihu_phone_model/zhihu_aswer_purified.txt",'w')

line = in_file.readline()
while line:
    for keyword in word_array:
        line=line.replace(keyword,"")
    out_file.write(line)
    line = in_file.readline()

out_file.close()
in_file.close()
```

## 生成词云

```python
import chardet
import jieba
from wordcloud import WordCloud
import matplotlib.pyplot as plt

text = open("/Users/kent/Downloads/zhihu_phone_model/src/main/java/com/example/zhihu_phone_model/zhihu_aswer_purified.txt").read()
text+=' '.join(jieba.cut(text,cut_all=False)) # cut_all=False 表示采用精确模式
# 设置中文字体
font_path = '/System/Library/Fonts/Supplemental/Songti.ttc'
# 设置中文停止词
stopwords = set('')
stopwords.update(['128g','pro','64g','e','0','但是','一个','自己','因此','没有','很多','可以','这个','虽然','因为','这样','已经','现在','一些','比如','不是','当然','可能','如果','就是','同时','比如','这些','必须','由于','而且','并且','他们'])

wc = WordCloud(
       font_path = font_path, # 中文需设置路径
       margin = 2, # 页面边缘
       scale = 2,
       max_words = 400, # 最多词个数
       min_font_size = 4, #
       stopwords = stopwords,
       random_state = 42 ,
       max_font_size = 70,
       )
wc.generate(text)
# 获取文本词排序，可调整 stopwords
process_word = WordCloud.process_text(wc,text)
sort = sorted(process_word.items(),key=lambda e:e[1],reverse=True)
print(sort[:50]) # 获取文本词频最高的前个词
# 存储图像
wc.to_file('zhihu_phone_model.png')
# 显示图像
plt.imshow(wc,interpolation='bilinear')
plt.axis('off')
plt.tight_layout()
plt.show()
```

结果

![image-20200319005551962](/img/image-20200319005551962.png)

像华为荣耀这种应该算作是荣耀的，etc. 手动标记下。

修正结果如图

![image-20200319010301784](/img/image-20200319010301784.png)



参考：

[中文词云生成](https://cloud.tencent.com/developer/article/1373142)

