- [Abstract](#abstract)
- [1.Introduction](#1introduction)
- [2. Protocol Overview](#2-protocol-overview)
  - [2.1. Handshake Overview](#21-handshake-overview)
- [3. TLS in Stream 1](#3-tls-in-stream-1)
  - [3.1. Handshake and Setup Sequence](#31-handshake-and-setup-sequence)
- [4. QUIC Record Protection](#4-quic-record-protection)
  - [4.1. Key Phases](#41-key-phases)
    - [4.1.1. Retransmission of TLS Handshake Messages](#411-retransmission-of-tls-handshake-messages)
    - [4.1.2. Key Update](#412-key-update)
  - [4.2. QUIC Key Expansion](#42-quic-key-expansion)
  - [4.3. QUIC AEAD application](#43-quic-aead-application)
  - [4.4. Sequence Number Reconstruction](#44-sequence-number-reconstruction)
- [5. Pre-handshake QUIC Messages](#5-pre-handshake-quic-messages)
  - [5.1. Unprotected Frames Prior to Handshake Completion](#51-unprotected-frames-prior-to-handshake-completion)
    - [5.1.1. STREAM Frames](#511-stream-frames)
    - [5.1.2. ACK Frames](#512-ack-frames)
    - [5.1.3. WINDOW_UPDATE Frames](#513-window_update-frames)
    - [5.1.4. Denial of Service with Unprotected Packets](#514-denial-of-service-with-unprotected-packets)
  - [5.2. Use of 0-RTT Keys](#52-use-of-0-rtt-keys)
  - [5.3. Protected Frames Prior to Handshake Completion](#53-protected-frames-prior-to-handshake-completion)
- [6. QUIC-Specific Additions to the TLS Handshake](#6-quic-specific-additions-to-the-tls-handshake)
  - [6.1. Protocol and Version Negotiation](#61-protocol-and-version-negotiation)
  - [6.2. QUIC Extension](#62-quic-extension)
  - [6.3. Source Address Validation](#63-source-address-validation)
  - [6.4. Priming 0-RTT](#64-priming-0-rtt)
- [7. Security Considerations](#7-security-considerations)
  - [7.1. Packet Reflection Attack Mitigation](#71-packet-reflection-attack-mitigation)
  - [7.2. Peer Denial of Service](#72-peer-denial-of-service)
- [8. IANA Considerations](#8-iana-considerations)
- [9.  References](#9--references)
  - [9.1.  Normative References](#91--normative-references)
  - [9.2.  Informative References](#92--informative-references)
- [Appendix A.  Acknowledgments](#appendix-a--acknowledgments)

# Abstract
この文書はQUICでどのようにTLSを使用できるか記述する

# 1.Introduction
QUIC [I-D.tsvwg-quic-protocol]  はHTTP[RFC7230]セマンティクスのために多重化されたトランスポートを提供し、HTTP/1.1 や HTTP/2 over TCPより幾つかの重要な利点を持ちます。

この文書はQUICでどのようにTransport Layer Security (TLS)1.3を使用するか記述する。TLS1.3は以前のバージョンに比べコネクションの確立のレイテンシが改善されています。パケットロスがなければ、殆どの新しいコネクションは1回のラウンドトリップで安全に確立されます。同じクライアントとサーバ間での後続のコネクションは、クライアントはおおむねすぐにアプリケーションデータを送信できます。これが、0ラウンドトリップ セットアップです。

この文書は標準化されたTLS1.3がどのようにQUICのセキュリティコンポーネントとして動作するか記述します。同じようなデザインがTLS1.2でも動作できますが、QUICが提供する幾つかの利点は、TLS1.3より前のTLSでは接続レイテンシによって実現されます。


# 2. Protocol Overview
QUIC [I-D.tsvwg-quic-protocol]は幾つかのモジュールに分割できます。

1. 基本的なフレームエンベロープは通常のパケットレイアウトを示します。このレイヤはコネクションの識別、バージョンのネゴシエーション、フレームとpublic resetを識別するためのマーカーを含みます

2. public resetは保護されていないパケットであり、中間装置(セキュリティ的コンテキストではないエンティティ)がQUICコネクションの切断を要求できるようにします

3. バージョンネゴシエーションフレームは使用するQUICのバージョンを合意するのに使用されます

4. フレームングはQUICプロトコルの大部分を含みます。フレーミングは幾つかの異なるタイプのフレームを提供し、それらは特定の目的を持ちます。フレーミングはフレームに対して輻輳管理とストリームの多重化をサポートします。さらにフレーミングは生存性の有効テストを提供します(PINGフレーム)

5. 暗号化はフレームの機密性と完全性を提供します。全てのフレームはストリーム1上で実行されるTLSコネクションによって得られるキーマテリアルに基いて保護されます。それより先のデータは0-RTT鍵によって保護されます。

6. 多重化されたストリームは主にQUICのペイロードです。信頼性、データの順番通りの配送機能を提供し、encryption handshakeやトランスポートパラメータ(stream 1)、HTTPヘッダフィールド(stream3)、HTTPリクエストとレスポンスの送信に使われます。多重化管理のためのフレームは、ストリームの作成及び削除やフローコントロールや優先度制御のフレームを含みます

7. 輻輳管理はパケットの確認応答とリンクのキャパシティを有効に使用するために必要なその他のシグナルを含みます

8. TLSコネクションはstream 1上で行われます。これはTLSレコードレイヤすべてを含みます。TLSコネクションが特定の状態になると、keying materialは残りのQUICのトラフィックを保護するためにQUICのエンクリプションレイヤーに渡されます。

9. HTTPマッピングはHTTP/2に基づいたHTTPの適応を提供します。

これらのコンポーネントの相対的な関係は、図1に示されています。

```
   +-----+------+
   | TLS | HTTP |
   +-----+------+------------+
   |  Streams   | Congestion |
   +------------+------------+
   |         Frames          +--------+---------+
   +   +---------------------+ Public | Version |
   |   |     Encryption      | Reset  |  Nego.  |
   +---+---------------------+--------+---------+
   |                   Envelope                 |
   +--------------------------------------------+
   |                     UDP                    |
   +--------------------------------------------+
   
               図1 QUIC Structure
```

この文書ではQUICの暗号部分について定義します。これはstream 1上で交換されるハンドシェイクメッセージ及び、全てのフレームの暗号及び認証に使用されるレコードプロテクションを含みます。

## 2.1. Handshake Overview
TLS1.3はQUICにとって重要な基本的な2つのハンドシェイクモードが有ります。
- フルハンドシェイクは、クライアントは1回のラウンドトリップ後に、サーバはクライアントからの最初のメッセージを受信した直後からアプリケーションデータを送信できます

- 0-RTT ハンドシェイクは、クライアントは直ちに送信するのにサーバに関する情報を使用します。このデータは攻撃者によってリプレイ可能なため、それ自身で完結する非冪等なアクションを引き起こす契機になるものを送ってはいけません(MUST NOT)

0-RTTのアプリケーションデータを持つ単純化されたTLS1.3のハンドシェイクは図2に示されます、オプションのより詳細は [I-D.ietf-tls-tls13]を参照ください。

```
       Client                                             Server

       ClientHello
      (Finished)
      (0-RTT Application Data)
      (end_of_early_data)        -------->
                                                     ServerHello
                                            {EncryptedExtensions}
                                            {ServerConfiguration}
                                                    {Certificate}
                                              {CertificateVerify}
                                                       {Finished}
                                <--------      [Application Data]
      {Finished}                -------->

      [Application Data]        <------->      [Application Data]

                         図2 TLS Handshake with 0-RTT
```

さらに2つのバリエションが基本のハンドシェイク交換がこのドキュメントに関連します。

- サーバはClientHelloにHelloRetryRequestで応答できます。これは基本的な交換に1回のラウンドトリップを追加します。これは、もしサーバが鍵交換に別の鍵をクライアントに要求する際に必要です。HelloRetryRequestはクライアントが所持していると主張しているアドレスでパケットを正しく受信できることを検証するのにも使用されます。

- 事前共有鍵モードは、後続のハンドシェイクで公開鍵のやり取りを避けるのに使用されます。接続の残りは新しいDiffie-Hellman鍵交換によって保護されるとしても、これが0-RTTデータの基礎になります。

# 3. TLS in Stream 1
QUICはstream 1上で暗号的ハンドシェイクを行います、これはキーイング・、アテリアルのネゴシエーションはQUICプロトコルが開始してた後に発生することを意味します。これはQUICがTLSハンドシェイクのパケットの配送順番と信頼性を確実にするため、TLSの使用を単純化出します。

QUIC Stream 1は完全なるTLSコネクションを転送します。これはTLSコネクションの全てのTLSレコードレイヤを含みます(*)。QUICはこのストリーム上でTLSハンドシェイクメッセージの配信順番と信頼性を提供します。

TLSハンドシェイクが完了する前はにもQUICフレームは交換できます。しかしながら、これらのフレームは認証・機密に保護されません。Section 5がこの段階のQUICの合意設計と制限の操作についてカバーしています。

一度完了スロt、QUICフレームはQUICレコードプロテクションを使用して保護されます。Section 4を参照。


## 3.1. Handshake and Setup Sequence
TLSハンドシェイクを行うQUICの整合性は、図3でより詳細に確認できます。stream 1上のQUIC STREAMフレームはTLSハンドシェイクを配送します。QUICはハンドシェイクのパケットが損失した場合は再送を行い、順番を正しく並び替えを行い確実にすることに責任をもちます(*)

```
    Client                                             Server

@A QUIC STREAM Frame(s) <1>:
     ClientHello
       + QUIC Setup Parameters
                            -------->
                         0-RTT Key -> @B

@B QUIC STREAM Frame(s) <1>:
     (Finished)
   Replayable QUIC Frames <any stream>
                            -------->

                                      QUIC STREAM Frame <1>: @B/A
                                               ServerHello
                                      {Handshake Messages}
                            <--------
                        1-RTT Key -> @C

                                                 QUIC Frames @C
                            <--------
@B QUIC STREAM Frame(s) <1>:
     (end_of_early_data <1>)
     {Finished}
                            -------->

@C QUIC Frames              <------->            QUIC Frames @C
                           
                   図3 QUIC over TLS Handshake
```

図3の記号は下記を意味する
- "<" と ">" に囲まれているのが ストリームの数字
- "@" はQUICのパケットを保護するのに使用される鍵のフェーズを示します
- "(" と ")" で囲まれるのが、TLS 0-RTTハンドシェイクやアプリケーションキーで保護されるメッセージです
- "{" と "}"で囲まれるのが、 TLSハンドシェイクの鍵で保護されるメッセージです

もし 0-RTTが不可能な場合、その時はクライアントは (@B)の0-RTT 鍵で保護されたフレームは送信しません。 クライアント上での鍵の移行(*)は、平文(@A)から1-RTT保護(@C)になります。

もし サーバによって0-RTTデータが受理されなかった場合、その時はサーバは(@A)保護無しでハンドシェイクメッセージを送ります。クライアントは@Aから@Bに移行しますが、平文のServerHelloを受信した場合は0-RTTデータの送信を停止でき、1-RTTデータを直ちに処理できます。

# 4. QUIC Record Protection
QUICはパケットの認証つき暗号に責任を持つレコード保護レイヤを提供します。レコード保護レイヤは、パケットの内容の完全性と機密性を提供するために、TLSコネクションにより得られる鍵と認証付き暗号をを使用します。

TLSレコードの保護とQUICでは異なる鍵が使用されます。QUICとTLSレコード保護で別の鍵を持つことは、TLSレコードは２つの異なる鍵で保護されていることを意味します。この冗長性は簡素化のために維持されます。

## 4.1. Key Phases
鍵移行を生じさせるTLSハンドシェイクメッセージを送った直後に、新しいQUIC鍵の使用への移行が行われます。アウトバウンドのメッセージ保護のために新しい鍵セットを使用するときはいつでも、publicフラグ内のKEY_PHASE bitはトグルされます。KEY_PHASE bitは暗号化されてないメッセージでは0になります。

publicフラグのKEY_PHASE bit は最も重要なbitです(0x80).

KEY_PHASE bitは受信者が交換の契機のとるメッセージの受信を必要としない、キーイングマテリアルの変更を検知可能にします。これは復号の試行に依存した鍵移行に関するhead-of-line blockingを回避します。

以下の移行が定義されています
- クライアントは、ClientHelloの送信後に 0-RTT鍵の使用へと移行します。この移行では、KEY_PHASE bitクライアントによって送信されたパケットのKEY_PHASE bit を1にセットします(*)
- サーバは、ServerHelloを送信する前に0-RTT鍵の使用へと移行しますが、クライアントからのearly dataデータが受信された場合のみです。この移行は、サーバによって送信されるパケットのKEY_PHASE bit を1にセットします。もしサーバが0-RTTデータを拒否した場合、サーバのハンドシェイクメッセージはQUICレベルのレコード保護なしにKEY_PHASE bitは0で送信されます。それでもTLSハンドシェイクメッセージはTLSハンドシェイクトラフィック鍵に基づいて、TLSレコード保護によって保護されます(*)。
- サーバはFinishedメッセージの送信後に1-RTT鍵の使用へ移行します。KEY_PHASE bit に、もしearly dataが受理されていれば0を、サーバがearly dataを拒否していれば 1をセットします。

- Finished メッセージの送信後にクライアントは 1-RTT鍵の使用へ移行します。クライアントからの後続のメッセージのKEY_PHASE は、もし0-RTTデータを送信していれば 0を、ソレ以外では1をセットします。

- 両方のピアはTLS KeyUpdate メッセージを送信した後に、すぐに新しい鍵によって保護されたメッセージの送信を開始します。

それぞれのポイントで、TLSによって使用される両方のキーイングマテリアルとAEAD関数は、現在アウトバウンドパケットの保護に使用される値によって取り替えられます。一度鍵交換が行われると、高いシーケンスナンバーを持つパケットは、より新しい鍵(及びAEAD)が使用されるまでは、新しいキーイングマテリアルを使用しなければなりません(MUST)。

一度新しい鍵で保護されたパケットが受信されると、受信者は短い間以前の鍵を保持しておくべきです。古い鍵を保持しておくことで、受信者は順番が入れ替わったパケットをデコードできます。エンドポイントが、新しい鍵を使用しているもっと最小のシーケンス番号より小さいシーケンスナンバーの全てのパケットを受信した場合、又はパケットの順番の入れ替えが発生していないと判断できた場合は、鍵は破棄されるべきです(SHOULD)。0-RTT鍵はハンドシェイクが完了するまで保持されるべきです。

KEY_PHASE bitは使用されている鍵を直接示しません。0-RTTデータが送信され受信されたかに依存して、同じ秘密値によって得られた鍵で保護されたパケットでも異なるKEY_PHASE 値を持つかもしれません。

### 4.1.1. Retransmission of TLS Handshake Messages
TLSハンドシェイクメッセーは、元々それらを保護していたのと同じ暗号的保護レベルで再送される必要があります。新しい鍵はTLSメッセージを転送するQUICパケットの保護には使用できません。

アプリケーションデータ鍵の計算はハンドシェイクメッセージの内容に依存するため、クライアントはサーバからの 1-RTT鍵で保護されているハンドシェイクメッセージの再送を複合できません。この制限は、一度それが可能になると、送信するのに常に新しい鍵を使用する要求の例外を作成することになります。区別して受信できるように、サーバは再送されたハンドシェイクメッセージに元と同じKEY_PHASEを付加しなければなりません。

### 4.1.2. Key Update
一度TLSハンドシェイクが完了すると、KEY_PHASE bitは鍵更新の契機となるTLS KeyUpdate メッセージの受信の必要なく、メッセージの処理を許可します(*)
これは、KeyUpdateのロスによってピアがメッセージを判読不能にすること無く、エンドポイントが直ちに更新された鍵を使用できるようにします(*)。

エンドポイントは一度に鍵更新を開始しては行けません(MUST NOT)。エンドポイントがピアから一致するKeyUpdate メッセージを受信するまで、新しい鍵交換は送信できません。
又は、もしエンドポイントが元の鍵の更新を開始しない場合は、自身のKeyUpdateへの確認応答を受信するまでです。

任意の時点間で２つの鍵を区別出来ることを確実にし、KEY_PHASE がbitで十分にしています(*)

```
   Initiating Peer                    Responding Peer

@M KeyUpdate
                    New Keys -> @N
@N QUIC Frames
                      -------->
                                            KeyUpdate @N
                      <--------
  -- Initiating Peer can initiate another KeyUpdate --
 @N Acknowledgment
                      -------->
  -- Responding Peer can initiate another KeyUpdate --
            
                    図4 Key Update
```

図3, 図4で見られるように、異なる2つ以上のキーイングマテリアルがピアによって受信される状況はありません。

サーバはクライアントからのFinishedメッセージが受信されるまで鍵更新を開始できません。一方で、更新された鍵で保護されたパケットはハンドシェイクメッセーの再送にたいして混乱する可能性があります(*)。クライアントはFinished メッセージが受信されたことを示す確認嘔吐を受信するまで鍵更新を開始できません。

注意: このモデルのハンドシェイク中の鍵は、KeyUpdateの中のFinishedメッセージを持って、サーバから開始される鍵交換で変更されます。

## 4.2. QUIC Key Expansion
以下の表はQUICの鍵と、それが生成された時のTLS secretの元を示します。

| Key | TLS Secret | Phase |
|-----------:|------------:|------------:|
|0-RTT|early_traffic_secret|“QUIC 0-RTT key expansion”|
|1-RTT|traffic_secret_N|“QUIC 1-RTT key expansion”|

0-RTT鍵は、TLSハンドシェイクの完了より前にコネクションを再開するのに使用される鍵です。0-RTT鍵を使用して送られるデータは再送される可能性もあるため、その使用には制限があります。セクション5.2を参照ください。0-RTT 鍵はClientHelloを送信するか受信した後に使用されます。

1-RTT 鍵は、TLSハンドシェイクが完了した後に使用されます。潜在的に復数の1-RTT鍵セットがあります。新しい1-RTT鍵はTLS KeyUpdateメッセージの送信によって生成されます。1-RTT鍵はFinished か KeyUpdateメッセージの送信後に使用されます。

鍵拡大は、[I-D.ietf-tls-tls13] のセクション7.3で定義される鍵拡大鍵の拡張と同じプロセスを使用します。例えば、TLS Finishedメッセージを送信た後すぐにデータを送るのに使用するClient Write Keyは

```
   label = "QUIC 1-RTT key expansion, client write key"
   client_write = HKDF-Expand-Label(traffic_secret_0, label,
                                    "", key_length)
```
2オクテット長のフィールド、 文字列 “TLS 1.3, QUIC 1-RTT key expansion, client write key”、0オクテットを含むHKDFへ、ラベルは入力となります。

QUICレコード保護は最初にキーイングマテリアルなしで開始されます。TLSステートマシーンが該当するsecretを生成した時に、新しい鍵はTLSコネクションから生成されQUICレコード保護に使用されます。

Authentication Encryption with Associated Data (AEAD) [RFC5116]関数は、TLSコネクションにおいてネゴシエーションされたものと同じものが使用されます。たとえば、もしTLSがTLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256を使用した場合、AEAD_AES_128_GCMが使用されます。

## 4.3. QUIC AEAD application
通常のQUICパケットはAEAD[RFC5116]で保護されます。バージョンネゴシエーションとpublic reset パケットは保護されません。
一度TLSが鍵を提供すると、TLSメッセージの後は、QUICパケットの中身ははすぐにTLSで選択されたAEADで保護されます。

AEADの鍵Kは、セクション4.2で定義されていように得られたClient Write Key か the Server Write Keyです。
AEADのノンスNはシーケンスナンバーとClient Write IV か Server Write IVをあわせて生成されます。
ネットワークバイトオーダーの48bitに再構成されたQUICのシーケンスナンバー(セクション4.4参照)はAEADのN_MAXparame-taまで0で左側パディングしたものです([RFC5116]のセクション4参照)。パディングされたシーケンスナンバーとIVの排他的論理和がAEADのnonceを構成します。
AEADの関連するデータAは空のシーケンスナンバーです。
AEADの入力となる平文Pは、 [I-D.tsvwg-quic-protocol]に定義されている通り、QUICフレームの内容とそれに続くパケットナンバーです。
AEADの出力となる暗号文は、Pの代わりに送信されます。
TLSが鍵を提供する前は、レコード保護は行なわれず、平文Pが変更なく送信されます。

Note: QUICは平文のパケットにハッシュを本にしたチェックサムを加えたNULL 暗号化を定義しています。これをココに追加するが、より複雑になります。

## 4.4. Sequence Number Reconstruction
各ピアは、再送信を含む送信したパケットごとにインクリメントされる 48bitのシーケンスナンバーを保持します。すべてのパケットは、シーケンスナンバーの最下位の8-, 16-, 32-, もしくは48bitはQUICシーケンスナンバーにエンコードされます。

受信者は同じ値を保持します、受信したパケットを元に値を回復されます(*)。これは受信したパケットのシーケンスナンバーに基づきます。単純な方式は、もっとも最近受信したパケットのシーケンスナンバーをインクリメントすることで、受信したシーケンスナンバーを予想します(*)(*)

より洗練されたアルゴリズムは最も最近受信したパケットのシーケンスの後ろから確認することで探索領域をほぼ二倍にします。もしパケットが受信されていれば、その時はパケットは最も最近に受信したシーケンスナンバーより大きな値になります。もしそのようなパケットが見つけられなかった場合は、単純な方式で、ナンバーは次のシーケンスナンバーを中心に小さなウィンドウであると仮定されます。(**)

Note: QUICは1つの隣接したシーケンスナンバー領域を持ちます。比較して、TLSはレコード保護書きが変わるたびにシーケンスナンバーをリスタートさせます。TLSにおけるシーケンスナンバーのリスタートは
現在のトラフィック鍵が鍵更新後に古い鍵で追加のパケットを送信することで攻撃者によってデータ削除(新しいパケットが破棄される)を不可能なことを確実にします。
QUICは信頼性のあるトランスポートを仮定していませんし、経路上で攻撃者がパケットを落とすということを扱う必要があります。TLSはstream 1上でホストされるコネクションのレコードを保護するために異なるシーケンスナンバーを保持します。このシーケンスナンバーはTLSプロトコルのルールに従ってリセットされます。

# 5. Pre-handshake QUIC Messages
実装はTLSハンドシェイクが完了する前にstream 1以外のストリームでデータを交換してはいけません(MUST NOT)。しかし、QUICはロス検出と回復管理のために幾つかのフレームタイプの使用を要求します。加えて、輻輳管理のために認証されていないメッセージの交換時に得られたデータは有用かもしれません。

このセクションは両ピアが送信するTLSハンドシェイクメッセージとそれらのメッセージを転送するパケットへの確認応答のみに適応されます。多くの場合、クライアントが送信するメッセージは小さくサーバの返信によって暗黙的に確認されるので、サーバが確認応答を提供する必要性は最小です。

認証されてないパケットを受信した結果としてピアが取る行動は制限される必要があります。特に、これらのパケットによって確立された状態は一度レコードが保護が開始されると維持できません(*)

ハンドシェイクが完了する前の認証されてないパケットを扱うことができるアプローチは幾つかあります。
- それらを破棄し無視する
- それらを使用するが、一度ハンドシェイクが完了し確立された状態をリセットする
- それらを使い、その後認証する。もしそれらが認証できなければ、ハンドシェイクは失敗となる。
- それらを保存し、正しくそれらが検証された時に使用する
- それらをfatal errorとして扱う

異なる戦略は異なるデータ・タイプに適切です(*)。この文書では、全ての戦略はメッセージょのタイプに依存することを提案します(*)

- とランポートパラメータとオプションはTLSハンドシェイクの一部として使用可能とし認証されます。
- ハンドシェイクを完了させるのに必要な少数をのぞいて、ほとんどの保護されてないメッセージはfatal errorとして扱われます。
- 保護されたパケットは破棄するか保存して後で使用することが出来ます。

## 5.1. Unprotected Frames Prior to Handshake Completion
このセクションはTLSハンドシェイクが完了する前に送受信されるメッセージの扱いについて記述します。

保護されていないメッセージの送受信は危険です。明示的に許可されてないかぎり、保護されてないどのようなする居のメッセージの受信はfatal errorとして扱う必要があります(MUST)。

### 5.1.1. STREAM Frames
stream 1のSTREAM フレームは許可されます。これらはTLSハンドシェイクメッセージを転送します。それ以外のストリームで保護されていないSTREAM フレームの受信は、fatal errorとして扱わなければならい(MUST)。


### 5.1.2. ACK Frames
ACK フレームはハンドシェイクが完了する以前でも許可されます。
攻撃者がこれらのパケットを挿入できるため、ACK フレームから得られる情報に完全に依存することはできません。
ACK フレームによって得られるタイミングとパケット再送の情報はプロトコルの昨日として重要ですが、これらのフレームは偽装されたり変更される可能性があります。

エンドポイントは0-RTTや1-RTT鍵で保護されたデータフレームの確認応答に保護されていないACK フレームを使用してはいけません(MUST NOT)。もしACK が保護されたデータの確認を主張する場合、エンドポイントは保護されていないACK フレームを無視しなければいけません(MUST)。保護されたデータを読めるエンドポイントは常に保護されたデータを送信が許可されているため、このような確認応答はサービス拒否攻撃としての役割を果たすことことしかできません(*)。

エンドポイントはハンドシェク中か1-RTTで保護されたACKから不十分な情報しか持たない時は、保護されてないか0-RTTで保護されたACK フレームでのみデータを使用すべきです(SHOULD)。保護されたメッセージから十分な情報が得られたら、信頼性の低い情報源から得られた情報は破棄できます。

### 5.1.3. WINDOW_UPDATE Frames
WINDOW_UPDATEは保護せず送ってはいけません(MUST NOT)。

stream 1上でデータが交換されますが、最初のフロー制御windowはTLSハンドシェイクを完了するのに十分な大きさです。これによりTLSはどシェイクの最大サイズの大きさは再現され、サーバやクライアントを異常に大きな証明書チェーンの使用から防ぐことが出来ます。

Stream 1はコネクションレベルのフロー制御ウィンドウの対象になりません。

### 5.1.4. Denial of Service with Unprotected Packets
エンドポイントにとって、明確に認証されていない保護されていないパケットにはdenial of serviceのリスクがあります。攻撃者は保護されていないパケットを挿入することが出来き、これは一致するシーケンスナンバーの保護されたパケットの破棄を引き起こします。怪しいパケットは本物のパケットを覆い尽くし、正しいパケットが上長として無視されような状況を引き起こします。

TLSハンドシェイクが完了すると、両方のピアは保護されていないパケットを無視しなければいけません(MUST)。サーバがClientのFinished を受信し、クライアントが自身が送信したFinishedメッセージ対する確認応答を受信したとき、 ハンドシェイクが完了します。その時点以降、保護されてないメッセージは安全に破棄されます。クライアントはFinished メッセージをサーバに再送するかもしれない点に注意してください。サーバはこれらのメッセージを拒否できません。

TLSハンドシェイクのパケットと確認応答のみが平文で送られるため、攻撃者は実装に損失したり埋め尽くされたパケットの再送に依存するように強制できます。このように、エンドポイントにdeny serviceしようとする攻撃者は被害者が保護されていないパケットを受け続けるのを確実にするために、エンドポイントのパケットを落としたりおおいつくしたりします。shadow packetsパケットの能力は攻撃者は経路上にいる必要が無いということを意味します。

ISSUE: もしQUICがランダムなシーケンスナンバーから開始すれば、これは問題にはならないでしょう。私たちはこの問題を修正し、経路上の攻撃者へのdenial of serviceへの漏出を軽減します(*)問題の可能性は初期値の認証のみです。これはピアが最初のメッセージを欠かすことが出来なくなるということです。

エンドポイントのメッセージを拒否することに加えて、攻撃者は受信時に状態変更を発生させないパケットを生成します。これらのリスクはセクション 7.2 参照。

使用できるデータを含まないTLSパケットの受信を避けるため。TLS実装はからのTLSハンドシェイクレコードおよびTLSステートマシーンが許可しないレコードを拒否しなければなりません(MUST)。適切な時間のend_of_early_data 以外のTLSアプリケーションやデータは、それらがTLSハンドシェイクが完了する前に受信された場合はfatal errorとして扱わなければなりません(*)

## 5.2. Use of 0-RTT Keys
もし 0-RTT鍵が有効であれば、再送保護の欠如は、これらの使用上の制限は、プロトコル上のリプレイ攻撃を回避するために必要です(*)。

クライアントは 冪等なデータを保護するためだけに 0-RTT鍵を使用しなければなりません(MUST)。TLSハンドシェイクが完了する前に送られるデータに、クライアントはさらなる制限を適応しようとしても良いです。クライアントはそうでなければ、0-RTT鍵を1-RTT鍵と同等として扱うことが出来ます。(*)

サーバが0-RTTデータを送ることで0-RTTデータが受け取られることを示唆されたクライアントは、サーバからのハンドシェイクメッセージすべてを受信するまで0-RTTデータを送信できます。もし0-RTTデータが拒否された事が示唆された場合、クライアントは0-RTTデータの送信をやめるべきです(SHOULD)。early_data 拡張のないServerHello に加えて、KEY_PHASE bitが0の保護されていないハンドシェイクメッセージは0-RTTデータは拒否されたことを示します。

ハンドシェイクメッセージを全てを受信した場合あとのみ、クライアントは end_of_early_dataアラートを送信すべきです。言い換えれば(*)、クライアントは0-RTT鍵を1-RTT鍵が有効になるまで0-RTT鍵を使用することが推奨されます。これはコネクションの失速を避け、クライアントが断続的にデータを送信できるようになります(*)

サーバはTLSハンドシェイクメッセージ以外を保護するために0-RTT鍵を使用してはいけません(MUST NOT)。したがってサーバはどのようなパケットの送信が許可されているか判断するときに、0-RTTで保護されたパケットを保護されていないパケットとして扱います。もし0-RTT鍵が受け入れられると判断できた場合は、サーバは0-RTT鍵でハンドシェイクメッセージを保護します。サーバはServerHello メッセージのearly_data 拡張を含めなければいけません(MUST)。

この制限は0-RTT鍵で保護されたフレームを使用したリクエストへの応答からサーバを防ぐためです。これは、サーバからの全てのアプリケーションはforward secrecyを保つ鍵で常に保護されていることを保証します。しかしながら、その結果、クライアントが全てのサーバのハンドシェイクメッセージを受信するまで、サーバのレスポンスは復号できないので、クライント側でhead-of-line blockingを起こします。

## 5.3. Protected Frames Prior to Handshake Completion
順番の入れ替えやロスのため、最後のハンドシェイクメッセージを受信する前に、エンドポイントは保護されたパケットを受信するかもしれません。これらが正常に復号できた場合、そのようなパケットは保存され、ハンドシェイクが完了したら、使用します。以下の事に明示的な許可が無ければ、暗号化されたパケットはTLSハンドシェイクが完了するまで、特に、Finishedメッセージの受信とピアの認証がすむまで、使用してはいけません(MUST NOT)。

もしパケットがハンドシェイク前に処理されると、攻撃者は実装がそのようなパケットを使用することを様々な攻撃に利用するでしょう。一度鍵合意が完了すると、TLSハンドシェイクメッセージがハンドシェイク中はレコード保護によってカバーされます(*)。これは、保護されたメッセージは、これがTLSハンドシェイクメッセージなのか判断するために複合する必要があるということです。同様にACK とWINDOW_UPDATEフレームもTLSハンドシェイクが完了する必要があるかもしれません(*)

ACLフレームのタイムスタンプはfatal errorを引き起こすのではなく無視しなければいけません(*)。保護されたフレームのタイムスタンプはTLSハンドシェイクが完了するまで保存して置いても良いです(MAY)。エンドポイントはそれぞれのストリーム上で受信した最後のWINDOW_UPDATEフレームを保存しておき、TLS ハンドシェイクが完了したらその値を適応しても良いです(MAY)。

これに失敗すると影響を受けるストリームで一時的な原則となります。

# 6. QUIC-Specific Additions to the TLS Handshake
QUICのバージョンネゴシエーションの仕組みはハンドシェイクが完了するまでに使用するQUICのバージョンのネゴシエーションに使用されます。しかし、パケット認証れないので能動的攻撃者がバージョンのダウングレードを強制できます。

## 6.1. Protocol and Version Negotiation
QUICのバージョンのダウングレードは攻撃者に強制されないことを保証するために、バージョン情報はTLSハンドシェイクにコピーされ、QUICネゴシエーションの完全性を提供します。

ハンドシェイク中のバージョンのダウングレードを防ぐことはできませんが、そのようなダウングレードはハンドシェイクの失敗になることを意味します。

QUICトランスポートを使用するプロトコルは、Application Layer Protocol Negotiation (ALPN) [RFC7301]を使用しなければなりません(MUST)。ALPN識別子に動作するQUICのバージョンが指定されなければなりません。(*)サポートするどのQUICバージョンが現在選択されたにかかわらず、ClientHelloを構成する際、クライアントはサポートするALPN識別子リストを付加する必要があります(*)。

サーバはクライアントが選択したQUICバージョンを使うのではなく、ClientHelloの情報に基づいてアプリケーションプロトコルを選択すべきです(SHOULD)。
選択されたプロトコルが使用されているQUICバージョンをサポートしていない場合、可能でああればサーバはQUICバージョンのネゴシエーションパケットを送信しなければなりません(MUST)、そうでなければコネクションを失敗させます。

## 6.2. QUIC Extension
QUICはTLSの使用に伴って拡張を定義しています。この拡張はトランスポートに関わるパラメータを定義します。TLSハンドシェイクでこれらを含めると、QUICフレームでパラメータを送信するより早く1ラウンドトリップでクライアントはサーバにパラメータをセットできます。
この文書では、その拡張を定義していません。

## 6.3. Source Address Validation
QUIC実装は送信元アドレストークンを評します。
与えられた送信元アドレスを最初に使用するとき、これはサーバはクライアントに提供される不明瞭なデータです。
クライアントは後続のメッセージとして帰りの経路確認としてこのトークンを返します(*)。これは、クライアントは主張する送信元アドレスでパケットを受信できる証明として、このトークンを送り返します。反射攻撃のパケットに使用されることからサーバを守ります。(セクション7.1を参照)

送信元アドレストークンは不明瞭なデータで、サーバによってのみ使用されます。そのため、0-RTTハンドシェイクのためにTLS1.3の事前鍵識別子のなかで使用出来ます。0-RTTを使用するサーバは、全てのハンドシェイクの後に、能動的観測者が経路上で接続することを防ぐために新しい事前共有鍵の識別子を提供することを通知します。クライアントは新しく開始するコネクション全てで新しい事前共有鍵の識別子を使用しなければなりません(MUST)。
もし事前共有鍵の識別子が有効でなければ、セッションの再開は不可能です。

負荷がかかってるサーバでは、HelloRetryRequestのcookie拡張に送信元アドレストークンを含めるかもしれません。

(注意：現在のTLS1.3ではHelloRetryRequestにCookieを含めることは出来ません。)

## 6.4. Priming 0-RTT
QUICはTLSを変更なく使用すします。そのため、TCP over TLSのコネクションで得られた0-RTTに使用できる事前共有鍵をQUICで使用することが出来ます。同様に、QUICは、TCPで0-RTTが使用できる事前共有鍵を提供できます。

0-RTTの使用には全ての制限を適応します、そして証明書は双方の接続で有効である必要があります(MUST)。これは、異なるプロトコルスタックと異なるポートがしようされます。

例えばTCPとTLS上のHTTP/1.1とHTTP/2、QUIC over UDPです。

送信元アドレス検証は異なるプロトコルスタック間で完全には移植されません。もし送信元IPアドレスが同じままであっても、ポート番号は異なるかもしれません。攻撃可能なホストは現象しますが、パケットの反射攻撃はこの状況でもまだ可能です。
サーバはこのようなコネクションに対して送信元アドレス検証を行わないことを選ぶかもしれませんが、それはクライアントに送信されるデータの送信元検証されないデータの量を増加させます。

# 7. Security Considerations
There are likely to be some real clangers here eventually, but the current set of issues is well captured in the relevant sections of the main text.
Never assume that because it isn’t in the security considerations section it doesn’t affect security. Most of this document does.

## 7.1. Packet Reflection Attack Mitigation
小さなClientHelloから、結果としてサーバからの大きなハンドシェイクメッセージとなるため、攻撃者によってトラフィックの増幅する反射攻撃として使用されます。

証明書のキャッシュ[I-D.ietf-tls-cached-info] は、サーバのハンドシェイクメッセージのサイズを大きく減らすことが出来ます。

クライアントはClientHelloを最低でも1024オクテット(TODO:この値を調整する)パディングするべきです(SHOULD)。サーバは
それが送信したデータが受信した復数の小さなデータであれば、サーバはより反射攻撃のパケットを発生しにくくなります(*)。
もし送信するハンドシェイクメッセージのサイズがClientHelloのサイズより大きくなりそうであれば、サーバはHelloRetryRequest を使用すべきです(SHOULD)。

## 7.2. Peer Denial of Service
QUICとTLSとHTTP/2はいくつかのコンテキストで正当な用途を持つメッセージを含みます(*)が、コネクションの状態に観測可能な影響を与えること無く、ピアに処理にリソースを費やすように不正利用されるかもしれません。

もし処理が、帯域幅や状態に観測可能な影響と比較して不釣り合いに大きければ、悪意あるピアはキャパシティを枯渇させること事ができます(*)


Fin bitがセットされていなQUICは空のSTREAM フレームの送信を禁止しています。これは廃棄されるだけのSTREAM フレームを送信することを防ぐためです(*)。

TLSレコードは常にハンドシェイクメッセージかアラートの最後の1オクテットを保持しておくべきです。パディングのみを含むレコードはハンドシェイク中のみ許可されていますが、大きすぎる数が不必要な処理をさせるのに使用されるかもしません。TLSハンドシェイクが完了すると、エンドポイントはQUICレコード長を隠さないのであれば、TLSアプリケーションデータレコードを送信すべきではありません(SHOULD NOT)

QUICのパケット保護はパディングのために割り当てられる領域を含んでいません(*)。パディングされたTLSアプリケーションデータレコードは、QUICフレームの長さを隠すために使用できます。

冗長なパケットを使用することに正当性がありますが、実装は冗長なパケットをトラックし、非生産的な箇条なパケット量を攻撃の判断として扱います。


# 8. IANA Considerations
This document has no IANA actions. Yet.

# 9.  References
## 9.1.  Normative References

   [I-D.hamilton-quic-transport-protocol]
              Hamilton, R., Iyengar, J., Swett, I., and A. Wilk, "QUIC:
              A UDP-Based Multiplexed and Secure Transport", draft-
              hamilton-quic-transport-protocol-00 (work in progress),
              July 2016.

   [I-D.ietf-tls-tls13]
              Rescorla, E., "The Transport Layer Security (TLS) Protocol
              Version 1.3", draft-ietf-tls-tls13-17 (work in progress),
              October 2016.

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119,
              DOI 10.17487/RFC2119, March 1997,
              <http://www.rfc-editor.org/info/rfc2119>.

   [RFC5116]  McGrew, D., "An Interface and Algorithms for Authenticated
              Encryption", RFC 5116, DOI 10.17487/RFC5116, January 2008,
              <http://www.rfc-editor.org/info/rfc5116>.

   [RFC7301]  Friedl, S., Popov, A., Langley, A., and E. Stephan,
              "Transport Layer Security (TLS) Application-Layer Protocol
              Negotiation Extension", RFC 7301, DOI 10.17487/RFC7301,
              July 2014, <http://www.rfc-editor.org/info/rfc7301>.

## 9.2.  Informative References

   [RFC0793]  Postel, J., "Transmission Control Protocol", STD 7,
              RFC 793, DOI 10.17487/RFC0793, September 1981,
              <http://www.rfc-editor.org/info/rfc793>.

   [RFC7230]  Fielding, R., Ed. and J. Reschke, Ed., "Hypertext Transfer
              Protocol (HTTP/1.1): Message Syntax and Routing",
              RFC 7230, DOI 10.17487/RFC7230, June 2014,
              <http://www.rfc-editor.org/info/rfc7230>.

   [RFC7258]  Farrell, S. and H. Tschofenig, "Pervasive Monitoring Is an
              Attack", BCP 188, RFC 7258, DOI 10.17487/RFC7258, May
              2014, <http://www.rfc-editor.org/info/rfc7258>.

   [RFC7540]  Belshe, M., Peon, R., and M. Thomson, Ed., "Hypertext
              Transfer Protocol Version 2 (HTTP/2)", RFC 7540,
              DOI 10.17487/RFC7540, May 2015,
              <http://www.rfc-editor.org/info/rfc7540>.

   [RFC7685]  Langley, A., "A Transport Layer Security (TLS) ClientHello
              Padding Extension", RFC 7685, DOI 10.17487/RFC7685,
              October 2015, <http://www.rfc-editor.org/info/rfc7685>.

   [RFC7924]  Santesson, S. and H. Tschofenig, "Transport Layer Security
              (TLS) Cached Information Extension", RFC 7924,
              DOI 10.17487/RFC7924, July 2016,
              <http://www.rfc-editor.org/info/rfc7924>.

# Appendix A.  Acknowledgments

   Christian Huitema's knowledge of QUIC is far better than my own.
   This would be even more inaccurate and useless if not for his
   assistance.  This document has variously benefited from a long series
   of discussions with Jana Iyengar, Adam Langley, Roberto Peon, Eric
   Rescorla, Ian Swett, and likely many others who are merely forgotten
   by a faulty meat computer.

Authors' Addresses

   Martin Thomson
   Mozilla

   Email: martin.thomson@gmail.com


   Ryan Hamilton
   Google

   Email: rch@google.com
