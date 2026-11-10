# gpt-image-2 中转 API：国内直连 AI 生图接口，2.6 的 Flare 与 Sunburst 都能调

`gpt-image-2` 是 OpenAI 这一代图像模型的统称，2.6 这一版拆成了两个模型：`Flare` 和 `Sunburst`。两者都新增了 GIF 动图和 SVG 矢量输出，生成速度官方说法是提升 50%，Flare 在低画质下最快约 6 秒。对做电商图、社媒配图、表情包的人来说，这是比较实际的变化。

官方 `api.openai.com` 在国内不好连，这个仓库说明的是怎么通过 Token173（https://token173.net）的国内入口调这套图像接口，请求格式和官方保持一致。

## 模型版本

| 模型 ID | 用途 |
|---|---|
| `gpt-image-2.6-flare` | 2.6 的快速迭代版，做草图、多方案、日常出图、交互改图 |
| `gpt-image-2.6-sunburst` | 2.6 的细节版，复杂画面修改、最终素材，编辑一致性更好 |
| `gpt-image-2.5-sunburst` | 上一代细节版，稳，适合返工少的批量交付 |
| `gpt-image-2.5-flare` | 上一代快速版，比 2.6 便宜 |
| `gpt-image-2` | 基础版，便宜，跑草图合适 |

2.6 对"场景规划"和"画面渲染"是分开控制的，同一个提示词改复杂度档位就能出不同画质，不用整段重写 prompt。

## Flare 和 Sunburst 怎么选

| 你的情况 | 选哪个 |
|---|---|
| 先出十几张草图挑方向 | `gpt-image-2.6-flare`，迭代快 |
| 海报、商品图的排版对齐 | `gpt-image-2.6-flare`，官方说布局优化在这一侧 |
| 同一张图要连着改五六轮，人物产品不能走样 | `gpt-image-2.6-sunburst` |
| 最终交稿的精细视觉素材 | `gpt-image-2.6-sunburst` |
| 出 GIF 表情包、循环动画 | 先 Flare 出静态图，再转 GIF，一致性更好 |

有一点要说清楚：官方没有公布两者的独立速度分和质量分，所以别理解成"一个最快一个最好"。实测里也有人觉得 Flare 的速度提升不明显，这跟画质档位、服务端排队都有关系，自己跑一批看更准。

## 文生图

```bash
curl https://token173.net/v1/images/generations \
  -H "Authorization: Bearer sk-你的令牌" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-image-2.6-flare",
    "prompt": "白色陶瓷咖啡杯放在浅灰色水泥台面上，晨光从左侧窗进来，浅景深",
    "size": "1024x1024",
    "n": 1
  }'
```

Python：

```python
from openai import OpenAI
import base64

client = OpenAI(api_key="sk-你的令牌", base_url="https://token173.net/v1")

r = client.images.generate(
    model="gpt-image-2.6-sunburst",
    prompt="白色陶瓷咖啡杯放在浅灰色水泥台面上，晨光从左侧窗进来，浅景深",
    size="1024x1024",
)

with open("cup.png", "wb") as f:
    f.write(base64.b64decode(r.data[0].b64_json))
```

## 改图与扩图

```python
from openai import OpenAI

client = OpenAI(api_key="sk-你的令牌", base_url="https://token173.net/v1")

r = client.images.edit(
    model="gpt-image-2.6-sunburst",
    image=open("cup.png", "rb"),
    prompt="把背景换成暖木桌面，杯子不变",
    size="1024x1024",
)
```

扩图就是把画布改成非正方形尺寸，模型会自己补画边缘。

## 提示词怎么写更省

从实际出图看，四段式比一句话靠谱：

1. 主体：什么物件、什么材质、什么状态
2. 环境：放在哪、光源从哪来
3. 镜头：景别、焦段、景深
4. 限制：不要出现文字、不要出现多余手指

比如"白色陶瓷咖啡杯，浅灰水泥台面，晨光侧入，50mm 浅景深，画面不要出现文字"——出图命中率比"一杯咖啡"高得多。

负面词用英文写在 prompt 末尾也管用，`no text, no watermark, no extra fingers`。

## 计费

图像按张计费，不同分辨率和质量档价位不同，具体在后台模型页能看到当前单价。账户里 2 元换 1 美元额度，注册送 0.2 美元，出几十张测试图没问题。

批量出图的省钱做法：草图用 `gpt-image-2.6-flare` 或基础版 `gpt-image-2` 跑，定稿再用 `gpt-image-2.6-sunburst` 出高清，别一步到位。

## 常见问题

**Q：返回的是 b64 还是 URL？**
默认返回 base64，和官方一致。要 URL 就在请求里加参数，或者自己存下来传 CDN。

**Q：一次能出几张？**
`n` 参数控制，一张一张出更稳，批量并发建议不超过 4。

**Q：中文提示词行不行？**
行，模型支持中文。但精确控制构图时英文更稳，尤其涉及镜头语言。

**Q：2.6 的 SVG 怎么拿？**
在请求里指定矢量输出参数，返回 SVG 文本。适合图标、logo 这类需要放大的素材。目前 SVG 生成比出图慢不少，稳定性也一般，复杂矢量图建议多试几次。

**Q：GIF 能出透明底吗？**
能。背景色要和主体颜色拉开，不然抠出来边缘会糊。批量做表情包的话，先出静态图再转 GIF，一致性更好。

**Q：出图被审核拦了怎么办？**
换成更中性的描述，避免写具体人名、品牌 logo、真实人物肖像。

## 相关

- 文档站：https://docs.token173.net
- 图像接口总览：[gpt-image-api](https://github.com/Token173/gpt-image-api)
- 谷歌图像模型：[Nano-Banana-api](https://github.com/Token173/Nano-Banana-api)
- 视频生成：[veo-proxy-](https://github.com/Token173/veo-proxy-)
