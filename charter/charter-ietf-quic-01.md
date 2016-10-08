# Charter

QUIC ワーキンググループは、プレ標準化実装とデプロイの経験及び  
draft-hamilton-quic-transport-protocol、  
draft-iyengar-quic-loss-recovery、  
draft-shade-quic-http2-mapping、  
draft-thomson-quic-tls  
で記述されているデザインに基づいた、UDPベース、ストリームの多重化、暗号化されたトランスポートであるプロトコルの仕様のスタンダードトラックを提供します。

QUICの主な目的は
- HTTP/2を開始する際の、アプリケーションのための接続の確立とトランスポート全体のレイテンシの最小化
- ヘッドオブラインブロッキングのない多重化の提供
- デプロイのためにパスエンドポイントの変更のみを要求
- デフォルトでTLS1.3 を使用し、トランスポートの常時暗号を提供

グループの作業は5つの成果に対応する5つのエリアに主にフォーカスします。

これらのコアトランスポートの作業の1つめは、ワイヤフォーマット、それに加えてコネクションの確立の仕組み、ストリームの多重化、データの信頼性、ロス検出とリカバリ、輻輳制御、バージョンネゴシエーション、そしてオプションのネゴシエーションについて記述します。輻輳制御の作業は標準化された輻輳制御を使用してQUICのデフォルトの方式として記述されます。新しい輻輳制御の仕組みは明示的にスコープ外です。QUICは、高速で分散したデプロイメントと機能のテストのサポートを期待されます。

フォーカスするエリアの2つめはセキュリティです。この作業は、プロトコルが鍵ネゴシエーションのためにどのようにTLS1.3を使用するか記述し、それらの鍵をどのように使用してアプリケーションデータとQUICのヘッダ両方の機密性と完全性を保護する方法も記述します。この作業は、QUICが少なくともTCP(もしくはマルチパスを使用するときはMPTCP)を使用するTLS1.3のスタック構成と同じセキュリティとプライバシを持つことを確かにします。

フォーカスするエリアの3つめは、特定のアプリケーションプロトコルとQUICのトランスポート機能のマッピングについて記述します。最初のマッピングはQUICを使用したHTTP/2セマンティクスの説明です。特に、QUICを使用したWebのレイテンシの最小化です。このマッピングはHTTP/2の仕様で定義された仕組みの拡張も入ります。マッピングが完了したときに、Charterの更新をもってさらなるプロトコルが追加されるかもしれません。

フォーカスする4つめのエリアは、経路と負荷分散を復数の経路でマイグレーションするためのマルチパスを可能にするために、コアプロトコルの機能を拡張します。

フォーカスする5つめのエリアは、どのような状況下で、どのようにQUICを安全に使用できるを説明する、適用性と管理性のステートメントです。そして、プロトコル実装の管理性とデプロイメントについて記述します。


トランスポートプロトコルのネットワーク管理の現在の慣例は以下の機能を含みます、アクセスコントロールリスト(ACL)、equal-cost multipath routing (ECMP)のためのフローの振り分け、フローのシグナルの指向性、フローの作成と削除のシグナリング、課金目的のフローに関する情報の出力。QUICプロトコルはこれらの機能を有効にするように定義される必要はありませんし、TLS1.3を使用してる際のTCPで出来た方法と同じ方法で出来るようにうする必要もありません。しかし、ワーキンググループはRFC 7258で記述される、ネットワーク管理の慣例への影響を考慮しなければなりません。

部分的な信頼性のサポート、ネゴシエーションと前方誤り訂正(FEC)の使用はワーキンググループ Charterの現在のバージョンではスコープ外です。

プロトコルの仕組みを変更するのにも現在の仕組みを維持するのにもコンセンサスが必要とされることに注意して下さい。特に、最初の文書に含まれる物は、機能やそれらがどのように定義されるかのコンセンサスが得られたということを意味するものではありません。

QUICワーキンググループはHTTPbisワーキンググループと密接に作業を行います。特にQUIC上でのHTTP2のマッピングにおいてです。

以下に記載するマイルストーンを達成するために、グループは広範囲に中間ミーティングを使用することを期待します、特に最初の1年の間は。

# Milestones
```
May 2019	Multipath extension document to IESG  
Nov 2018	QUIC Applicability and Manageability Statement to IESG  
Nov 2018	HTTP/2 mapping document to IESG  
Mar 2018	TLS 1.3 Mapping document to IESG  
Mar 2018	Loss detection and Congestion Control document to IESG  
Mar 2018	Core Protocol document to IESG  
Nov 2017	Working group adoption of Multipath extension document  
Feb 2017	Working group adoption of QUIC Applicability and Manageability Statement  
Feb 2017	Working group adoption of HTTP/2 mapping document  
Feb 2017	Working group adoption of TLS 1.3 mapping document  
Feb 2017	Working group adoption of Loss detection and Congestion Control document  
Feb 2017	Working group adoption of Core Protocol document  
```
