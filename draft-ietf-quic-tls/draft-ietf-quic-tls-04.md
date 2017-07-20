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
