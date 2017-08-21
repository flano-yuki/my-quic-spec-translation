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

