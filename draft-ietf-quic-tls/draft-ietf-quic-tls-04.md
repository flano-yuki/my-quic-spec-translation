QUIC                                                      M. Thomson, Ed.
インターネットドラフト                                    Mozilla
Intended status: Standards Track                          S. Turner, Ed.
期限: 2017年12月15日
                                                           2017年6月13日

                    セキュアなQUICのためのTLSの使用
                         draft-ietf-quic-tls-04

# Abstract

 この文書はどのようにセキュアなQUICにTLSが使われるかを説明します

# 読者への注釈

 このドラフトの議論は https://mailarchive.ietf.org/arch/search/?email_list=quic.
 にアーカイブされる QUICワーキンググループメーリングリスト (quic@ietf.org) で行われています

 ワーキンググループの情報は https://github.com/quicwg で見ることが出来ます。
 このドラフトのためのソースコードと提案の一覧は https://github.com/quicwg/base-drafts/labels/tls.
 で見ることが出来ます。

# この文書の状況

  このインターネットドラフトはBCP78とBCP79の規定に完全に準拠し提出されます。

  インターネットドラフトは Internet Engineering Task Force (IETF)の書類の作業です。
  他の団体への通達もまたインターネットドラフトとして書類の作業が配布します。
  現行のインターネットドラフトのリストは http://datatracker.ietf.org/drafts/current/ です。

  インターネットドラフトは最大六ヶ月草稿文書の有効性をもち、
  いつでも他の文書により更新、置換、廃止されるかもしれません。
  インターネットドラフトを参照文献や"作業中"としてではなく参照することは不適切です。
  このインターネットドラフトは2017/12/15に廃止される予定です。


# 著作権表示

   Copyright (c) 2017 IETF Trust and the persons identified as the
   document authors.  All rights reserved.





Thomson & Turner        Expires December 15, 2017               [Page 1]

Internet-Draft                QUIC over TLS                    June 2017


   This document is subject to BCP 78 and the IETF Trust's Legal
   Provisions Relating to IETF Documents
   (http://trustee.ietf.org/license-info) in effect on the date of
   publication of this document.  Please review these documents
   carefully, as they describe your rights and restrictions with respect
   to this document.  Code Components extracted from this document must
   include Simplified BSD License text as described in Section 4.e of
   the Trust Legal Provisions and are provided without warranty as
   described in the Simplified BSD License.

# 目次

   1.  導入  . . . . . . . . . . . . . . . . . . . . . . . .   3
   2.  表記と用語  . . . . . . . . . . . . . . . . . . .   4
   3.  プロトコル概略 . . . . . . . . . . . . . . . . . . . . . .   4
     3.1.  TLS 概略  . . . . . . . . . . . . . . . . . . . . . .   5
     3.2.  TLS ハンドシェイク . . . . . . . . . . . . . . . . . . . . . .   6
   4.  TLS 使用法 . . . . . . . . . . . . . . . . . . . . . . . . . .   7
     4.1.  ハンドシェイク と 設定手順  . . . . . . . . . . . . . .   7
     4.2.  TLSへのインターフェース  . . . . . . . . . . . . . . . . . . . .   9
       4.2.1.  ハンドシェイク インターフェース . . . . . . . . . . . . . . . . .   9
       4.2.2. ソースアドレス検証   . . . . . . . . . . . . . .  10
       4.2.3.  鍵準備イベント   . . . . . . . . . . . . . . . . . .  11
       4.2.4.  暗号鍵エクスポート  . . . . . . . . . . . . . . . . . . . .  12
       4.2.5.  TLS インターフェース まとめ . . . . . . . . . . . . . . . .  12
     4.3.  TLS バージョン . . . . . . . . . . . . . . . . . . . . . . .  13
     4.4.  ClientHello サイズ  . . . . . . . . . . . . . . . . . . . .  13
     4.5.  ピアー 認証 . . . . . . . . . . . . . . . . . . .  13
     4.6.  TLS エラー  . . . . . . . . . . . . . . . . . . . . . . .  14
   5.  QUIC パケット保護  . . . . . . . . . . . . . . . . . . .  14
     5.1.  新しい鍵の入手 . . . . . . . . . . . . . . . . . . .  14
     5.2.  QUIC 鍵 拡張  . . . . . . . . . . . . . . . . . . .  15
       5.2.1.  0-RTT 秘密鍵  . . . . . . . . . . . . . . . . . . . .  15
       5.2.2.  1-RTT 秘密鍵 . . . . . . . . . . . . . . . . . . . .  15
       5.2.3.  パケット保護 鍵 と IV  . . . . . . . . . . . .  17
     5.3.  QUIC AEAD 使用法 . . . . . . . . . . . . . . . . . . . . .  17
     5.4.  Packet Numbers  . . . . . . . . . . . . . . . . . . . . .  18
     5.5.  保護されたパケットの受信 . . . . . . . . . . . . . . .  19
     5.6.  Packet Number 差分  . . . . . . . . . . . . . . . . . . .  19
   6.  暗号化されないパケット . . . . . . . . . . . . . . . . . . . . .  19
     6.1.  整合性確認手順  . . . . . . . . . . . . . . .  19
     6.2.  64-ビット FNV-1a アルゴリズム . . . . . . . . . . . . . . .  20
   7.  キーフェーズ  . . . . . . . . . . . . . . . . . . . . . . . . .  20
     7.1.  TLSハンドシェイクのためのパケット保護 . . . . . . . . .  21
       7.1.1.  初期鍵遷移 . . . . . . . . . . . . . . .  21
       7.1.2.  再送と保護されないパケットの承認 Packets . . . . . . . . . . . . . . . . . . . . . . .  22
     7.2.  鍵の更新  . . . . . . . . . . . . . . . . . . . . . . .  23



Thomson & Turner        Expires December 15, 2017               [Page 2]

Internet-Draft                QUIC over TLS                    June 2017


   8.  クライアントアドレス検証  . . . . . . . . . . . . . . . . . .  24
     8.1.  HelloRetryRequest アドレス検証   . . . . . . . . . .  24
       8.1.1.  Stateless アドレス検証  . . . . . . . . . . . .  25
       8.1.2.  HelloRetryRequestの送信 . . . . . . . . . . . . . .  26
     8.2.  NewSessionTicket アドレス検証 . . . . . . . . . . .  26
     8.3.  アドレス検証トークンの整合性  . . . . . . . . . . .  27
   9.  ハンドシェイク前 QUIC メッセージ . . . . . . . . . . . . . . . . .  27
     9.1.  ハンドシェイクの完了以前の保護されないパケット . . . .  28
       9.1.1.  STREAM フレーム . . . . . . . . . . . . . . . . . . . .  28
       9.1.2.  ACK フレーム  . . . . . . . . . . . . . . . . . . . . .  28
       9.1.3.  データの更新とストリーム制限 . . . . . . . . . .  29
       9.1.4.  保護されないパケットに伴うサービスの拒否  . . . . .  29
     9.2.  0-RTT 鍵の使用 . . . . . . . . . . . . . . . . . . . .  30
     9.3.  順不同の保護されたフレームの受信 . . . . . . . . .  30
   10. TLSハンドシェイクへのQUIC拡張  . . . . . . . .  31
     10.1.  プロトコルとヴァージョンのネゴシエーション . . . . . . . . . . . .  31
     10.2.  QUIC トランスポートパラメータ拡張 . . . . . . . . . .  31
     10.3.  0-RTTの誘導 . . . . . . . . . . . . . . . . . . . . .  32
   11. セキュリティへの考察 . . . . . . . . . . . . . . . . . . .  32
     11.1.  パケット反射攻撃の緩和  . . . . . . . . . .  33
     11.2.  ピアーのサービス拒否 . . . . . . . . . . . . . . . . .  33
   12. エラーコード . . . . . . . . . . . . . . . . . . . . . . . . .  33
   13. IANA Considerations . . . . . . . . . . . . . . . . . . . . .  34
   14. 参照  . . . . . . . . . . . . . . . . . . . . . . . . .  34
     14.1.  基本的な参照 . . . . . . . . . . . . . . . . . .  34
     14.2.  有益な参照 . . . . . . . . . . . . . . . . .  35
   Appendix A.  Contributors . . . . . . . . . . . . . . . . . . . .  36
   Appendix B.  Acknowledgments  . . . . . . . . . . . . . . . . . .  36
   Appendix C.  Change Log . . . . . . . . . . . . . . . . . . . . .  36
     C.1.  Since draft-ietf-quic-tls-02  . . . . . . . . . . . . . .  36
     C.2.  Since draft-ietf-quic-tls-01  . . . . . . . . . . . . . .  36
     C.3.  Since draft-ietf-quic-tls-00  . . . . . . . . . . . . . .  37
     C.4.  Since draft-thomson-quic-tls-01 . . . . . . . . . . . . .  37
   Authors' Addresses  . . . . . . . . . . . . . . . . . . . . . . .  37

1.  導入

この文書はどのようにQUICがトランスポートレイヤーセキュリティ(TLS)　ヴァージョン1.3 [T-D.ietf-tls-tls13]
を用いて保護されているかを説明します。
TLS1.3は以前のバージョンに対して、コネクション確立においての決定的なレイテンシーの向上
を提供します。パケット損失がなければ、ほとんどの新たなコネクションは
単一のラウンドトリップでの確立と保護を行うことが出来ます。
同一のサーバとクライアントの間で、続くコネクションにおいて、
クライアントはしばしば直ちにデータを送ることができます。
すなわち、ゼロラウンドトリップ設立を使っていることを意味します。

この文書は標準化されたTLS1.3がQUICのセキュリティ部分としてどのように振る舞うかを説明します。
TLS1.2に対しても同様の設計を行うことが出来ます。
ですが、QUICの提供する利点のいくらかは、TLS1.3を用いたハンドシェイクレイテンシー
を扱うさいにより感じることが出来ます。

2.  表記と用語

   The words "MUST", "MUST NOT", "SHOULD", and "MAY" are used in this
   document.  It's not shouting; when they are capitalized, they have
   the special meaning defined in [RFC2119].

   This document uses the terminology established in [QUIC-TRANSPORT].

   For brevity, the acronym TLS is used to refer to TLS 1.3.

   TLS terminology is used when referring to parts of TLS.  Though TLS
   assumes a continuous stream of octets, it divides that stream into
   _records_. Most relevant to QUIC are the records that contain TLS
   _handshake messages_, which are discrete messages that are used for
   key agreement, authentication and parameter negotiation.  Ordinarily,
   TLS records can also contain _application data_, though in the QUIC
   usage there is no use of TLS application data.

3.  プロトコル概略

QUIC [QUIC-TRANSPORT] は気密性への責任とパケットの保護生後性を仮定します。
これはTLS1.3コネクションより導入された鍵を使用します。
また、QUICはTLS1.3の認証とセキュリティと性能に重大なパラメータのネゴシエーション
に依存します。

厳密に分割されると言うよりは、これら２つのプロトコルは相互に依存します。
QUICはTLSハンドシェイクを用い、TLSはQUICストリームから提供される信頼性と
順序付けられた配送を用います。

この文書はどのようにQUICがTLSとやり取りをするかを定義します。
これはどのようにTLSが使われ、どのように鍵の素材がTLSから提供され、
またQUICのパケットの保護のために鍵の素材の申請が使われるかの説明を含みます。
図1は呼び出されるQUICパケット保護と共に基本的なTLSとQUICの間のやりとりを説明します。











Thomson & Turner        Expires December 15, 2017               [Page 4]

Internet-Draft                QUIC over TLS                    June 2017


   +------------+                        +------------+
   |            |------ Handshake ------>|            |
   |            |<-- Validate Address ---|            |
   |            |-- OK/Error/Validate -->|            |
   |            |<----- Handshake -------|            |
   |   QUIC     |------ Validate ------->|    TLS     |
   |            |                        |            |
   |            |<------ 0-RTT OK -------|            |
   |            |<------ 1-RTT OK -------|            |
   |            |<--- Handshake Done ----|            |
   +------------+                        +------------+
    |         ^                               ^ |
    | Protect | Protected                     | |
    v         | Packet                        | |
   +------------+                             / /
   |   QUIC     |                            / /
   |  Packet    |-------- Get Secret -------' /
   | Protection |<-------- Secret -----------'
   +------------+

                    図 1: QUIC と TLS のやり取り

QUICコネクションの初期状態はなんらかの保護の形式なくパケットを交換します。
この状態においてQUICはstream 0と関連したパケットのの使用に制限されます。
Stream 0はTLSコネクションのために予約されます。
これはover TCPとして階層づけられたときに現れる完全なTLSコネクションとされるものです。
ただ一つの違いは、QUICは信頼性とTCPによって提供されるときと違った順序付けを提供することです。

TLSハンドシェイクの間に対しかな点は鍵の素材はQUICが用いるためにTLSコネクションから提供されることです。
この鍵の素材はパケット保護鍵の提供のために使われます。
どのようにまた、いつ鍵が導かれるかの詳細はセクション5に含まれます。

# 3.1.  TLS 概略

TLSは２つのエンドポイントが信用できない媒体（インターネット）を越えて
観察、修正、偽造されない交換されるメッセージを確保する方法を確立することを
提供します。

TLSの昨日は２つの基本機能に分割することができます。
認証された鍵交換と記録の保護です。
QUICは主にTLSに寄ってい提供される認証された鍵交換を主に用い、
自らでパケット保護を提供します。

TLSの認証された鍵交換は２つの実体、クライアントとサーバの間で起こります。
クライアントは交換をはじめ、サーバが応答します。



Thomson & Turner        Expires December 15, 2017               [Page 5]

Internet-Draft                QUIC over TLS                    June 2017


もし鍵交換が完全に成功すると、クライアントとサーバとの両方は
秘密鍵を了承します。TLSは事前共有鍵（PSK)とデッフィーハフマン（DH)鍵交換の
両方を保証します。PSKは0-RTTの基礎です。
DH鍵が消滅したとき、後者は完全な前方安全性(PFS)を提供します。

TLSハンドシェイクの完了のあと、
クライアントはサーバーへの一意性を認証し備え、また
サーバは必要に応じてクライアントへの一意性を認証し備えることができます。
TLSはX.509[RFC5280] 証明書ベース認証をサーバとクライアントの両方にサポートします。

TLS鍵交換は攻撃者による改ざんに抵抗しまたいかなる特定のピアーに操作されることのない
秘密鍵の処理です。

# 3.2.  TLS ハンドシェイク

TLS 1.3はQUICにとって興味深い２つの基本的なハンドシェイクの様式を提供します。

- クライアントが1ラウンドトリップののちアプリケーションデータを送ることの出来、
サーバーがクライアントからの最初のハンドシェイクメッセージ受け取った後即座に行える
を完全な1RTTハンドシェイク

-  クライアントが前回知った即座にアプリケーションデータを送るためのサーバの情報を用いる0-RTTハンドシェイク。
このアプリケーションデータは攻撃者によって再現されうるため、これはべき等性のない自己完結した原因によって
引き起こされてはいけません（MUST NOT)

0-RTT アプリケーションデータを共に、単純化されたTLS1.3ハンドシェイクは図1に示されます。
より詳細は [I-D.ietf-tls-tls13]を見てください。

       Client                                             Server

       ClientHello
      (0-RTT Application Data)  -------->
                                                     ServerHello
                                            {EncryptedExtensions}
                                                       {Finished}
                                <--------      [Application Data]
      (EndOfEarlyData)
      {Finished}                -------->

      [Application Data]        <------->      [Application Data]

                    図 2: TLS Handshake with 0-RTT




Thomson & Turner        Expires December 15, 2017               [Page 6]

Internet-Draft                QUIC over TLS                    June 2017


この0-RTTハンドシェイクはクライアントとサーバが以前通信したときのみ可能です。
1-RTTハンドシェイクにおいて、クライアントは保護されたアプリケーションデータを
それがサーバによって送られるハンドシェイクメッセージをすべて受け取るまで送ることが出来ません。

基本的なハンドシェイク交換の２つの追加されたバリエーションはこの文書に関係します。

- サーバはHelloRetryRequestをもつ ClientHelloに応答することが出来ます。
これは基本的な基本的な交換より余分なラウンドトリップを加えます。
これはもしサーバがクライアントから異なる鍵交換鍵を要求することを望むなら必要です。
HelloRetryRequestはクライアントがたしかにそれの主張するアドレスのパケットを受け取ることが
出来る証明のためにも用いられます。 ([QUIC-TRANSPORT] を参照してください)

-事前共有鍵方式は後に続くハンドシェイクのために公開鍵操作の数々を短縮するために使われます。
これは残りのコネクションが新たなデッフィー・ハフマン鍵で保護されるとしても、 0-RTTの基本です。

# 4.  TLS 使用法

QUUCはTLS接続のためにストリーム番号0を予約します。
ストリーム0はTLSレコード層を含む完全なTLS接続を含みます。
QUICのための拡張(Section 10.2を参照)の定義以外は、この使用んタメにTLSは修正されません。
これはTLSが気密性とそのレコードへの整合性のある保護を適用することを意味します。
特にTLSレコード保護はサーバによって送られるTLSハンドシェイクメッセージのための機密性保護のためにあります。

QUICはクライアントがストリームが始まる最初のパケットからフレームを送ることを許可します。
クライアントからの初期化パケットは、クライアントからの最初のTLSハンドシェイクメッセージを含む
stream 0のストリームフレームを含みます。
これはTLSハンドシェイクが最初のパケットからクライアントが送信することを許可します。

QUICパケットはQUICの機能の方式を用いて保護されます。Section 5を参照してください。
鍵はTLS Exporterを用いて可能になるTLSコネクションから提供されます。
( [I-D.ietf-tls-tls13] の Section 7.5とSection 5.2を参照してください)
TLSから鍵が提供された後、QUICは自らの鍵の詳細を管理します。


# 4.1.  ハンドシェイクと設定手順

TLSハンドシェイクとのQUICの統合は図3で詳細に示されています。
ストリーム0のQUIC "STREAM" フレームはTLSハンドシェイクを配送します。
QUICはこのストリームへの損失回復を行いまた、TLSハンドシェイクメッセージが
正しい順序で配送されることを保証します。



Thomson & Turner        Expires December 15, 2017               [Page 7]

Internet-Draft                QUIC over TLS                    June 2017


       Client                                             Server

   @C QUIC STREAM Frame(s) <0>:
        ClientHello
          + QUIC Extension
                               -------->
                           0-RTT Key => @0

   @0 QUIC STREAM Frame(s) <any stream>:
      Replayable QUIC Frames
                               -------->

                                         QUIC STREAM Frame <0>: @C
                                                  ServerHello
                                     {TLS Handshake Messages}
                               <--------
                           1-RTT Key => @1

                                              QUIC Frames <any> @1
                               <--------
   @C QUIC STREAM Frame(s) <0>:
        (EndOfEarlyData)
        {Finished}
                               -------->

   @1 QUIC Frames <any>        <------->      QUIC Frames <any> @1

                     Figure 3: QUIC over TLS Handshake

   図3において, 記号は以下のように意味します :

   -  "<" と ">" は ストリーム番号を囲います.

   -  "@" はQUICパケットを保護するために使われる鍵を示します。
   (C = 平文、整合性のために表記します。; 0 = 0-RTT 鍵; 1 = 1-RTT 鍵).

   -  "(" と ")" は TLS 0-RTT ハンドシェイクかアプリケーション鍵で保護されたメッセージを囲います

   -  "{" と "}" は TLSハンドシェイク鍵で保護されたメッセージを囲います

もし0-RTTが未遂なら、クライアントは0-RTT鍵のよって保護されません。
その場合、クライアントにおける鍵の変化は平文のパケットから1-RTTへ遷移します。
保護に、TLSハンドシェイクメッセージの最後のパケットが送られる後に起こります。


Thomson & Turner        Expires December 15, 2017               [Page 8]

Internet-Draft                QUIC over TLS                    June 2017


Note: クライアントはハンドシェイクの間に２つの異なった種類の平文パケットを用います。
Client Initial パケットはTLS ClientHelloメッセージを運送します。TLSハンドシェイクの残りは
Client Cleartext パケットに運ばれます。

サーバは保護されないTLSハンドシェイクメッセージ(@C)を送ります。
ハンドシェイクメッセージの最後が送られた後、
サーバの遷移は無保護(@C)から完全な 1-RTT 保護(@1) になります。

いくつかのTLSハンドシェイクメッセージはTLSレコード保護によって保護されます。
これらの鍵はQUICでの使用のためのTLSコネクションから提供されません。

0-RTTデータを送るとき、平文(@C)から0-RTT 鍵(@0)へのクライアント遷移、
そしてTLSハンドシェイクメッセージの２つめの送信の後に続く1-RTT 鍵(@1)。
これは1-RTTに保護されたパケットを受信するための保護されないパケットの見込みを
作ります。

鍵遷移の詳細はSection7.1に含まれます。


# 4.2. TLSへのインターフェース

図1に示されるように、QUICからTLSへの接続は４つの主要な機能です。
ハンドシェイク、ソースアドレス検証、鍵準備イベント、秘密鍵提供

追加の機能はTLSの変更を必要とするでしょう。

# 4.2.1.  ハンドシェイク インターフェース

ハンドシェイクを運用するため、TLSはstream 0で送受信できることを要請します。
このインターフェースのためにふたつの基本的な機能があります、
一つはQUICがハンドシェイクメッセージを要求すること、
もうひとつはQUICがハンドシェイクパケットをどこで提供するか
です。

QUICが配送されるべきトランスポートパラメータ(Section 10.2を参照)付きのTLSを提供するハンドシェイクを
始める前に、

QUICクライアントはTLSからTLSハンドシェイクオクテットを始めます。
クライアントは最初のパケットを送る前にハンドシェイクオクテットを得ます。





Thomson & Turner        Expires December 15, 2017               [Page 9]

Internet-Draft                QUIC over TLS                    June 2017

QUICサーバはstream 0オクテットの提供のための処理を始めます。

毎回、stream 0のデータをエンドポイント受け取る時、
それは可能ならTLSにオクテットを配送します。

stream 0においてデータをうけとるときは毎回、
可能ならTLSにオクテットを配送します。
新しいデータと共に提供されるTLSは毎回、新たしいハンドシェイクオクテットがTLSから
提供されます。
もし受け取ったハンドシェイクメッセージが不完全かそれが送るためのデータがないなら
TLSはいかなるオクテットを提供しないかもしれません。

一度、TLSハンドシェイクが完了した場合、
これはTLSが送る必要なるいかなる最終ハンドシェイクオクテットに合わせてQUICへ示されます。

一度ハンドシェイクが完了したなら、TLSは受動的になります。
TLSはまだそのピアーからデータを受け取り、同じように応答することが出来ます。
しかし、それは特定の要求なしにより多くのデータを送る必要はありません、
アプリケーションにもQUICにもです。
データを送る一つの理由はサーバがクライアントへセッションチケットの
追加もしくは更新を提供することを望むかもしれないからです。

ハンドシェイクが完了するとき、ただQUICはTLSにstream 0へ来るいかなるデータを提供
するです。
同じ理由で、ハンドシェイクの間、新しいデータはTLSから受信されたデータを提供することを要求されます。


Important:
ハンドシェイクが完了と通知されるまで、コネクションと鍵交換は主に
サーバで適切に認証されません。
クライアントからサーバへ最初のハンドシェイクメッセージを受け取った後
1-RTT鍵が使用可能であっても、サーバはクライアントの終了メッセージを
受取り検証するまで認証されたものと考えることは出来ません。

クライアント終了メッセージを待つサーバへの要件は配達されたメッセージへの
依存を作ります。
クライアントは複数のパケットに終了メッセージが運ばれたSTREAMフレームの複製が送られること
が暗示するhead-of-line blockingの可能性を避けることが出来ます。
これはサーバがこれらのパケットへのただちに行うことをできるようにします。

# 4.2.2.  ソースアドレス検証

TLS ClientHelloの処理の間、TLSは転送がクライアントからソースアドレス検証を要求するかどうか
決めることを要求します。

セッションを再開する初期TLS ClientHello はセッションチケットに
アドレス検証トークンを含みます。
これを0-RTT におけるすべての試行を含みます。
   During the processing of the TLS ClientHello, TLS requests that the
   transport make a decision about whether to request source address
   validation from the client.

   An initial TLS ClientHello that resumes a session includes an address
   validation token in the session ticket; this includes all attempts at



Thomson & Turner        Expires December 15, 2017              [Page 10]

Internet-Draft                QUIC over TLS                    June 2017

もしクライアントがセッション再開を試みないなら、
トークンは存在しません。初期ClientHelloの処理の間、
TLSはQUICに存在するなんらかのトークンを提供します。
レスポンスにおいて、QUICはみっつのレスポンスから一つを返します。

- コネクションの処理
- クライアントアドレス検証への応答
- コネクションの中断


もしQUICがソースアドレス検証を要求するなら、
それもまた新たなソースアドレス検証トークンを提供します。
TLSはTLS HelloRetryRequest メッセージのクッキー拡張の要求されたなんらかの情報を含みます。
その他の場合、コネクションは処理するかもしくはハンドシェイクエラーとともに中断します。


クライアントは２つめのClientHelloの中のクッキー拡張に応答します。
有効なクッキー拡張を含むClientHello は常にHelloRetryRequestへのレスポンスが存在します。
もしアドレス検証がQUICから要求されたなら、これはアドレス検証トークンを含むでしょう。
TLSはQUICの二回目のアドレス検証リクエスト - クッキー拡張から展開された値を含みます -を作ります。
このリクエストに対するレスポンスにおいて、QUICはクライアントアドレス検証に応答できません。
それは中断かコネクション試行の進行を許可することのみができます。

QUICはハンドシェイクが完了したあといつでもセッション再送のために使われる新たな
アドレス検証トークンを提供できます。
新たなトークンが提供されたときはいつでも、
TLSがNewSessionTicketメッセージが生成した、チケットの中に含まれたトークンが提供されます。

クライアントアドレス検証の詳細はSection 8を見てください。

# 4.2.3.  Key Ready イベント

0-RTT 鍵や1-RT鍵が使用のために準備されているとき、
TLSはQUICにシグナルと共に提供します。
これらのイベントは非同期ではなく、
彼らは常にTLSにが新たなハンドシェイクオクテットを提供されたあと、
もしくはTLSがハンドシェイクオクテットを処理するあと、
ただちにに起こります。

TLSがハンドシェイクを完了したとき、1-RTT 鍵はQUICへ提供されることができます。
クライアントとサーバの両方に置いて、これはTLS Finished メッセージを送信したあと
起こります。

この順序はフレームに運ばれたTLSハンドシェイクメッセージがアプリケーションデータ利用できるの
と同時に送る準備ができることを意味します。
実装はTLSハンドシェイクメッセージが暗号文パケットの中で送られることを保証しなければいけません(MUST)。



Thomson & Turner        Expires December 15, 2017              [Page 11]

Internet-Draft                QUIC over TLS                    June 2017

パケットの分割はデータに1-RTT 鍵に保護される必要を要求します。

もし、0-RTT が可能なら、クライアントがTLS ClientHello メッセージを送信するか
サーバがそのメッセージを受け取った後にそれは準備されます。
最初のハンドシェイクオクテットをQUICクライアントに提供した後、
TLSスタックは0-RTT 鍵が準備できたことを通知するかもしれません。
サーバにおいて、ClientHello メッセージを含むハンドシェイクオクテットを受け取った後、
TLSサーバは0-RTT 鍵が使用できることを通知するかもしれません。

1-RTT鍵が双方向に置いてパケットのために使われます。
0-RT 鍵はただクライアントから送られるパケットを保護するためにのみ使われます。

# 4.2.4.  暗号鍵エクスポート

どのようにして暗号鍵がTLSから提供されるかの詳細は5.2章に
含まれます。


# 4.2.5.  TLS インタフェース概略

図4はクライアントとサーバの両方に対して
QUICとTLSの間の交換を要約します。

   Client                                                    Server

   Get Handshake
   0-RTT Key Ready
                         --- send/receive --->
                                                 Handshake Received
                                                    0-RTT Key Ready
                                                      Get Handshake
                                                   1-RTT Keys Ready
                        <--- send/receive ---
   Handshake Received
   Get Handshake
   Handshake Complete
   1-RTT Keys Ready
                         --- send/receive --->
                                                 Handshake Received
                                                      Get Handshake
                                                 Handshake Complete
                        <--- send/receive ---
   Handshake Received
   Get Handshake

            図4: QUICとTLSの間の相互関係の概要





Thomson & Turner        Expires December 15, 2017              [Page 12]

Internet-Draft                QUIC over TLS                    June 2017


# 4.3.  TLS バージョン

この文書はどのようにTLS 1.3 [I-D.ietf-tls-tls13] がQUICとともに使われるか
を説明します。

実践に置いて、TLSハンドシェイクは仕様のためにTLSのバージョンを交渉するでしょう。
これはもし両方のエンドポイントがバージョンをサポートしているのなら、
交渉された1.3より新しいTLSのバージョンが得ることができます。
これは新しいバージョンによりサポートされたQUICにより使われるTLS1.3の機能が
提供されることを受け入れられます。

不適切に設定されたTLS実装はTLS1.3かそれより古いTLSのバージョンと交渉することができます。
エンドポイントはもしTLS1.3より古いバージョンが交渉されたなら、
コネクションを終了しなくてはいけません(MUST)

# 4.4.  ClientHello サイズ

  QUICはクライアントからの初期化ハンドシェイクパケットがひとつのパケットのペイロードとして
  調整されていることを要求します。
  QUICパケットにおけるサイズ制限はClientHelloを保持するレコードを
  1197オクテットに焼成することを意味します。

  TLS ClientHelloは十分なスペースの残りを備えることが出来ます。
  しかしながら、制限の超過を起こすのに十分な変数が存在します。
  実装は巨大なセッションチケットやHelloRetryRequestクッキー、
  複数もしくは大きなカギ共有、サポートされた暗号、
  シグネチャーアルゴリズム、バージョンの長いリスト、
  QUICトランスポートパラメタまたほかのネゴシエーションパラメタ
  と拡張はメッセージの増量を引き起こすことがあります。


  サーバに対して、セッションチケットと HelloRetryRequest クッキー拡張のサイズは
  接続するクライアントの能力において影響を与えることができます。
  小さな数の増加率を選ぶことはそれらの数値がクライアントにより正しく使われることが出来ます。
  
  TLS実装はClientHelloが十分に大きいことを必要としません。
  QUIC PADDING フレームは必要に応じてパケットのサイズを増加するために追加されます。

# 4.5.  ピアー認証

  認証への要件は使われるアプリケーションプロトコルに依存します。
  TLSはサーバ認証を提供しサーバがクライアント認証を要求することを許可します。

  クライアントはサーバに一意性を認証しなくてはいけません(MUST)
  これは特にサーバの一意性が証明書に含まれまた証明書は信用された
  実体に提案されることの検証を含みます。 (example [RFC2818]を見てください)




Thomson & Turner        Expires December 15, 2017              [Page 13]

Internet-Draft                QUIC over TLS                    June 2017



  サーバはクライアント認証をハンドシェイクの間に要求しても良いです(MAY)
  サーバはもし要求されたときクライアントの認証ができないときコネクションを
  拒否しても良いです(MAY)
  クライアント認証への要件はアプリケーションプロトコルと実装に
  よって変化します。

  サーバは ポストハンドシェイククライアント認証をつかってはいけません(MUST NOT)
  ([I-D.ietf-tls-tls13] の 4.6.2章を参照してください)


# 4.6.  TLS エラー

  TLSコネクション時のエラーはstream 0のTLSアラートを用いて通知されるべきです(SHOULD)
  ハンドシェイクにおける失敗はTLS_HANDSHAKE_FAILED失敗の
  QUIC コネクションエラー として扱われるべきです(SHOULD)
  一度ハンドシェイクが完了したら、
  TLSアラートの送受信が起こったTLSコネクションにおけるエラーは
  TLS_FATAL_ALERT_GENERATEDもしくはTLS_FATAL_ALERT_RECEIVEDタイプの
  QUIC コネクションエラーとして扱われるべきです(MUST)
   respectively.

# 5.  QUIC パケット保護

   QUICパケットの保護はパケットの認証された暗号化を提供します。
   これはパケットのコンテクストに対して機密性と完全な保護を提供します。
   パケット保護はTLS コネクションから提供された鍵を用います。(5.2章を参照してください)

  QUICパケット保護とTLS レコード保護には異なる鍵が用いられます。
  TLS ハンドシェイクメッセージはもっぱらTLSレコード保護と共に保護されます。
  しかしポストハンドシェイクメッセージはQUICパケット保護とTLSレコード
  の両方と共に残念ながら保護されます。
  このメッセージは制限されるため、追加のオーバヘッドは小さいです。

# 5.1.  新しい鍵の入手

TLSがキーマテリアルの可用性の通知によって、
パケット保護鍵と初期化ベクター(IV)は更新されます。(5.2章を参照してください)
AEAD関数の選択もまたTLSにより交渉された適切なAEADのために更新されます。

任意の保護されないハンドシェイクパケット以外のパケット(7.1章を参照)のために
一度鍵の交換が行われたなら、
より大きなパケットナンバーを伴うパケットでは新たなキーマテリアルによって
送信されなくてはいけません(MUST)。

新しい鍵が導入されるたびにそれらのパケットのKEY_PHASEビットは、
新たな鍵の複製の使用を通知するために反転されます。



Thomson & Turner        Expires December 15, 2017              [Page 14]

Internet-Draft                QUIC over TLS                    June 2017


新たなパケットにおいて、エンドポイントはストリームデータを再送します。
新たなパケットは新たしいパケットナンバーをもち、また最新の
パケット保護鍵を用います。
この単純化された鍵管理は鍵が更新されたときのものです。

   An endpoint retransmits stream data in a new packet.  New packets
   have new packet numbers and use the latest packet protection keys.
   This simplifies key management when there are key updates (see
   Section 7.2).

# 5.2.  QUIC 鍵拡張

QUICはそのシステムが使うTLSにおいて構造化されたパケット保護秘密鍵、鍵そしてIVの
システムを用います。
QUICがその鍵管理にの基本として用いる秘密鍵はTLSエクスポーターによって
得られます。

QUICは鍵導出のためにTLSが交渉したのと同様のハッシュ関数とともにHKDFを用います。
たとえば、もしTLSがTLS_AES_128_GCM_SHA256を用いるならば、
SHA-256ハッシュ関数が用いられます。

# 5.2.1.  0-RTT 秘密鍵

0-RTT 鍵はTLSハンドシェイクの完了というよりコネクションの再開に使われる鍵です。
0-RTT回gを用いて送られるデータは再生されるかもしれません、
またその仕様に置いていくつかの制約があります、9.2章を見てください。
0-RTT鍵はClientHelloの送受信ののちに使われます。

暗号鍵はエクスポーターラベル "EXPORTER-QUIC 0-RTT Secret"と空のコンテクスト
を使うTLSから提供されます。
暗号鍵のサイズはTLSに交渉されたPRFハッシュ関数のためのハッシュの出力サイズ
でなくてはいけません（MUST)
これはTLS early_exporter_secretを用います。
QUIC 0-RTT 暗号鍵はクライアントから送られるパケットの保護のためだけに使われます。

      client_0rtt_secret
          = TLS-Exporter("EXPORTER-QUIC 0-RTT Secret"
                         "", Hash.length)

# 5.2.2.  1-RTT 暗号鍵

1-RTT鍵はTLSハンドシェイクが完了したあとクライアントとサーバの両方で使われます。
これらの２つの鍵はいつでも使われます。
ひとつはクライアントから送られうパケットのパケット保護鍵の導出のために使われ、
もうひとつはサーバが送られるパケットにおいてパケット保護鍵のためです。

初期化クライアントパケット保護鍵は
エクスポーターラベル "EXPORTER-QUIC client 1-RTT Secret"を用いた
TLSから提供されます。
初期化サーバパケット保護鍵はエクスポーターラベル "EXPORTER-QUIC server  1-RTT Secret"
を用います。
両方のエクスポーターは空のコンテクストを用います。
暗号鍵のサイズはTLSにより交渉されたPRFハッシュ関数のためにハッシュ出力のサイズでなければいけません(MUST)



Thomson & Turner        Expires December 15, 2017              [Page 15]

Internet-Draft                QUIC over TLS                    June 2017


      client_pp_secret_0
          = TLS-Exporter("EXPORTER-QUIC client 1-RTT Secret"
                         "", Hash.length)
      server_pp_secret_0
          = TLS-Exporter("EXPORTER-QUIC server 1-RTT Secret"
                         "", Hash.length)

これらの秘密鍵は初期化クライアントとサーバパケット保護鍵の導出に用いられます。
鍵更新ののち、これらの秘密鍵は[I-D.ietf-tls-tls13] の7.1章によって定義されるHKDF-Exp-Label 関数
を用いて更新されます。
HKDF-Expand-LabelはTLSにより交渉されたPRFハッシュ関数を用います。
秘密鍵の置換は クライアントには"QUIC Client 1-RTT Secret"のラベルの
サーバには"QUIC Server 1-RTT Secret"のラベル、
空のHashValue、そしてそのPRFのためにTLSが選択したハッシュ関数と
同じ出力の長さの現在の秘密鍵
を用いて導かれます。


      client_pp_secret_<N+1>
          = HKDF-Expand-Label(client_pp_secret_<N>,
                              "QUIC client 1-RTT Secret",
                              "", Hash.length)
      server_pp_secret_<N+1>
          = HKDF-Expand-Label(server_pp_secret_<N>,
                              "QUIC server 1-RTT Secret",
                              "", Hash.length)

これは必要に応じて作られた新しい秘密鍵の継承を許可します。
   This allows for a succession of new secrets to be created as needed.

HKDF-Expand-Labelは以下のように特別にフォーマットされたパラメータとともに
HKDF-Expand [RFC5869] を用います。

       HKDF-Expand-Label(Secret, Label, HashValue, Length) =
            HKDF-Expand(Secret, HkdfLabel, Length)

       Where HkdfLabel is specified as:

       struct {
           uint16 length = Length;
           opaque label<10..255> = "TLS 1.3, " + Label;
           uint8 hashLength;     // Always 0
       } HkdfLabel;

例として、クライアントパケット保護秘密鍵がパラメータを使用するには





Thomson & Turner        Expires December 15, 2017              [Page 16]

Internet-Draft                QUIC over TLS                    June 2017


      info = (HashLen / 256) || (HashLen % 256) || 0x21 ||
             "TLS 1.3, QUIC client 1-RTT secret" || 0x00

# 5.2.3.  パケット保護鍵とIV

完全な鍵拡張は[I-D.ietf-tls-tls13]の7.3章に定義されるような鍵拡張のために
同様の処理を用い、入力された秘密鍵に対して異なる値を用います。
QUICはTLSに交渉されたAEAD関数を用います。

クライアントから送られる0-RTTパケット保護のために用いられたパケット保護鍵とIVは
QUIC 0-RTT秘密鍵から導出されます。
クライアントとサーバから送られた1-RTTパケットはパケット保護鍵とIVは
QUIC 0-RTT 秘密鍵は各々client_pp_secretとserver_pp_secretの現在の生成
から導出されます。
出力の長さはTLSに選ばれたAEAD関数の要件によって決定されます。
鍵の長さはAEAD鍵長です。
任意のsecret Sに対して、以下のように導出された鍵とIVは対応します。


      key = HKDF-Expand-Label(S, "key", "", key_length)
      iv  = HKDF-Expand-Label(S, "iv", "", iv_length)

QUICレコード保護は最初キーマテリアルなしで開始します。
TLSステートマシーンはClientHelloが送られることを報告したとき、
0-RTT鍵は書き込みのために生成し入手されます。
TLSステートマシーンはハンドシェイクの完了を報告したとき、
1-RTT鍵は書き込みのために生成され入手されます。

# 5.3.  QUIC AEAD 仕様

   The Authentication Encryption with Associated Data (AEAD) [RFC5116]
   関数はTLSコネクションにおける使用のために交渉されたAEADがQUICパケット保護の
   ために用いられます。
   たとえば、もしTLSがTLS_AES_128_GCM_SHA256を用いているのなら、
   AEAD_AES_128_GCM関数が用いられます。

正規のQUICパケットはAEADアルゴリズム[RFC5116]によって保護されます。
Version negotiationとpublic resetは保護されません。

一度TLSが鍵を提供したなら、
正規のQUICパケットのコンテンツは任意のTLSメッセージが送られた後直ちに
TLSによって選ばれたAEADによって保護されます。

鍵Kはクライアントパケット保護鍵(client_pp_key_n)また
サーバパケット保護鍵(server_pp_key_n)のどちらかです。
5.2症の定義に由来します。




Thomson & Turner        Expires December 15, 2017              [Page 17]

Internet-Draft                QUIC over TLS                    June 2017

ナンスNはパケット保護IV(client_pp_iv_nかserver_pp_iv_nのどちらか)と
パケットナンバーの結合によって構成されます。
ネットワークバイトオーダーに再構築されたQUICパケットナンバーの64ビットは
IVのサイズの左をゼロで埋められています。
埋められたパケットなんばーとIVの排他的論理和が
AEADナンスを構築します。


関連するデータAは、AEADにおけるQUICヘッダーのコンテンツ,は
汎用ヘッダーのフラグオクテットから始まります。

入力の平文PはAEADにおけるパケットナンバーから始まる
QUICフレームです。[QUIC-TRANSPORT]において説明されるように。

出力の暗号文Cは、AEADのPに代わって転送されるものです。

TLSが鍵を提供する前はレコード保護なしで振る舞い平文Pが修正なく転送されます。

