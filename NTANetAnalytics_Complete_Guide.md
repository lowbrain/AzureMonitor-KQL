# Azure Log Analytics: NTANetAnalytics テーブル 完全解説（最終版）

---

このドキュメントは、Azure Log Analyticsの`NTANetAnalytics`テーブルについて、基本的な使い方から、これまで交わされた詳細な質疑応答の内容まで、すべての情報を網羅した総合ガイドです。会話の中での具体例や比喩表現をすべて含めることで、より実践的で理解しやすい内容を目指しました。

## 1. NTANetAnalytics テーブルとは？

`NTANetAnalytics`テーブルは、Azureの**Traffic Analytics**ソリューションによって生成される、非常にリッチなネットワークフローログの格納先です。VNetフローログの生データに、地理情報、セキュリティ情報、サービス情報などを付与し、集計・加工したデータが格納されており、高度なネットワークトラフィック分析を可能にします。

## 2. 分析に特に重要なカラム

通信量を分析する上で、特に重要となるカラムの一覧です。

| カラム名 | 型 | 説明 |
| :--- | :--- | :--- |
| `TimeGenerated` | datetime | このログレコードが生成された時刻 (UTC) |
| `SrcIp`, `DestIp` | string | 送信元と宛先のIPアドレス |
| `SrcSubnet`, `DestSubnet` | string | 送信元と宛先のサブネット名 |
| `SrcVnet`, `DestVnet` | string | 送信元と宛先の仮想ネットワークのGUID |
| `BytesSrcToDest` | long | 送信元から宛先へ送信されたバイト数 |
| `BytesDestToSrc` | long | 宛先から送信元へ送信されたバイト数 |
| `FlowType` | string | フローの種類（VNet内、インターネットなど）。詳細は後述。 |
| `FlowStatus` | string | フローの状態 (`Allowed` または `Denied`) |
| `FlowDirection` | string | NICから見た通信の方向 (`Inbound` または `Outbound`)。詳細は後述。 |

## 3. KQLクエリのサンプル

### 3.1. VNet間の総通信量トップ10

```kusto
NTANetAnalytics
| where SubType == "FlowLog" and FlowType == "InterVNet"
| where TimeGenerated > ago(7d)
| summarize TotalBytes = sum(BytesSrcToDest + BytesDestToSrc) by SrcVnet, DestVnet
| top 10 by TotalBytes desc
| render barchart
```

### 3.2. 特定のVNetからインターネットへの通信量が多い宛先トップ10

```kusto
NTANetAnalytics
| where SubType == "FlowLog" and FlowType == "ExternalPublic"
| where TimeGenerated > ago(1d)
| where SrcVnet == "YOUR_VNET_GUID" // 分析したいVNetのGUIDに置き換えてください
| summarize TotalBytes = sum(BytesSrcToDest) by DestIp
| top 10 by TotalBytes desc
```

### 3.3. サブネットごとの総通信量（送受信の内訳）

```kusto
NTANetAnalytics
| where SubType == "FlowLog" and FlowType == "IntraVNet"
| where TimeGenerated > ago(1d)
| extend Subnet = iif(isnotempty(SrcSubnet), SrcSubnet, DestSubnet)
| where isnotempty(Subnet)
| summarize SentBytes = sum(BytesSrcToDest), ReceivedBytes = sum(BytesDestToSrc) by tostring(Subnet)
| extend TotalBytes = SentBytes + ReceivedBytes
| order by TotalBytes desc
```

## 4. 詳細な解説（よくある質問）

これまでの会話で挙がった、より詳細なトピックについて、具体的な例を交えて解説します。

### Q1: `FlowType`にはどのような種類がありますか？

`FlowType`は通信経路を分類する重要な列です。

| FlowType 値 | 通信の概要 | 解説 |
| :--- | :--- | :--- |
| **`IntraVNet`** | VNet内の通信 | 同じVNet内のリソース間の通信。 |
| **`InterVNet`** | VNet間の通信 | 異なるVNet間の通信（VNetピアリングなど）。 |
| **`AzurePublic`** | AzureパブリックIPとの通信 | VMからAzureのPaaSサービス（Storageなど）への通信。 |
| **`ExternalPublic`** | 外部パブリックIPとの通信 | VMからインターネット上のサーバーへの通信など。 |
| **`S2S`** | サイト間VPN/ExpressRoute | オンプレミスとAzure VNet間の通信。 |
| **`P2S`** | ポイント対サイトVPN | クライアントPCとAzure VNet間の通信。 |
| **`MaliciousFlow`** | 悪意のあるIPとの通信 | Microsoftの脅威インテリジェンスで悪意があると識別されたIPとの通信。 |
| **`UnknownPrivate`** | 不明なプライベートIP | RFC 1918のプライベートIPだが、どのVNetにも属さないIPとの通信。 |
| **`Unknown`** | 不明な通信 | 上記のいずれにも分類できない通信。 |

### Q2: フローログは、どのような単位で保存されますか？

**時間単位**と**データ単位**の2つの側面があります。

1.  **時間単位**: 生ログは1分間隔で生成され、Traffic Analyticsが約10分間隔で処理・集計します。そのため、データが`NTANetAnalytics`に反映されるまで最大1時間程度の遅延があります。
2.  **データ単位**: 1つのレコードは、**双方向の会話（セッション）**を示します。処理インターバル内に発生した、同じ5タプル（送信元IP, 宛先IP, 送信元ポート, 宛先ポート, プロトコル）を持つ複数の通信が集約されたものです。

### Q3: フローログはOSIのどの階層のログですか？

主に**レイヤー3（ネットワーク層）**と**レイヤー4（トランスポート層）**のログですが、Traffic Analyticsによって**レイヤー7（アプリケーション層）**の情報も付与されます。

| OSI参照モデル | 関連するフローログの列 |
| :--- | :--- |
| **レイヤー7** | `L7Protocol` (例: HTTP, DNS) ※Traffic Analyticsによる付加情報 |
| **レイヤー4** | `L4Protocol` (TCP/UDP), `SrcPort`, `DestPort` |
| **レイヤー3** | `SrcIp`, `DestIp` |

### Q4: `FlowDirection`について詳しく教えてください。

ログを記録した**ネットワークインターフェース（NIC）から見た**通信の方向です。

- **`Inbound`**: NICに**入ってくる**通信。
- **`Outbound`**: NICから**出ていく**通信。

### Q5: 1つの通信で2つのログレコードが生成されるのは本当ですか？

**はい、本当です。** これは、フローログが個々のNICの視点で記録されるためです。

> **具体例：監視カメラの比喩**
> 建物の出入り口に、外向きのカメラAと内向きのカメラBがあるとします。一人の人がドアを通過する**1つのイベント**に対し、カメラAは「出ていった」記録を、カメラBは「入ってきた」記録を残します。合計2つの記録が残るのと同じです。
> Azureでは、VM-AからVM-Bへの1つの通信に対し、VM-AのNICは`Outbound`のログを、VM-BのNICは`Inbound`のログを生成するため、合計2つのレコードが生成される可能性があります。

### Q6: なぜ `Inbound` フローでも `BytesDestToSrc` が記録されるのですか？

`NTANetAnalytics`の1レコードが示す「フロー」は、一方通行の通信ではなく、**双方向の会話（セッション）**全体を指すためです。

> **具体例：Webサイトへのアクセス**
> 1. **あなた → サーバー**: 「このページをください」というリクエストを送信します。これが `BytesSrcToDest` です。
> 2. **サーバー → あなた**: 「了解」という返事と、Webページのデータを返信します。これが `BytesDestToSrc` です。
>
> `FlowDirection: Inbound`は、この会話が「外部から開始された」ことを示すだけで、会話の中身であるリクエストとレスポンスの両方のバイト数が記録されます。

### Q7: 長いセッションはログが分割されますか？

**はい、分割されます。** セッションの継続時間がTraffic Analyticsの処理インターバル（約10分）をまたぐ場合、セッションはインターバルごとに分割され、複数のログレコードとして記録されます。

> **具体例：タイムライン上の長いセッション**
> 1時間にわたるファイル転送セッションがあったとします。Traffic Analyticsは10分ごとにログを生成するため、この1つのセッションは、約6個のログレコードに分割されます。各レコードには、その10分間に転送された分のバイト数が記録されます。これらのレコードは5タプルがすべて同じになるため、後から集計することが可能です。

### Q8: セッションの途中で `FlowDirection` が変わることはありますか？

**いいえ、ありません。** `FlowDirection`はセッションが**最初に確立された瞬間**に一度だけ決まり、そのセッションが終了するまで不変です。

> **具体例：電話の比喩**
> あなたが友人に電話をかけた場合、その通話はあなたにとって`Outbound`です。途中で友人が一方的に話し続けたとしても、その通話が`Inbound`に変わることはありません。`FlowDirection`は、セッションの「発信者」が誰かを示す静的な属性であり、途中のデータ量やパケットの向きによって変化しません。

## 5. 全カラム定義一覧

`NTANetAnalytics`テーブルの全カラムの定義です。公式ドキュメントに基づいています。

### 基本情報 (Basic Information)
| カラム名 | 型 | 説明 |
| :--- | :--- | :--- |
| `TimeGenerated` | datetime | このログデータがLog Analyticsワークスペースに取り込まれた時刻 (UTC)。 |
| `TimeProcessed` | datetime | トラフィック分析機能がストレージアカウントから生のフローログを処理した時刻 (UTC)。 |
| `SourceSystem` | string | イベントを収集したエージェントの種類 (例: `Azure`)。 |
| `TenantId` | string | Log AnalyticsワークスペースのテナントID。 |
| `Type` | string | テーブル名 (`NTANetAnalytics`)。 |
| `_IsBillable` | string | データ取り込みが課金対象かどうかを示します。 |
| `_BilledSize` | real | レコードのサイズ (バイト単位)。 |
| `FASchemaVersion` | string | 取り込みスキーマのバージョン。 |
| `SubType` | string | ログのサブタイプ。通常は `FlowLog` を使用します。 |

### フロー情報 (Flow Information)
| カラム名 | 型 | 説明 |
| :--- | :--- | :--- |
| `FlowStartTime` | datetime | フローの開始時刻 (UTC)。 |
| `FlowEndTime` | datetime | フローの終了時刻 (UTC)。 |
| `FlowDirection` | string | フローの方向。`Inbound` (内向き) または `Outbound` (外向き)。 |
| `FlowStatus` | string | フローの状態。`Allowed` (許可) または `Denied` (拒否)。 |
| `FlowType` | string | フローのカテゴリ。`IntraVNet`, `InterVNet`, `AzurePublic`, `ExternalPublic` など。 |
| `CompletedFlows` | long | 完了したフローの数。 |
| `L4Protocol` | string | トランスポート層プロトコル。`T` (TCP) または `U` (UDP)。 |
| `L7Protocol` | string | アプリケーション層プロトコルの名前。 |
| `FlowEncryption` | string | フローの暗号化の種類。 |
| `IsFlowCapturedAtUdrHop` | bool | フローがUDR (ユーザー定義ルート) のホップでキャプチャされたかどうか。 |

### 通信量情報 (Traffic Volume Information)
| カラム名 | 型 | 説明 |
| :--- | :--- | :--- |
| `BytesSrcToDest` | long | 送信元から宛先へ送信された合計バイト数。 |
| `BytesDestToSrc` | long | 宛先から送信元へ送信された合計バイト数。 |
| `PacketsSrcToDest` | long | 送信元から宛先へ送信された合計パケット数。 |
| `PacketsDestToSrc` | long | 宛先から送信元へ送信された合計パケット数。 |
| `AllowedInFlows` | long | 許可された内向きフローの数。 |
| `AllowedOutFlows` | long | 許可された外向きフローの数。 |
| `DeniedInFlows` | long | 拒否された内向きフローの数。 |
| `DeniedOutFlows` | long | 拒否された外向きフローの数。 |

### 送信元情報 (Source Information)
| カラム名 | 型 | 説明 |
| :--- | :--- | :--- |
| `SrcIp` | string | 送信元IPアドレス。 |
| `SrcPorts` | string | パイプ (`|`) 区切りの送信元ポート番号。 |
| `SrcVm` | string | 送信元IPに関連付けられた仮想マシン名。 |
| `SrcSubnet` | string | 送信元IPに関連付けられたサブネット名。 |
| `SrcVnet` | string | 送信元IPに関連付けられた仮想ネットワークのGUID。 |
| `SrcNic` | string | 送信元IPに関連付けられたNIC (ネットワークインターフェース)。 |
| `SrcRegion` | string | 送信元リソースのAzureリージョン。 |
| `SrcSubscription` | string | 送信元リソースのサブスクリプションID。 |
| `SrcPublicIps` | string | 送信元のパブリックIPアドレス情報。 |
| `SrcLoadBalancer` | string | 送信元IPに関連付けられたロードバランサー。 |
| `SrcApplicationGateway` | string | 送信元IPに関連付けられたApplication Gateway。 |
| `SrcExpressRouteCircuit` | string | 送信元IPに関連付けられたExpressRoute回線。 |
| `SrcLocalNetworkGateway` | string | 送信元IPに関連付けられたローカルネットワークゲートウェイ。 |
| `SrcServiceTags` | string | 送信元パブリックIPのサービスタグ。 |

### 宛先情報 (Destination Information)
| カラム名 | 型 | 説明 |
| :--- | :--- | :--- |
| `DestIp` | string | 宛先IPアドレス。 |
| `DestPort` | int | 宛先ポート番号。 |
| `DestVm` | string | 宛先IPに関連付けられた仮想マシン名。 |
| `DestSubnet` | string | 宛先IPに関連付けられたサブネット名。 |
| `DestVnet` | string | 宛先IPに関連付けられた仮想ネットワークのGUID。 |
| `DestNic` | string | 宛先IPに関連付けられたNIC (ネットワークインターフェース)。 |
| `DestRegion` | string | 宛先リソースのAzureリージョン。 |
| `DestSubscription` | string | 宛先リソースのサブスクリプションID。 |
| `DestPublicIps` | string | 宛先のパブリックIPアドレス情報。 |
| `DestLoadBalancer` | string | 宛先IPに関連付けられたロードバランサー。 |
| `DestApplicationGateway` | string | 宛先IPに関連付けられたApplication Gateway。 |
| `DestExpressRouteCircuit` | string | 宛先IPに関連付けられたExpressRoute回線。 |
| `DestLocalNetworkGateway` | string | 宛先IPに関連付けられたローカルネットワークゲートウェイ。 |
| `DestServiceTags` | string | 宛先パブリックIPのサービスタグ。 |
| `Country` | string | パブリックIPアドレスの国コード (ISO 3166-1 alpha-2)。 |

### 関連リソース情報 (Related Resource Information)
| カラム名 | 型 | 説明 |
| :--- | :--- | :--- |
| `TargetResourceId` | string | フローログが有効になっているリソースのID。 |
| `TargetResourceType` | string | フローログが有効になっているリソースの種類 (VNET, SUBNET, NIC)。 |
| `MacAddress` | string | フローがキャプチャされたNICのMACアドレス。 |
| `NsgList` | string | フローに関連するNSG (ネットワークセキュリティグループ)。 |
| `NsgRule` | string | このフローを許可または拒否したNSGルール。 |
| `AclGroup` | string | 接続を許可/拒否したNSGまたはセキュリティ管理構成に関連するグループ。 |
| `AclRule` | string | 接続を許可/拒否したNSGルール名またはセキュリティ管理ルール名。 |
| `ConnectionName` | string | 接続名。 |
| `ConnectionType` | string | 接続の種類。`VNetPeering`, `VpnGateway`, `ExpressRoute` など。 |
| `PrivateEndpointResourceId` | string | プライベートエンドポイントのリソースID。 |
| `PrivateLinkResourceId` | string | プライベートリンクサービスのリソースID。 |
