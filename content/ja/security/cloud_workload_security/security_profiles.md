---
further_reading:
- link: /security/cloud_workload_security/setup
  tag: Documentation
  text: CWS のセットアップ
- link: /security/cloud_workload_security/workload_security_rules
  tag: Documentation
  text: クラウドワークロードセキュリティルールの管理
kind: documentation
title: CWS Workload Security Profiles
---

Workload Security Profiles は、潜在的な脅威や誤構成の特定に役立つ行動学習モデルによって、予想されるワークロードのアクティビティのベースラインを提供します。この洞察は、既知の許容可能なワークロード動作に対する[抑制サジェストの生成](#suppress-signals based-on-suggestions)や、未知の異常動作の特定など、セキュリティアラートの調査時に利用することができます。

## 互換性

Workload Security Profiles を Datadog の構成と互換性を持たせるためには、[CWS を有効にし][1]、Datadog Agent 7.44 以降を使用し、アクティビティスナップショットを有効にしている必要があります。Workload Security Profiles はデフォルトで有効になっており、コンテナ化されたワークロードのみをサポートします。

## Workload Security Profiles の生成方法

Workload Security Profiles は、イメージ名とイメージバージョンのタグが共通するコンテナからの[アクティビティスナップショット](#activity-snapshots)がマージされてセキュリティプロファイルを形成するときに生成されます。コンテナイメージの各バージョンに対して、異なるセキュリティプロファイルが生成されます。その結果、`{ image-name, image-version}` タグの組み合わせごとに、1 つのプロファイルが生成されます。

各セキュリティプロファイルは、与えられたワークロードイメージの[行動モデル](#behavior-learning-model)として機能し、既知の許容できる行動を特定します。

### アクティビティスナップショット

アクティビティスナップショットは、Workload Security Profiles の構成要素です。各スナップショットは、プロセス、ネットワーク、およびファイルアクセスの詳細を含む、コンテナ上のカーネルレベルのアクティビティをキャプチャします。この情報は、Agent によって 30 分 (デフォルト) 間隔で収集され、Datadog のバックエンドに送信されます。

Agent の構成でアクティビティスナップショットを有効にすると、Agent は、Agent を起動したときにすでに実行されているコンテナを含む、実行中のコンテナのスナップショットを自動的に生成します。また、Linux カーネル制御グループまたは cgroups を使用するクラウドネイティブ環境での新しいコンテナのスナップショットもキャプチャされます。

各 Agent は、ホストシステム上の複数のコンテナを同時にプロファイルすることができます。パフォーマンスのオーバーヘッドを最小限に抑えるため、一度に 5 つ (デフォルト) 以上のコンテナがプロファイルされることはありません。

次の図は、複数のコンテナからのアクティビティスナップショットが 1 つの Workload Security Profile にマージされる方法を概要で表現したものです。共通性スコアは、指定されたプロセスやファイルのアクティビティが、通常の既知の動作である可能性がどれだけ高いかを表します。

{{< img src="security/cws/security_profiles/security-profiles-diagram.png" alt="自動生成されたセキュリティプロファイルを表示する CWS Security Profiles ページ" width="50%">}}

### 動作学習モデル

Workload Security Profiles は、特定のワークロードイメージとイメージバージョンのための動作モデルです。個々のコンテナからアクティビティスナップショットをキャプチャして比較することで、各セキュリティプロファイルは、同様のコンテナのワークロードの集計動作のモデルを提供することができます。より多くのコンテナのプロファイルが作成されると、この表現が特定のワークロードに対するより正確な動作モデルとなります。コンテナ化されたワークロードは不変であることが予想されるため、イメージ名とイメージバージョンのタグを共有するワークロード間には、ほとんど差異がないはずです。

プロセスノードとファイルおよびネットワークアクティビティの関係を含む動作モデルの視覚化は、[セキュリティプロファイルの詳細ページ](#explore-security-profiles)に表示されます。プロファイルの視覚化では、プロファイル化されたさまざまなコンテナで、これらの関係がどの程度一貫しているか、または共通しているかが表示されます。

#### プロファイルステータス

セキュリティプロファイルは、次の 2 つのステータスのいずれかを持つことができます。

- **Learning**: セキュリティプロファイルは、クラウドインフラストラクチャー内の異なるコンテナから Agent が収集したアクティビティスナップショットからまだモデル化されています。これらのコンテナは、イメージ名とイメージバージョンのタグが共通で、異なるホストや環境に存在することがあります。
- **Stable**: 受信したスナップショットの数、または最初のアクティビティスナップショットが処理されてからの経過時間において、指定されたアクティビティスナップショットのしきい値に到達しました。現在のデフォルトのしきい値は、200 スナップショットと 2 日 (48 時間) のうち、最初に到達したほうのしきい値です。

プロファイルが Stable ステータスになると、新しいバージョンとプロファイルが作成されるまで、そのプロファイルは不変となり、それ以上のアクティビティは追加されません。一方、プロファイルが Learning ステータスになると、追加のアクティビティが処理され、共通性スコアが更新されます。

## セキュリティプロファイルを調べる

CWS で生成されたセキュリティプロファイルは、[Security Profiles][2] ページで、イメージ名ごとに分類して表示されます。イメージ名をクリックすると、異なるイメージバージョンやイメージタグを含む、イメージに関連するセキュリティプロファイルの全リストが表示されます。各セキュリティプロファイルの詳細には、プロファイルを作成するためにマージされたアクティビティスナップショットの数、動作学習モデルのステータス、プロファイルが最後に更新された日付などが表示されます。

{{< img src="security/cws/security_profiles/security-profiles-overview.png" alt="自動生成されたセキュリティプロファイルを表示する CWS Security Profiles ページ" width="100%">}}

セキュリティプロファイルを選択すると、その詳細ページが表示されます。セキュリティプロファイルを構成するプロセス、関連するセキュリティシグナル、イメージ名でタグ付けされたインフラストラクチャーコンテナの観測可能性ステータス、プロファイルの作成に使用されたアクティビティスナップショットを確認できます。

### 共通性スコア

**Behavior Commonality** スコアは、全体的なセキュリティプロファイルレベルと、個々のプロセス、ファイル、およびネットワーク DNS アクティビティレベルの両方に存在します。プロファイルレベルのパーセンテージは、セキュリティプロファイルの動作モデルを構築するために使用されたすべてのアクティビティのスナップショットの集計レベルの類似性です。

{{< img src="security/cws/security_profiles/security-profile.png" alt="CWS セキュリティプロファイル詳細ページ" width="100%">}}

プロセスノードを選択すると、サイドパネルに追加の共通性スコアが表示され、セキュリティプロファイルを構成する異なるコンテナ間で、個々のプロセス実行、ファイルアクセス、およびネットワークリクエストがどの程度共通しているかが示されます。

一般に、共通性スコアが高いほど、指定されたプロセスやファイルのアクティビティが正常で既知の動作である可能性が高くなります。共通性パーセンテージは、学習段階ではより多くのアクティビティスナップショットがプロファイルに統合されるため変動する可能性がありますが、セキュリティプロファイルが安定状態に達すると一貫性を保つ必要があります。新しいバージョンのワークロードイメージをリリースすると、学習サイクルが再開され、新しいプロファイルが作成されるため、共通性スコアがリセットされます。

### セキュリティシグナルサイドパネル

セキュリティプロファイルの詳細は、[セキュリティシグナルエクスプローラー][3]のシグナルパネルにも表示されます。この情報を使用して、ワークロードのアクティビティをより理解し、潜在的な脅威と通常のワークロードの動作を区別するのに役立たせることができる可能性があります。また、**View Security Profile** リンクを使用して、関連するセキュリティプロファイルに直接移動することができます。

{{< img src="security/cws/security_profiles/signal-security-profile.png" alt="クラウドワークロードセキュリティのセキュリティプロファイルページ" width="80%">}}

## 提案に基づくシグナルの抑制

シグナルがセキュリティプロファイルの既知のワークロード動作と一致する場合、セキュリティシグナルのサイドパネルに抑制の提案が表示されます。この提案を受け入れるかどうかを選択する前に、対応するセキュリティプロファイルとシグナルのトリガーとなったプロセスを表示することができます。

提案を受け入れるには、**Suppress Signals** をクリックし、**Add Suppression to Rule** をクリックします。これにより、そのプロセスが将来セキュリティシグナルをトリガーするのを防ぐ抑制クエリがルールに作成されます。

{{< img src="security/cws/security_profiles/suppression-suggestion.png" alt="抑制の提案を示すセキュリティシグナルパネル" width="80%">}}

## その他の参考資料

{{< partial name="whats-next/whats-next.html" >}}

[1]: /ja/security/cloud_workload_security/setup
[2]: https://app.datadoghq.com/security/workload/profiles
[3]: /ja/security/explorer
[4]: /ja/security/cloud_workload_security/setup