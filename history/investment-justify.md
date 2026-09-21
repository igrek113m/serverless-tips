はい。むしろビジネスオーナー向けには、**完成済みのROIグラフを見せるより、「その場で条件を変えて結果がどう動くか」を見せる**方が効果的です。

私ならこれを **Interactive Evidence（対話型エビデンス）** として、4段階で組みます。

## 1. Google Trends：まず「市場の関心」を本人に操作してもらう

Google Trendsでは、期間、地域、比較対象をその場で切り替えられます。Google自身も、データは検索全体に対する相対的関心として正規化され、0〜100で表示されると説明しています。つまり絶対的な市場規模ではありません。([Google サポート][1])

例えばプレゼン中に、

**Serverless / AWS Lambda / Kubernetes / EC2**

などを入れて、

```text
世界 → 日本 → 中国
5年間 → 10年間
Web検索 → News
```

と切り替えます。

ここで重要なのは、

> 「Serverlessが人気だから投資しましょう」

とは結論づけないことです。

むしろ、

> 「地域や時期によって関心がどう変化する技術なのか、一緒に見てみましょう」

と操作してもらう。

Google Trendsには検索語そのものと、複数言語等を束ねた「Topic」の違いもあるので、国際比較ではTopicを選ぶ方が適しているケースがあります。([Google サポート][2])

[Google Trends Explore](https://trends.google.com/explore?utm_source=chatgpt.com)

---

## 2. BuiltWith：検索人気から「実際に使われているか」へ進む

ここから説得力が一段上がります。

BuiltWithでは、Webサイトから検出された技術について、

* 現在利用しているサイト
* 過去に利用していたサイト
* 国
* トラフィック規模
* Technology Spend

などをインタラクティブに絞れます。

現在もAWS Lambdaについてライブサイト、過去利用サイト、国別などのデータを提供しています。日本だけに絞ったビューもあります。([BuiltWith Trends][3])

ここで、

```text
Google Trends
「みんな検索している」

       ↓

BuiltWith
「実際に使われている」

       ↓

AWS Pricing Calculator
「では経済性はどうなのか」
```

というストーリーにします。

これはかなり強いです。

ただしBuiltWithはWebから検出可能な技術が対象なので、**企業内部のLambda利用を含むServerless市場全体のシェアではありません**。したがって「採用状況を示す一つの観測窓」と位置づけます。

[BuiltWith AWS Lambda Trends](https://trends.builtwith.com/Web-Server/AWS-Lambda?utm_source=chatgpt.com)

---

## 3. GitHub Topics：「開発者エコシステム」をその場で見せる

GitHubもかなり使えます。

現在GitHubの `serverless` topicには多数の公開リポジトリがあり、言語別に絞ったり、更新日時順に並べたりできます。`aws-lambda` も独立したTopicとして大量の公開プロジェクトがあります。([GitHub][4])

例えば、

| 操作                 | ビジネス側に見せられること |
| ------------------ | ------------- |
| `serverless` を検索   | エコシステム規模      |
| Pythonだけにする        | 自社技術との親和性     |
| Recently updated   | 現在も活動があるか     |
| AWS Lambdaへ変更      | ベンダー別活動       |
| Azure Functionsへ変更 | マルチクラウド化      |

これによって、

> 「Serverlessというマーケティング用語が流行っている」

から、

> 「実際に開発者がコードを書き続けている技術領域である」

へ議論を進められます。

[GitHub Serverless Topic](https://github.com/topics/serverless?utm_source=chatgpt.com)

---

# 4. AWS Pricing Calculator：ここから「経済効果」に入る

ビジネスオーナー向けなら、**ここが一番重要です**。

AWS Pricing Calculatorを開き、その場でLambda等の条件を変えます。AWS自身もPricing Calculatorをアーキテクチャのコスト見積りツールとして提供しています。([AWS プライシングカルキュレーター][5])

例えば最初に、

```text
100万 request/月
512 MB
平均実行時間 300 ms
```

を設定する。

次に、

```text
100万
↓
1,000万
↓
1億 request
```

と変えます。

ビジネスオーナーには技術的なGB-secondより、

> **「ユーザー数が10倍になるとITコストはいくら増えるのか？」**

を見てもらいます。

さらにEC2/EKS側について、

```text
平常時 10%
ピーク時 80%
```

のような負荷を想定すると、

### 従来型

```text
Cost
│
│       ───────────────── Capacity
│
│
│     /\          /\
│ ___/  \________/  \____ Actual demand
│
└────────────────────────── Time
```

### Serverless

```text
Cost
│
│     /\          /\
│ ___/  \________/  \____ Usage ≒ Cost
│
└────────────────────────── Time
```

という違いが視覚的に理解できます。

Lambda Functionsはリクエスト数と実行時間を基本に課金するモデルであるため、この説明との相性がよいです。([Amazon Web Services, Inc.][6])

[AWS Pricing Calculator](https://calculator.aws/?utm_source=chatgpt.com)

---

# 5. 最もおすすめなのは「Serverless Economics Simulator」

さらに一歩進めるなら、Google Trendsのような操作感を**自分たちの投資判断専用に作ってしまう**方法です。

例えばStreamlitで、

```text
Serverless Economics Simulator

Monthly transactions
[──────●────────] 10,000,000

Peak / Average ratio
[────────●──────] 8x

Execution duration
[──●────────────] 300 ms

Engineers
[────●──────────] 6

Infrastructure work
[──────●────────] 120 h/month
```

というスライダーを用意します。

右側にはリアルタイムで、

```text
                    Current      Serverless

Infrastructure       ¥8.2M         ¥3.1M
Operations           ¥5.0M         ¥1.5M
Development          ¥18M          ¥15M
------------------------------------------------
Annual TCO            ¥31.2M        ¥19.6M

Saving                              ¥11.6M
```

を表示します。

さらに重要なのが**損益分岐点**です。

```text
年間コスト

^
|                 Serverless
|                    /
|                   /
| EC2/EKS ----------X---------
|                  /
|                 /
+--------------------------------> Transaction
                 ↑
           Break-even point
```

ビジネスオーナーに、

> 「ではアクセス量を10倍にしてください」

と実際にスライダーを動かしてもらう。

すると、

> 「あ、ずっとServerlessが安いわけではないんですね」

というところまで本人が発見できます。

これが非常に重要です。

**Serverlessを売り込むデモではなく、Serverlessが経済合理性を持つ領域を発見するデモ**になるからです。

---

# 私なら「15分ライブデモ」にする

最も説得力がある順序は、

```mermaid
flowchart LR

A["① Google Trends<br/>関心はあるか"]
--> B["② BuiltWith<br/>実利用はあるか"]

B --> C["③ GitHub<br/>開発活動はあるか"]

C --> D["④ AWS Calculator<br/>単価はいくらか"]

D --> E["⑤ Economics Simulator<br/>自社ならどうなるか"]

E --> F["投資対象<br/>ワークロード選定"]
```

です。

ポイントは、**①〜④までは公開データなので反論しにくく、⑤だけ自社条件を入力する**構造にすることです。

そして最終的には、

> **「Serverlessは安いです」**

ではなく、

> **「このスライダーのこの範囲、つまり負荷変動が大きく、待機時間が長く、運用作業比率の高いワークロードではServerless投資の経済合理性が高くなる」**

というところまで現場で一緒に発見させる。

この形式なら、Google Trendsにかなり近い「**触って納得する投資説明**」になります。

特に今回のテーマなら、**Google Trends + BuiltWith + GitHub + AWS公開価格をデータソースにしたStreamlit製の「Serverless Investment Explorer」**を1本作るのがかなり面白いと思います。市場動向と自社ROIを同じ画面で動かせるので、技術説明から経営判断へ自然につなげられます。

[1]: https://support.google.com/trends/answer/4365533?hl=en&utm_source=chatgpt.com "FAQ about Google Trends data - Trends Help"
[2]: https://support.google.com/trends/answer/17309543?utm_source=chatgpt.com "Compare search terms and topics - Trends Help"
[3]: https://trends.builtwith.com/Web-Server/AWS-Lambda?utm_source=chatgpt.com "AWS Lambda Usage Statistics"
[4]: https://github.com/topics/serverless?utm_source=chatgpt.com "serverless · GitHub Topics · GitHub"
[5]: https://calculator.aws/ "AWS Pricing Calculator"
[6]: https://aws.amazon.com/lambda/pricing/ "AWS Lambda Pricing"
