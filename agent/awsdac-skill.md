`skill-creator` に渡す前提で整理するなら、Skill の中心機能は **「AWS構成を聞き取る → 図の意味構造を決定する → awsdac のレイアウト構造へ変換する」** と定義するのがよいです。

現行の `diagram-as-code` は、`Children` による包含関係だけでなく、Vertical/Horizontal Stack、`BorderChildren`、`SpanResources`、orthogonal link、auto-positioning、link labels、link grouping offset など、かなり高度なレイアウト機能を持っています。特に v0.23 では UnorderedChildren、auto-positioned link labels、layout-aware orthogonal links、SpanResources などが追加されているため、Skill側でこれらを積極的に選択する価値があります。 ([GitHub][1])

## Skill の設計方針

AWS Labs の Best Practices では、大きく **Action Flow / Infrastructure / Management** の3種類を分けています。また、AWSリソースの階層を `Canvas → AWS Cloud → Region → VPC → Subnet → Instance` として捉え、North-South を処理・通信方向、East-West を冗長性/AZ展開に使う設計が推奨されています。 ([GitHub][2])

そのため、Skillには次の考え方を持たせると安定します。

```text
ユーザーのAWSリソース一覧
        ↓
図の目的を決定
        ↓
論理モデル
  - boundary
  - resource
  - relationship
  - flow
        ↓
レイアウト戦略
  - vertical / horizontal
  - grouping
  - redundancy
  - link routing
        ↓
awsdac YAML
```

つまり、**いきなり YAML を生成させない**のが重要です。

---

# skill-creator 投入用たたき台

以下は、そのまま `skill-creator` に渡して初期Skillを作ることを想定した内容です。

# Skill name

awsdac-architecture-layout

# Purpose

AWSクラウドアーキテクチャの構成情報をユーザーとの対話によって整理し、awslabs/diagram-as-code（awsdac）でレンダリング可能な Diagram-as-Code YAML を生成する。

単純にAWSリソースをYAMLへ変換するのではなく、以下を対話的に決定する。

* 図の目的
* AWSリソースのグルーピング
* ネットワーク境界
* リソースの包含関係
* リソース間の通信・イベント・依存関係
* 矢印の方向
* 矢印に記載するメッセージ、プロトコル、イベント名
* 図の詳細度
* North-South / East-West のレイアウト
* 冗長構成の見せ方
* awsdac固有のStack、BorderChildren、SpanResources、Links等の利用方法

最終的には、人が直接メンテナンスでき、Git管理できる awsdac YAML を出力する。

---

# Main use cases

このSkillは以下の場合に利用する。

1. AWSアーキテクチャの構成説明から awsdac YAML を作りたい
2. CloudFormation、CDK、SAMなどから抽出したAWSリソース一覧を図にしたい
3. AWSサービスの一覧は決まっているが、図のレイアウトを決めたい
4. VPC、Subnet、AZ、Regionなどの境界を正しく表現したい
5. SNS、SQS、EventBridge、Lambda、API Gatewayなどの処理フローを矢印で表現したい
6. 既存の awsdac YAML のレイアウトを改善したい
7. AWSアーキテクチャ図を、概要図・設計図・運用図など用途別に作り分けたい

---

# Core design principle

このSkillは

「AWSリソース → YAML」

と直接変換してはいけない。

必ず内部的に次のモデルを構築する。

## Architecture semantic model

### Boundary

物理または論理的な境界。

例:

* AWS Cloud
* Account
* Region
* VPC
* Availability Zone
* Subnet
* Environment
* Security Boundary
* Application Tier

### Resource

AWSサービスまたは外部コンポーネント。

例:

* User
* API Gateway
* Lambda
* SNS
* SQS
* DynamoDB
* NAT Gateway
* EC2

### Relationship

リソース間の関係。

例:

* contains
* calls
* publishes
* subscribes
* reads
* writes
* invokes
* routes
* monitors
* authenticates

### Flow

処理やデータの方向。

例:

User
→ API Gateway
→ Lambda
→ SNS
→ SQS
→ Worker Lambda

このsemantic modelを確定してから、awsdac YAMLへ変換する。

---

# Interactive workflow

ユーザーから構成情報を受け取ったら、不足情報をすべて一度に質問しない。

大きな設計判断から順番に対話する。

## Step 1 — Diagram intent

最初に図の目的を確認する。

候補:

1. Action Flow
2. Infrastructure
3. Management / Operations
4. Hybrid

### Action Flow

ユーザー操作やイベントがシステム内をどう流れるかを中心に表現する。

優先事項:

* 外部トリガー
* request / response
* HTTP / HTTPS
* event
* message
* arrow label

### Infrastructure

ネットワークと配置構造を中心に表現する。

優先事項:

* Region
* VPC
* Availability Zone
* Subnet
* Routing
* Security Boundary

### Management

管理・監視・運用経路を中心に表現する。

優先事項:

* Administrator
* Console
* CLI
* Systems Manager
* CloudWatch
* Logging
* Monitoring

Hybridの場合は、どの観点を主とするか確認する。

---

# Step 2 — Detail level

ユーザーに図の詳細レベルを選んでもらう。

## Level 1 — Concept

目的:
経営層、非エンジニア、概要説明。

表示するもの:

* 主要AWSサービス
* 主要な処理フロー
* AWS Cloud程度の大きな境界

省略するもの:

* Subnet
* Security Group
* Route Table
* NAT Gateway
* 詳細な通信プロトコル

目安:
5〜10リソース。

## Level 2 — Architecture

目的:
開発チーム、設計レビュー。

表示するもの:

* Region
* VPC
* Public / Private Subnet
* AZ
* AWSサービス
* 主な通信経路
* イベント名

目安:
10〜30リソース。

デフォルトはこのレベルとする。

## Level 3 — Implementation

目的:
インフラ構築、セキュリティレビュー、運用設計。

表示候補:

* Region
* VPC
* AZ
* Subnet
* Gateway
* Load Balancer
* Compute
* Database
* Messaging
* Observability
* Security Boundary
* 通信プロトコル
* イベント
* Route / ingress / egress

ただし、すべてを無条件に表示しない。

情報量が多い場合は、複数の図へ分割することを提案する。

---

# Step 3 — Resource boundaries

以下の順番で包含関係を整理する。

Canvas
→ AWS Cloud
→ Region
→ VPC
→ AZ / Subnet
→ Resource

ただしすべてのAWSサービスをVPC内に配置してはいけない。

AWSサービスの実際のネットワーク配置と
「図として近くに配置したい」という要求を区別する。

ユーザーの意図が曖昧な場合は、

「物理/ネットワーク境界として表現したいのか」
「論理的なグループとしてまとめたいのか」

を区別する。

---

# Step 4 — Grouping strategy

グルーピング候補を確認する。

* Network boundary
* Availability Zone
* Application tier
* Environment
* Business function
* Security boundary

デフォルトの優先順位:

1. AWS Cloud / Region
2. Network boundary
3. Availability Zone
4. Application tier
5. Logical function

awsdacのChildrenは原則として
「図の包含関係」を表現するために利用する。

---

# Step 5 — Layout orientation

AWS architecture best practicesを参考に、デフォルトではNorth-Southレイアウトを使う。

North:

* User
* Internet
* External system
* Entry point

Middle:

* API
* Load Balancer
* Compute
* Application

South:

* Database
* Internal processing
* Backend

East-West方向は主に以下を表現する。

* Availability Zone
* redundancy
* active-active resources
* parallel processing

例:

AZ-a       AZ-b
|          |
Subnet     Subnet
|          |
Lambda     Lambda

同一レイヤーで冗長構成を示す場合は HorizontalStack を優先する。

処理階層を示す場合は VerticalStack を優先する。

---

# Step 6 — awsdac layout resource selection

次の判断ルールを利用する。

## AWS::Diagram::VerticalStack

利用するケース:

* processing pipeline
* web → app → data
* north-south traffic
* sequential architecture

## AWS::Diagram::HorizontalStack

利用するケース:

* Availability Zone
* redundant resources
* parallel workers
* equivalent components

## BorderChildren

境界上に存在することを視覚的に示したいリソースに使う。

例:

* Internet Gateway
* VPN Gateway
* Transit Gateway Attachment

可能であればVPC境界上に配置する。

## SpanResources

単純な親子構造では表せない論理的グループに使用する。

例:

* Auto Scaling Group spanning multiple subnets
* 複数Subnetをまたぐ論理構成

Children と SpanResources は同一リソースで同時利用しない。

---

# Step 7 — Relationship interview

リソース間の関係を聞き取る。

単に

Lambda → SNS

とせず、

「LambdaからSNSには何が送信されますか？」

を確認する。

候補:

* HTTP request
* HTTPS request
* SNS message
* SQS message
* EventBridge event
* API call
* DB query
* object
* log
* metric
* notification

必要に応じて具体的なイベント名も確認する。

例:

Lambda
→ SNS

label:

OrderCreated

または

Publish: OrderCreated

---

# Step 8 — Link semantics

awsdac Links を使って関係を表現する。

デフォルト:

SourcePosition: auto
TargetPosition: auto

可能な限りauto-positioningを利用する。

矢印は原則として処理またはデータの流れる方向を示す。

例:

Lambda
→ SNS

の場合:

Source: Lambda
Target: SNSTopic

TargetArrowHead:
Type: Open

---

# Link type

デフォルト:

Type: orthogonal

理由:

複雑なAWSアーキテクチャで線の交差や斜線を減らし、
図を読みやすくするため。

単純な2リソース間や、直線の方が意味を理解しやすい場合のみstraight linkを利用する。

---

# Link labels

通信やイベントの意味が重要な場合はLabelsを追加する。

推奨:

Labels:
AutoRight:
Title: "OrderCreated"

または

Labels:
AutoLeft:
Title: "HTTPS"

ラベルには以下を優先する。

Action Flow:

* protocol
* API operation
* message
* event

Infrastructure:

* route
* port/protocol
* network relationship

Management:

* logs
* metrics
* alerts
* administrative access

ラベルが過密になる場合は省略する。

---

# Link overlap

同一リソースから多数のLinksが出入りする場合は

Options.GroupingOffset: true

の利用を検討する。

UnorderedChildrenが利用できる場合は、
リンク交差を減らせるレイアウトを優先する。

---

# Step 9 — Naming convention

論理IDには説明的な名前を使う。

Good:

PublicSubnetAzA
OrderProcessingLambda
OrderEventsTopic
ApplicationLoadBalancer

Avoid:

Subnet1
Lambda1
SNS1
ResourceA

環境を区別する必要がある場合:

ProdVPC
DevVPC

などを使用する。

Titleは人間が読む名称、
YAML keyは安定したlogical identifierとして扱う。

---

# Step 10 — Diagram complexity control

以下の場合は図の分割を検討する。

* 30以上のリソース
* 3階層以上のネットワーク境界が多数存在する
* Linksが多く交差する
* Action FlowとInfrastructureの両方を詳細に表現しようとしている
* Management Planeまで同じ図へ追加しようとしている

分割例:

01-overview.yaml
02-network.yaml
03-request-flow.yaml
04-event-flow.yaml
05-operations.yaml

---

# Conversational behavior

ユーザーとの対話では、
awsdacのプロパティ名を最初から質問しない。

Bad:

「Directionをverticalにしますか？」

Good:

「処理の流れは上から下に見せたいですか、それとも左右に見せたいですか？」

ユーザーの回答を内部的に

Direction: vertical

へ変換する。

Bad:

「HorizontalStackを使いますか？」

Good:

「2つのAvailability Zoneを横並びで、同等の構成として見せますか？」

---

# Recommended questions

必要なものだけ段階的に聞く。

## Diagram purpose

「この図で一番伝えたいものはどれですか？」

* 処理の流れ
* AWSリソース配置
* ネットワーク
* 運用・監視
* 全体概要

## Granularity

「どこまで細かく表示しますか？」

* 概要
* 設計レビュー
* 実装詳細

## Boundaries

「次の境界のうち、図に明示したいものはありますか？」

* AWS Account
* Region
* VPC
* Availability Zone
* Subnet
* Environment

## Relationships

「リソース間の矢印には何を表現しますか？」

* 呼び出し
* データ
* メッセージ
* イベント
* ネットワーク通信
* 運用・監視

## Arrow labels

「矢印に処理内容を表示しますか？」

例:

HTTPS
OrderCreated
SNS message
PutObject
Invoke

---

# Default decisions

ユーザーが特に指定しない場合:

Diagram type:
Hybrid / architecture-centric

Detail:
Level 2

Flow:
North → South

Redundancy:
East ↔ West

Link:
orthogonal

Link position:
auto

Arrow:
TargetArrowHead Open

Definition:
AWS icons light definition

Naming:
descriptive logical IDs

Grouping:
AWS Cloud → Region → VPC → Subnet

ただし、実際のAWSリソース境界と矛盾する包含関係を作らない。

---

# Output

最終回答では次の順番で提示する。

1. Diagram design summary
2. Resource hierarchy
3. Relationship summary
4. awsdac YAML

例:

Diagram:
DefinitionFiles:
- Type: URL
Url: [https://raw.githubusercontent.com/awslabs/diagram-as-code/main/definitions/definition-for-aws-icons-light.yaml](https://raw.githubusercontent.com/awslabs/diagram-as-code/main/definitions/definition-for-aws-icons-light.yaml)

Resources:

```
Canvas:
  Type: AWS::Diagram::Canvas
  Direction: vertical
  Children:
    - AWSCloud
    - User

AWSCloud:
  Type: AWS::Diagram::Cloud
  Preset: AWSCloudNoLogo
  Direction: vertical
  Children:
    - Region

Region:
  Type: AWS::Region
  Title: ap-northeast-1
  Direction: vertical
  Children:
    - VPC
    - OrderEventsTopic

VPC:
  Type: AWS::EC2::VPC
  Direction: vertical
  Children:
    - PrivateSubnet

PrivateSubnet:
  Type: AWS::EC2::Subnet
  Preset: PrivateSubnet
  Children:
    - ApplicationLambda

ApplicationLambda:
  Type: AWS::Lambda::Function
  Title: Application

OrderEventsTopic:
  Type: AWS::SNS::Topic
  Title: Order Events

User:
  Type: AWS::Diagram::Resource
  Preset: User
```

Links:

```
- Source: ApplicationLambda
  SourcePosition: auto
  Target: OrderEventsTopic
  TargetPosition: auto
  Type: orthogonal
  TargetArrowHead:
    Type: Open
  Labels:
    AutoRight:
      Title: "Publish: OrderCreated"
```

---

# Validation

YAML出力前に内部的に以下を確認する。

## Structural validation

* Canvasが1つだけ存在する
* Canvasから表示対象リソースへ到達できる
* Childrenに存在しないlogical IDを指定していない
* LinksのSource/TargetがResourcesに存在する
* circular parent-child relationshipがない

## Architecture validation

* VPC外サービスを誤ってSubnet配下へ配置していない
* AZ冗長構成が横方向で理解できる
* 外部→内部の流れがNorth-Southで理解できる
* 同一リソースを不必要に複数箇所へ配置していない

## Readability validation

* link crossingが多すぎない
* labelsが過密でない
* resource namesが意味を持っている
* redundant resourcesが一貫したレイアウトになっている

---

# Do not do

* AWSリソースをすべて機械的に図へ追加しない
* すべてのリソース間依存関係を矢印にしない
* VPCに属さないAWSサービスをVPC内へ入れない
* YAML上の依存関係と図として重要な関係を混同しない
* 見た目だけの理由でAWSの境界を誤って表現しない
* 存在しないAWS resource typeやPresetを推測して生成しない
* 不明なPresetはDefinitionFilesまたはawslabs/diagram-as-code documentationで確認する

---

# Design objective

最適化対象は

「AWSリソース数」

ではなく

「閲覧者が理解すべきアーキテクチャ上の関係」

とする。

良い図とは、
YAMLが詳細であることではなく、

* boundary
* responsibility
* flow
* dependency
* redundancy

が短時間で理解できる図である。

このたたき台で特に重要なのは、**「質問 → awsdacプロパティ」の翻訳層**をSkillに持たせている点です。

たとえば、

| ユーザーとの会話                     | Skill内部の判断                   | awsdac                               |
| ---------------------------- | ---------------------------- | ------------------------------------ |
| 「処理を上から下へ」                   | North-South flow             | `Direction: vertical`                |
| 「AZ-aとAZ-bを横並び」              | redundancy                   | `HorizontalStack`                    |
| 「IGWはVPCの入口」                 | boundary resource            | `BorderChildren`                     |
| 「ASGが2 Subnetをまたぐ」           | cross-boundary logical group | `SpanResources`                      |
| 「LambdaがSNSへOrderCreatedを発行」 | event flow                   | `Links + Labels`                     |
| 「線がごちゃつく」                    | routing optimization         | `orthogonal + auto + GroupingOffset` |

この変換を挟むことで、ユーザーが `HorizontalStack` や `SourcePosition` を知らなくても使えるSkillになります。`awsdac`自身も、自動リンク位置、直交リンク、リンクラベル、境界配置などをサポートしており、この設計との相性がよいです。 ([GitHub][3])

## さらに Skill を強くするなら

Skill本体の `SKILL.md` に全仕様を詰め込むより、次のようにリファレンスを分離する構成をおすすめします。

```text
awsdac-architecture-layout/
├── SKILL.md
├── references/
│   ├── layout-rules.md
│   ├── awsdac-resource-types.md
│   ├── link-rules.md
│   ├── aws-boundary-rules.md
│   └── interview-model.md
└── examples/
    ├── api-lambda-dynamodb.yaml
    ├── event-driven-sns-sqs.yaml
    ├── multi-az-vpc.yaml
    └── operations-monitoring.yaml
```

特に `examples/` は重要です。`diagram-as-code` 公式のBest Practicesでも、ネットワーク境界、North-South方向、East-West冗長性、機能/AZ/環境/セキュリティ境界によるグルーピング、説明的な命名が推奨されています。Skillにこれらの**良質なfew-shot例**を持たせると、単なるプロンプト指示よりレイアウト品質が安定します。 ([GitHub][2])

また、`awsdac` 自体に MCP Server があり、`getDiagramAsCodeFormat` でフォーマット仕様や例、ベストプラクティスを取得できる設計になっています。将来的にはこのSkillを **静的な知識だけでYAML生成するSkillではなく、「Skill → awsdac MCP → 最新フォーマット確認 → YAML生成 → awsdacでレンダリング → レイアウト修正」**という反復型エージェントに発展させると、かなり強力です。 ([GitHub][4])

なお、この会話環境では `skill-creator` 自体が利用可能なSkillとして公開されていなかったため、今回は**skill-creatorへ投入できる仕様書の形**まで具体化しています。次に進めるなら、この内容をさらに圧縮して、**実際の `SKILL.md` 相当の完成版**と、`event-driven-sns-sqs.yaml` など3〜4個のfew-shot教材まで作るのが最も効果的です。

[1]: https://github.com/awslabs/diagram-as-code/blob/main/doc/resource-types.md "diagram-as-code/doc/resource-types.md at main · awslabs/diagram-as-code · GitHub"
[2]: https://github.com/awslabs/diagram-as-code/blob/main/doc/best-practices.md "diagram-as-code/doc/best-practices.md at main · awslabs/diagram-as-code · GitHub"

---

# External Icon

はい。追加するなら、単に「GitHub/Slack/Teamsのアイコンも使う」と書くより、**AWS外リソースを第一級のアーキテクチャ要素として扱うルール**をSkillに追加した方がよいです。

`diagram-as-code` 自体は Definition File を追加して非AWS図へ拡張でき、`DefinitionFiles` にはURLまたはローカルファイルを指定できます。一方、Azure/GCPなど他プロバイダーの公式アイコン内蔵については、現時点でも機能要望が残っています。したがって GitHub / Slack / Teams 等は、**カスタムDefinition Fileを使う前提で設計する**のが堅実です。([GitHub][1])

前回の `skill-creator` 向け指示には、以下を追加するのがおすすめです。

# Multi-vendor / External Service Support

このSkillはAWSリソースだけでなく、AWS環境と連携する外部サービス、SaaS、Developer Tools、Enterprise Systemsをアーキテクチャ図の第一級リソースとして扱う。

対象例:

* GitHub
* GitHub Actions
* GitLab
* Slack
* Microsoft Teams
* Jira
* Confluence
* Datadog
* PagerDuty
* ServiceNow
* Terraform
* Jenkins
* Docker
* Kubernetes
* Salesforce
* SaaS API
* On-premises system
* External user / organization

AWS外リソースであるという理由だけでGeneric Resourceへ変換せず、対応するカスタムアイコン定義が利用可能な場合は、そのアイコンを優先して使用する。

---

# Icon source abstraction

リソースのアイコンは以下の3種類に分類して扱う。

## 1. AWS native icon

awsdac公式AWS Definition Fileで定義されているAWSリソース。

例:

AWS::Lambda::Function
AWS::SNS::Topic
AWS::SQS::Queue

これらについては公式AWS Definition Fileを優先する。

## 2. External / custom icon

GitHub、Slack、Microsoft Teamsなど、
awsdac標準AWS Definition Fileに存在しないサービス。

これらはCustom Definition Fileによって解決する。

例:

Custom::GitHub
Custom::GitHubActions
Custom::Slack
Custom::MicrosoftTeams
Custom::ServiceNow

具体的なType名は実際に読み込まれているDefinition Fileに従う。

存在しないType名を推測して生成してはいけない。

## 3. Generic external resource

適切なアイコン定義が存在しない場合のFallback。

Generic resourceを使用し、

Title:
"External CI System"

など、人間が理解できる名称を付ける。

アイコンが見つからないことを理由に、
アーキテクチャ上重要なコンポーネント自体を省略してはいけない。

---

# Definition File strategy

Diagramには必要に応じて複数のDefinition Fileを読み込む。

概念例:

Diagram:
DefinitionFiles:

```
# AWS official icons
- Type: URL
  Url: <AWS Definition File>

# External services
- Type: LocalFile
  LocalFile: ./definitions/external-services.yaml
```

AWSリソースと外部サービスを同一Diagram内で利用できる構成を優先する。

Custom Definition Fileの場所は以下の優先順位で決定する。

1. ユーザーが明示したDefinition File
2. プロジェクト標準Definition File
3. Skillに登録されたDefinition File
4. Generic Resource fallback

Skillは存在しないDefinition File URLやTypeを推測して生成してはいけない。

---

# External icon registry

Skill内部では、外部サービスについて次のような概念的なRegistryを持つ。

Example:

GitHub:
category: developer-platform
scope: external
preferred_type: Custom::GitHub

GitHubActions:
category: ci-cd
scope: external
preferred_type: Custom::GitHubActions

Slack:
category: collaboration
scope: external
preferred_type: Custom::Slack

MicrosoftTeams:
category: collaboration
scope: external
preferred_type: Custom::MicrosoftTeams

ServiceNow:
category: itsm
scope: external
preferred_type: Custom::ServiceNow

Datadog:
category: observability
scope: external
preferred_type: Custom::Datadog

preferred_typeはCustom Definition Fileで実在するTypeが確認できた場合のみ使用する。

---

# External service boundary rules

SaaSをAWS Cloud、Region、VPC、SubnetのChildrenへ配置してはいけない。

例えば:

GitHub
Slack
Microsoft Teams
ServiceNow
Datadog SaaS

などは原則としてAWS Cloudの外側に配置する。

Conceptual layout:

```
                GitHub
                   |
                   |
```

User → AWS Cloud → Application → Slack
|
|
ServiceNow

AWS Cloudと外部サービスをCanvas直下のSiblingとして配置することを基本とする。

例:

Canvas
├── GitHub
├── AWSCloud
├── Slack
└── ServiceNow

ただし、以下のようなSelf-hosted製品については配置場所を確認する。

例:

* GitHub Enterprise Server
* Self-hosted GitLab
* Jenkins
* Self-hosted Grafana

これらはEC2、EKS、オンプレミスなどに配置されている可能性があるため、

「これはSaaSですか、それともAWS/オンプレミス上に配置されたシステムですか？」

と確認する。

---

# External grouping

AWS境界とは別に、
外部サービスを論理グループとしてまとめることを許可する。

例:

Developer Platform
├── GitHub
└── GitHub Actions

Collaboration
├── Slack
└── Microsoft Teams

IT Operations
├── ServiceNow
├── PagerDuty
└── Datadog

ただし論理グループと物理的なAWS Boundaryを混同してはいけない。

---

# Interactive questions for external services

ユーザーがAWS外サービスを指定した場合、
必要に応じて以下を確認する。

## Service type

「このサービスはどのような位置付けですか？」

例:

* SaaS
* 外部API
* オンプレミスシステム
* AWS上のSelf-hostedサービス

## Architecture role

「このサービスはAWSシステムとどのように関係しますか？」

例:

* Source code repository
* CI/CD
* Notification
* ChatOps
* Monitoring
* Incident Management
* Ticket Management
* Authentication
* External API

## Relationship

「AWS側とは何をやり取りしますか？」

例:

GitHub
→ source code
→ CodePipeline

GitHub Actions
→ deploy
→ AWS

CloudWatch
→ alert
→ Slack

SNS
→ notification
→ Microsoft Teams

Application
→ incident
→ ServiceNow

---

# External relationship semantics

外部サービスとのLinksについても、
単純な線ではなく意味を表現する。

Example relationships:

GitHub
→ CodePipeline

Label:
Source code

GitHubActions
→ AWS

Label:
Deploy

CloudWatch
→ Slack

Label:
Alert

SNS
→ MicrosoftTeams

Label:
Notification

ServiceNow
← AWS

Label:
Create Incident

Datadog
← Application

Label:
Metrics / Traces

---

# Relationship categories

外部サービスとの関係を以下に分類する。

developer-flow:

* source code
* pull request
* build
* deploy
* artifact

event-flow:

* event
* message
* webhook

operations-flow:

* metric
* log
* trace
* alert
* incident

collaboration-flow:

* notification
* ChatOps
* approval

business-flow:

* ticket
* business event
* workflow request

security-flow:

* authentication
* authorization
* secret
* identity federation

このカテゴリはレイアウトやLabel選択の判断材料として使用する。

---

# Layout rules for external services

外部サービスは処理フローの方向に応じて配置する。

## Upstream systems

入力元:

* User
* GitHub
* External API
* SaaS
* Partner system

原則としてAWS CloudよりNorth側に置く。

Example:

GitHub
|
v
AWS CI/CD

## Downstream systems

通知・運用:

* Slack
* Microsoft Teams
* PagerDuty
* ServiceNow

原則としてAWS CloudのSouthまたはEast側に配置する。

Example:

AWS
|
+------> Slack
|
+------> ServiceNow

## Side systems

ObservabilityやManagement:

* Datadog
* Splunk
* New Relic
* Security SaaS

主処理フローを妨げないEast/West側へ置く。

---

# Primary architecture flow

外部サービスを追加しても、
主処理経路と補助経路を区別する。

Example:

Primary:

User
→ API Gateway
→ Lambda
→ DynamoDB

Supporting:

GitHub Actions
→ Lambda

CloudWatch
→ Slack

Lambda
→ ServiceNow

主処理経路を図の中央へ置き、
CI/CD、Observability、Notification等は周辺へ配置する。

---

# Icon resolution workflow

外部リソースをDiagramへ追加するときは、
以下の順序で解決する。

Step 1:

awsdac標準Definition Fileを確認。

Step 2:

読み込まれているCustom Definition Fileを確認。

Step 3:

該当するExternal Service Typeを検索。

Step 4:

見つかった場合はそのTypeを使用。

Step 5:

見つからない場合はGeneric Resourceを使用する。

Step 6:

必要であれば、

「このサービス用のCustom Definitionを追加しますか？」

とユーザーに提案する。

存在確認を行わずに、

Custom::GitHub

Custom::Slack

Custom::Teams

などのTypeを生成してはいけない。

---

# Trademark / icon handling

外部サービスのロゴについて、
Skill自身が任意のWebサイトからロゴ画像を自動取得することを前提としない。

以下のいずれかを使用する。

* ユーザー指定アイコン
* プロジェクトで管理されているアイコン
* 利用条件が確認された公式アセット
* 既存Custom Definition File
* Generic Resource fallback

Skillの主目的はアイコン収集ではなく、
Architecture Semantic ModelからDiagram Layoutを生成することである。

---

# Updated architecture semantic model

Resourceに次の属性を追加する。

Resource:

id:
title:

provider:
aws
github
microsoft
slack
external
on-premises

category:
compute
storage
messaging
ci-cd
collaboration
observability
itsm
external-api

hosting:
aws
saas
on-premises
external-cloud
unknown

icon_source:
aws-definition
custom-definition
generic

boundary:
AWSCloud
Region
VPC
Subnet
External
OnPremises

これにより、

「AWSリソースかどうか」

と

「どこに配置されているか」

を別々に判断する。

---

# Example semantic model

GitHubActions:

provider: github
category: ci-cd
hosting: saas
icon_source: custom-definition
boundary: External

ApplicationLambda:

provider: aws
category: compute
hosting: aws
icon_source: aws-definition
boundary: AWSCloud

Slack:

provider: slack
category: collaboration
hosting: saas
icon_source: custom-definition
boundary: External

---

# Example architecture

GitHub
|
| Source
v

+---------------- AWS Cloud ----------------+
|                                           |
| CodePipeline                              |
|      |                                    |
|      v                                    |
|   Lambda -----> SNS -------------------+  |
|                                       |   |
+---------------------------------------|---+
|
+------------------+----------------+
|                                   |
v                                   v

```
               Slack                             ServiceNow
```

このように、

AWS Boundary

と

External SaaS

の境界が視覚的に区別できる構成を優先する。

---

# Validation extension

既存Validationに次を追加する。

## External resource validation

* SaaSが誤ってVPC/Subnet配下に配置されていない
* External ResourceとAWS Resourceの境界が明確である
* Custom TypeがDefinition Fileに存在する
* 存在しない外部アイコンTypeを生成していない
* SaaSとSelf-hosted製品を混同していない
* 外部サービスとのLinkに意味がある
* CI/CDやMonitoringの線が主処理フローを妨げていない

---

# Design objective extension

このSkillはAWS Architecture Diagram Generatorではなく、

Cloud-centric System Architecture Diagram Generator

として振る舞う。

AWSは中心的なInfrastructure Boundaryとして扱うが、
GitHub、Slack、Microsoft Teams、ServiceNow、Datadog等も
システムアーキテクチャを構成する重要なResourceとして表現する。

最終的な最適化対象は

「AWSリソースを正確に並べること」

ではなく、

「クラウドと、その周辺サービスを含めたシステム全体の関係性を理解できること」

とする。

### 特に追加したい設計ポイント

私はさらに、Skillの内部モデルを **`AWS Resource / Non-AWS Resource` の2択にしない**ことをおすすめします。

むしろ、

```text
Resource
 ├─ Provider
 ├─ Hosting
 ├─ Architecture Role
 ├─ Boundary
 └─ Icon Source
```

と分けます。

たとえば GitHub Actions なら、

```yaml
provider: github
hosting: saas
role: ci-cd
boundary: external
icon_source: custom-definition
```

Jenkins on EC2なら、

```yaml
provider: jenkins
hosting: aws
role: ci-cd
boundary: vpc
icon_source: custom-definition
```

です。

これなら、**同じ「GitHub/Jenkinsのような外部製品」でも、どこに置くべきかを正しく判断できます。**

また、awsdacは「追加Definition Fileによってnon-AWS diagramへ拡張可能」と明示していますし、Definition Fileはローカルファイルとして読み込めます。したがって、最終的にはSkillと一緒に例えば次のような資産を管理する構成がきれいです。([GitHub][1])

```text
awsdac-architecture-layout/
├── SKILL.md
├── references/
│   ├── layout-rules.md
│   ├── external-resource-rules.md
│   └── icon-registry.md
│
├── definitions/
│   └── external-services.yaml
│
└── assets/
    ├── github.svg
    ├── github-actions.svg
    ├── slack.svg
    ├── microsoft-teams.svg
    ├── servicenow.svg
    └── datadog.svg
```

この構造にすると、**Skill = レイアウト判断ロジック、Definition = awsdacへのアイコン登録、Assets = 実際の画像資産**という責務分離になります。

特にこの「外部アイコンRegistry」をSkillの独立コンポーネントにする設計は、GitHub/Slack/Teamsだけでなく、将来的に **Datadog、ServiceNow、SonarQube、monday.com、Backstage、Terraform Cloud** などを追加するときにも横展開しやすいです。

[1]: https://github.com/awslabs/diagram-as-code?utm_source=chatgpt.com "GitHub - awslabs/diagram-as-code: Diagram-as-code for AWS architecture. · GitHub"

[3]: https://github.com/awslabs/diagram-as-code/blob/main/doc/links.md "diagram-as-code/doc/links.md at main · awslabs/diagram-as-code · GitHub"
[4]: https://github.com/awslabs/diagram-as-code/blob/main/doc/mcp-server.md?utm_source=chatgpt.com "diagram-as-code/doc/mcp-server.md at main · awslabs/diagram-as-code · GitHub"
