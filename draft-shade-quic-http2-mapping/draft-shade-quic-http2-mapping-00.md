# HTTP/2 Semantics Using The QUIC Transport Protocol  
https://tools.ietf.org/html/draft-shade-quic-http2-mapping-00  

- [Abstract](#abstract)
- [1. Introduction](#1-introduction)
- [2. QUIC advertisement](#2-quic-advertisement)
- [3. Connection establishment](#3-connection-establishment)
- [4. Sending a request on an HTTP/2-over-QUIC connection](#4-sending-a-request-on-an-http2-over-quic-connection)
  - [4.1. Terminating a stream](#41-terminating-a-stream)
- [5. Writing data to QUIC streams](#5-writing-data-to-quic-streams)
- [6.  Stream Mapping](#6--stream-mapping)
  - [6.1.  Reserved Streams](#61--reserved-streams)
    - [6.1.1.  Stream 3: headers](#611--stream-3-headers)
    - [6.1.2.  Stream states](#612--stream-states)
- [7.  Stream Priorities](#7--stream-priorities)
- [8.  Flow Control](#8--flow-control)
- [9.  Server Push](#9--server-push)
- [10.  Error Codes](#10--error-codes)
- [11.  Other HTTP/2 frames](#11--other-http2-frames)
  - [11.1.  GOAWAY frame](#111--goaway-frame)
  - [11.2.  PING frame](#112--ping-frame)
  - [11.3.  PADDING frame](#113--padding-frame)
- [12.  Normative References](#12--normative-references)

# Abstract
QUICトランスポートプロトコルは、ストリームの多重化、ストリーム毎のフロー制御、低レイテンシのコネクション確立といったHTTP/2のトランスポートとして好ましい機能を有しています。この文書はQUIC上でのHTTP/2セマンティクスのマッピングについて記述します。

特に、この文書ではQUICに含まれるHTTP/2の機能を明確にし、その他の機能がQUIC上でどのように実装されるか記述します。

# 1. Introduction
QUICトランスポートプロトコルは、ストリームの多重化、ストリーム毎のフロー制御、低レイテンシのコネクション確立といったHTTP/2のトランスポートとして好ましい機能を有しています。この文書はQUIC上でのHTTP/2セマンティクスのマッピングについて記述します。

特に、この文書ではQUICに含まれるHTTP/2の機能を明確にし、その他の機能がQUIC上でどのように実装されるか記述します。

QUICは[[draft-hamilton-quic-transport-protocol]](https://tools.ietf.org/html/draft-hamilton-quic-transport-protocol-00) で記述されています。HTTP/2に関する記述は、[[RFC 7540]](https://tools.ietf.org/html/rfc7540)を参照ください。

# 2. QUIC advertisement

サーバはAlt-Svc HTTP レスポンスヘッダを使用してHTTP/2-over-QUICを通信できる事を広告します。Alt-Svcヘッダは非QUIC(例: HTTP/2 over TLS)上で送られたレスポンスのヘッダに含められ送信されます。
```
   Alt-Svc: quic=":443"
```
加えて、`v = parameter`という形でサーバが対応しているQUICバージョンのリストを 指定することが出来ます。例えば、サーバが バージョン33と34をサポートしている場合、以下のようにヘッダを指定します。
```
   Alt-Svc: quic=":443"; v="34,33"
```
このヘッダを受信すると、クライアントはport 443でQUICコネクションの確立を試みます。もし成功すれば、この文書で記述されたマッピングを使用してHTTP/2リクエストを送信します。

接続性の問題(例: ファイアフォールのUDPブロック)のためにQUICコネクション確立は失敗するかもしれません。その場合、クライアントはグレースフルに HTTP/2-over-TLS/TCP にフォールバックすべきです。

# 3. Connection establishment

[[draft-hamilton-quic-transport-protocol]](https://tools.ietf.org/html/draft-hamilton-quic-transport-protocol-00)で記述されている通りにHTTP/2-over-QUIC のコネクションは確立されます。QUICのcrypto handshakeはTLSを使用しなければなりません[[draft-thomson-quic-tls]](https://tools.ietf.org/html/draft-thomson-quic-tls-00) (MUST)。

core QUICプロトコルに関わるコネクションレベルのオプションは、最初のcrypto handshake[Crypto とTransportを合わせたHandshake]でセットされますが、HTTP2の設定はHTTP/2 SETTINGSフレームで伝達されます。QUICコネクションが確立された後、HTTP/2 SETTINGSフレームはQUICのheadersストリーム上で最初のフレームとして送られるでしょう(ストリームID 3、 Stream Mapping参照)。HTTP/2のように、追加のSETTINGSフレームは接続中にどちらのエンドポイントからでも送信される可能性があります。

TODO: HTTP/2のようにACKビットが設定された空のSETTINGSフレームを送信することでSETTINGSの受信したことの確認応答とするか、トランスポートレイヤの確認応答に依存させるか決定する必要がある。

HTTP/2-over-TCPが定義するする幾つかのSTTINGSフレームを通して設定されるトランスポートレイヤのオプションはHTTP/2-over-QUICではQUICトランスポートパラメータで置き換えられます。

- SETTINGS_HEADER_TABLE_SIZE
  - HTTP/2 SETTINGS フレームで送信される。
- SETTINGS_ENABLE_PUSH
  - HTTP/2 SETTINGS フレームで送信される。 [TBD, 現在は QUIC "SPSH" コネクションオプションを使用してセットされます]
- SETTINGS_MAX_CONCURRENT_STREAMS
  - QUICでは、"MSPC"タグを使用して、コネクション毎の受信ストリームの最大数は最初のcrypto handshakeで指定されるように要求します。HTTP/2 SETTINGSフレーム内のSETTINGS_MAX_CONCURRENT_STREAMSはエラーです。
- SETTINGS_INITIAL_WINDOW_SIZE
  - QUICでは、"SFCW" と "CFCW" タグを使用して、ストリームとコネクションのフロー制御のウィンドウサイズはcrypto handshakeで指定されるように要求します。HTTP/2 SETTINGSフレーム内のSETTINGS_INITIAL_WINDOW_SIZE はエラーです。
- SETTINGS_MAX_FRAME_SIZE
  - この設定はQUICでは同じではありません。HTTP/2 SETTINGSフレームで指定することはエラーです
- SETTINGS_MAX_HEADER_LIST_SIZE
  - HTTP/2 SETTINGS フレームで送信される。

HTTP/2-over-TCP のように、未知のSETTINGSパラメータは許容されますが、無視されます。SETTINGSパラメータは、ACKビットが設定された空のSETTINGSフレームの送信によって、受信の確認が行なわれます。

# 4. Sending a request on an HTTP/2-over-QUIC connection
確立されたQUICコネクション上でのHTTP/2リクエスト送信の概要は以下のとおりです。より詳細な内容はこの文書の後のセクションにあります。

まずクライアントはHPACKを使用してHTTPヘッダをエンコードし、HTTP/2 HEADERSフレームとして構成します。これらは ストリームID 3で送られます([Stream Mapping]参照)。正確なHEADERSフレームのレイアウトは Section 6.2 of [RFC7540] で記述されます。QUICではPADDINGフレームを同様の目的で使用するので、HTTP/2パディングは要求されません。

HEADERSはストリーム3上で送信されます、各HEADERSフレームで必須のストリームIDは、対応するリクエストボディを送信することができる QUIC ストリームIDを示しています。もし、ヘッダ以外のデータが無ければ、指定されたQUICデータストリームは使用されません。

## 4.1. Terminating a stream
ストリームは次の3つのいずれかの方法で終了することができます。

- ヘッダのみのリクエスト/レスポンスでは、END_STREAM bitのセットされたHEADERSフレームは、HEADERSフレームで指定されているストリームを終了します。
- トレーラーを持たずヘッダとボディを持つリクエスト/レスポンスでは、最後のQUIC STREAMフレームは FIN bitがセットされるます。
- トレーラー、ヘッダ、ボディを持つリクエスト/レスポンスは最後のQUIC STREAMフレーム FIN bitはセットされません。そして、トレーラーのHEADERSフレームはEND_STREAM bitがセットされるでしょう。

(TODO: HTTP/2ストリームのステートマシーンとQUICストリームのステートマシーンのマッピングを記述する)

# 5. Writing data to QUIC streams
QUICストリームはストーリム内でのバイト列の送信順番について信頼性を提供します。
ワイヤ上では、データはQUIC STREAMフレームにフレーム化されますが、このフレーム化はHTTP/2レイヤからは見えません。QUICの受信者は受信したSTREAMフレームをバッファし順番通りにするため、アプリケーションに見えるデータは信頼できるバイト列になります。

ストリーム3に書かれるバイト列はHTTP/2 HEADERフレームで(もしくは HTTP/2のDATAフレームでないフレーム)である必要がある一方、データストリームに書かれるバイト列は単純にリクエストやレスポンスのボディであるべきです。HTTP/2によってさらなるフレーミングは要求されません (例: HTTP/2 データフレームは使用されません)。もし、対応するHEADERSがストリーム3上で届く前にデータストリーム上でデータが届いたとき、そのときはデータはHEADERSが届くまでバッファされます。

# 6.  Stream Mapping
HTTP/2のヘッダとデータがQUIC上で送信された時、QUICレイヤがほとんどのストリーム管理を行います。HTTP/2 ストリームIDはQUICのストリームIDで置き換えられます。HTTP/2はQUICを使用してる時は明示的にフレーミングをする必要はありません。
QUICストリーム上で送られるデータは単純にHTTP/2のヘッダやボディから構成されます。対応する方向のQUICストリームがクローズした時、リクエストとレスポンスは完了したと考えられます。

HTTP/2のように、QUICはクライアントが開始するストリームに奇数のストリームIDを使用し、サーバが開始するストリームに偶数のストリームIDを使用します(例: server push)。HTTP/2とは違ってQUICでは2つの予約されたストリームIDがあります。

## 6.1.  Reserved Streams
ストリームID 1は暗号的操作(handshake, 暗号の設定更新)のために予約されており、HTTP/2のヘッダやボディのために使用してはいけません(MUST NOT)。[[core protocol doc]](https://tools.ietf.org/html/draft-hamilton-quic-transport-protocol-00)を参照下さい。ストリームID 3は HTTP/2 HEADERS フレームを送受信するのに予約されています。そのため、最初のクライアントが開始するデータストリームはストリームID 5になります。

サーバが開始するストリームの ストリームIDは予約されていません。最初のサーバがカイスするストリームはID 2、つづいてID 4となります。

### 6.1.1.  Stream 3: headers
HTTP/2-over-QUIC は[RFC7541] に記述されている HPACKヘッダ圧縮を使用します。HPACKはHTTP/2のためにデザインされており、TCPが提供するような順番通りの通信を仮定しています。
エンコードされたヘッダブロックの順序はエンドポイントがエンコードした順番で到着(そしてデコード)すべきです。これは両エンドポイントの状態の同期を保証します。

QUICストリームはストリーム上で送信されたデータの順番通りの配送を提供します。しかし、ストリーム間の配送順番は保証されません。QUICにおいてHEADERSフレームの順番通りの配送をするために、それらは全てストリーム3上で送信されます。別のデータストリーム上で届くデータ(リクエスト/レスポンスのボディ)は、ストリーム 3で対応するHEADERSフレームが届き読まれるまでバッファされます。

これは、head-of-line blockingを引き起こします。もしStream NのHEADERSフレームを含むパケットが損失するか順番が変わった場合、N+2のStreamのHEADERSが既に到着してたとしても、再送が成功するまで処理を進めることが出来ません。

Trailingヘッダ(トレーラー)もストリーム3上で送信されます。これらはHTTP/2のHEADERS フレームとして送信されますが、END_STREAMビットが設定されている必要があり、そして":final-offset"擬似ヘッダを含んでいる必要があります。QUICは順番通りでない配送をサポートしているため、END_STREAM bitがセットされたHEADERSフレームの受信はリクエスト/レスポンスボディ全てが受信されたことを保証するものではありません。
そのため、対応するデータストリーム上で送信されるボディの合計バイト数を示すために特別な":final-offset"擬似ヘッダがトレーラのHEADERSフレームに含まれます。これはQUICレイヤでいつ全てのリクエストが受信されたかを決定するのに使用され、その時がローカルのストリーム状態を破棄可能な時です。":final-offset"擬似ヘッダはHTTP/2レイヤに渡される前にHEADERSで取り外されます。

### 6.1.2.  Stream states
HEADERSフレームの順番通りでない配送を行うHTTP/2-over-QUICのマッピングは、HTTP/2のストリームの状態遷移図に変更をもたらします[https://tools.ietf.org/html/rfc7540#section-5.1]
特に "open"から"half close"への遷移と、half closed (local)" から "closed"への遷移は以下の場合のみ起きます

- ピアが明示的に次の方法でストリームを終了した時
  - END_STREAM bitがセットされたHTTP/2 HEADERSフレームで、trailingヘッダの場合は:final-offset擬似ヘッダを持つとき 
  - もしくは、QUIC streamフレームのFIN bitがセットされているとき
- リクエストやレスポンスボディ全てを受信した時

# 7.  Stream Priorities
HTTP/2-over-QUICは、RFC7540で記述されるHTTP/2の優先度方式を使用します。HTTP/2の優先度方式は、指定されたストリームを別のストリームに依存するように指定できるようにデザインされています。これは後者のストリーム(親ストリーム)が前者のストリーム("依存する側の"ストリーム)より先にリソースが割り当てられることを表現しています。依存関係を合わせると、コネクション中のストリームの依存関係は、依存関係ツリーを構成します。

依存関係ツリーの構成はHTTP/2 HEADERSフレームとPRIORITYフレームにより追加、削除、ストリーム間の依存の変更されます。この方式は、暗黙的に優先度の変更(例: 依存関係ツリーの変化)は順番通りの配送という概念になっています。サブツリーの親を再設定するような依存関係ツリーの操作は可換ではありません。

送信者と受信者の両者が同じ依存関係ツリーを持つために、両方がそれらを同じ順番で適用しなければなりません。HTTP/2はPRIORITYフレームと(オプショナルで)HEADERSフレームで優先度を割り当てるように指定しています。HTTP/2-over-QUICでHTTP/2の優先度変更を順番どおりに配送するために、HTTP/2のPRIORITYフレームと加えてHEADERSフレームは予約されているストリーム3で送信されます。Stream Dependency, Weight, Eflag, (HEADERSフレームの) PRIORITYフラグのセマンティクスはHTTP/2-over-TCPと同様です。

STREAMフレームはそれらが指定するストリームのためでるが、HEADERSとPRIORITYフレームは異なるストリーム上で送られるため、それらはSTREAMフレームに対して順番どおりに配送されないかもしれません。

これに対して何か特別に取り扱うことはありません、受信者は単純にもっとも最近に受け取った優先度の情報にのっとってリソースを割り当てるべきです。

ALTERNATIVE DESIGN: もしcore QUICプロトコルが優先度を実装した場合、このドキュメントはcoreプロトコルが提供する方式とHTTP/2の優先度方式との対応付けを行うべきです。
コンフリクトを防ぐために、HTTP/2のPRIORITYフレームの送信や、HEADERSフレームをPRIORITYフラグを設定して送ることは禁止されるでしょう。

# 8.  Flow Control
QUICはストリームとコネクションレベルのフロー制御を提供しています。HTTP/2におけるフロー制御の原理と似ていますが、いくつかの実装上の違いがあります。QUICでフロー制御が行われるので、HTTP/2マッピングは自身でフロー制御の状態を管理したり、いつどうやってピアに対してフロー制御のフレームを送信するか考慮する必要はありません。HTTP/2とのマッピングではHTTP/2のWINDOW_UPDATEフレームは送信してはいけません。

フロー制御の初期ウインドウサイズ(ストリームとコネクション)は、crypto handshake中に伝えられます(参照 [Connection establishment])これらの値を最大サイズに設定することで事実上フロー制御を無効にできます。QUICは使用方に基づいて自動でフロー制御のウィンドウを調整するため、相対的に小さい初期ウインドウが使用されます。詳細は [[draft-
   hamilton-quic-transport-protocol]](https://tools.ietf.org/html/draft-hamilton-quic-transport-protocol)を参照。

# 9.  Server Push
HTTP/2-over-QUICはHTTP/2のサーバプッシュをサポートしています。コネクションが確立されている間、クライアントはSETTINGS_ENABLE_PUSHをセットしたHTTP/2 SETTINGSフレームを通してサーバプッシュを受信したいかサーバに通知します(参照[Connection Establishment])。これはデフォルトで1(true)です。

HTTP/2-over-TCPでのサーバプッシュのように、サーバからプッシュされるストリームのストリームIDとリクエストのリクエストヘッダ値を含むHTTP/2のPUSH_PROMISEフレームを送信します。
PUSHのPROMISEフレームは他のHEADERSと非データフレームに対して適切な順序付けを保証するために、ストリーム3に送信されます。PUSH_PROMISEフレームの中で、一般的なHTTP/2フレームのヘッダは、新しくプッシュされるストリームに関連付けられるストリームのストリームIDを示します。しかしPromised Stream ID 値は新しくプッシュされるストリームのストリームID指定します。

サーバプッシュのレスポンスは、サーバプッシュではないレスポンスと同じように配送されます。予約されたストリーム 3上でHTTP/2 HEADERSフレームによってレスポンスヘッダや(もしあれば)トレーラが送信され、対応するPUSH_PROMISEフレームで指定されたストリーム上でQUICのstreamフレームを介してレスポンスボディが送信されます。

# 10.  Error Codes
RFC7540で定義されるHTTP/2 エラーコードは以下のようにQUICのエラーコードと対応付けられます。

- NO_ERROR (0x0)
　- QUIC_NO_ERROR に対応する
- PROTOCOL_ERROR (0x1)
  - 一つに対応しない?
- INTERNAL_ERROR (0x2)
  - QUIC_INTERNAL_ERROR? (coreプロトコルの仕様ではまだ定義されていない)
- FLOW_CONTROL_ERROR (0x3)
  - QUIC_FLOW_CONTROL_RECEIVED_TOO_MUCH_DATA? (coreプロトコルの仕様ではまだ定義されていない)
- SETTINGS_TIMEOUT (0x4)
  - ? ( SETTINGS ackをサポートするかに依存します)
- STREAM_CLOSED (0x5)
  - QUIC_STREAM_DATA_AFTER_TERMINATION
- FRAME_SIZE_ERROR (0x6)
  - QUIC_INVALID_FRAME_DATA
- REFUSED_STREAM (0x7)
  - ?
- CANCEL (0x8)
  - ?
- COMPRESSION_ERROR (0x9)
  - QUIC_DECOMPRESSION_FAILURE (coreプロトコルの仕様ではまだ定義されていない)
- CONNECT_ERROR (0xa)
  - ? (CONNECTをサポートするかに依存します)
- ENHANCE_YOUR_CALM (0xb)
  - ?
- INADEQUATE_SECURITY (0xc)
  - QUIC_HANDSHAKE_FAILED, QUIC_CRYPTO_NO_SUPPORT
- HTTP_1_1_REQUIRED (0xd)

   TODO: マッピングしていない部分を埋める.

# 11.  Other HTTP/2 frames
QUICはHTTP/2にある幾つかの機能を有しています。その場合はHTTP/2とのマッピングにおいてそれらを実装する必要はありません。
その結果、QUICを使用する場合、QUICレイヤで直接実装されているか他の方法で提供されているため、幾つかのHTTP/2フレームタイプは必須ではなくなります。
このセクションではこれらの場合について記述します。

## 11.1.  GOAWAY frame
QUICは独自にGOAWAYフレームを有しており、QUICの実装はアプリケーションからGOAWAYの送信を可能にしても良いです。
QUICにおけるGOAWAYの送信のセマンティクスはHTTP/2と同様です。
GOAWAYを送信したエンドポイントはストリームのオープンを続けますが、ストリームの作成を受け付けることはありません。

QUICのGOAWAYフレームの詳細は[[draft-hamilton-quic-transport-protocol]](https://tools.ietf.org/html/draft-hamilton-quic-transport-protocol-00)で記述されます。

## 11.2.  PING frame

QUICは独自にPINGフレームを有しており、今はアプリケーションから触れます。
もしコネクション上にアクティブなデータストリームがない場合は、QUICクライアントは定期的にPINGをサーバに送信します。
QUICのPINGフレームの詳細は[[draft-hamilton-quic-transport-protocol]](https://tools.ietf.org/html/draft-hamilton-quic-transport-protocol-00)で記述されます。

## 11.3.  PADDING frame
このマッピングにおいて、HTTP/2のパディングに対応するものはありません。パディングは代わりにQUICレイヤでパケットのペイロードにQUIC PADDINGを付加する形で代替されます。QUIC over HTTP/2へのマッピングでは、エンドポイント間でのフロー制御の矛盾を防ぐためにHTTP/2でのパディングはエラーとして扱います。


# 12.  Normative References

   [RFC2119]  Bradner, S., "Key Words for use in RFCs to Indicate
              Requirement Levels", March 1997.

   [RFC7540]  Belshe, M., Peon, R., and M. Thomson, "Hypertext Transfer
              Protocol Version 2 (HTTP/2)", May 2015.

   [RFC7541]  Peon, R. and H. Ruellan, "HPACK: Header Compression for
              HTTP/2", May 2015.

   [draft-hamilton-quic-transport-protocol]
              Hamilton, R., Iyengar, J., Swett, I., and A. Wilk, "QUIC:
              A UDP-Based Multiplexed and Secure Transport", July 2016.

   [draft-thomson-quic-tls]
              Thomson, M. and R. Hamilton, "Porting QUIC to TLS", March
              2016.

   [draft-iyengar-quic-loss-recovery]
              Iyengar, J. and I. Swett, "QUIC Loss Recovery and
              Congestion Control", July 2016.
