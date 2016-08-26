- [Abstract](#abstract)
- [1.  Introduction](#1--introduction)
- [2.  Design of the QUIC Transmission Machinery](#2--design-of-the-quic-transmission-machinery)
  - [2.1.  Relevant Differences Between QUIC and TCP](#21--relevant-differences-between-quic-and-tcp)
    - [2.1.1.  Monotonically Increasing Sequence Numbers](#211--monotonically-increasing-sequence-numbers)
    - [2.1.2.  No Reneging](#212--no-reneging)
    - [2.1.3.  More ACK Ranges](#213--more-ack-ranges)
    - [2.1.4.  Explicit Correction For Delayed Acks](#214--explicit-correction-for-delayed-acks)
- [3.  An Overview of QUIC Loss Recovery](#3--an-overview-of-quic-loss-recovery)
  - [3.1.  On Sending a Packet](#31--on-sending-a-packet)
  - [3.2.  On Receiving an ACK](#32--on-receiving-an-ack)
  - [3.3.  On Timer Expiration](#33--on-timer-expiration)
- [4.  Congestion Control](#4--congestion-control)
- [5.  TCP mechanisms in QUIC](#5--tcp-mechanisms-in-quic)
  - [5.1.  RFC 6298 (RTO computation)](#51--rfc-6298-rto-computation)
  - [5.2.  FACK Loss Recovery (paper)](#52--fack-loss-recovery-paper)
  - [5.3. RFC 3782, RFC 6582 (NewReno Fast Recovery)](#53-rfc-3782-rfc-6582-newreno-fast-recovery)
  - [5.4.  TLP (draft)](#54--tlp-draft)
  - [5.5. RFC 5827 (Early Retransmit) with Delay Timer](#55-rfc-5827-early-retransmit-with-delay-timer)
  - [5.6. RFC 5827 (F-RTO)](#56-rfc-5827-f-rto)
  - [5.7.  RFC 6937 (Proportional Rate Reduction)](#57--rfc-6937-proportional-rate-reduction)
  - [5.8.  TCP Cubic (draft) with optional RFC 5681 (Reno)](#58--tcp-cubic-draft-with-optional-rfc-5681-reno)
  - [5.9.  Hybrid Slow Start (paper)](#59--hybrid-slow-start-paper)
- [6.  References](#6--references)
  - [6.1.  Normative References](#61--normative-references)
  - [6.2.  Informative References](#62--informative-references)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

https://tools.ietf.org/html/draft-iyengar-quic-loss-recovery-00

# Abstract
QUICはUDP上に多重化されたセキュアな新しいトランスポートプロトコルであり、現代の汎用的なトランスポートとして魅力的となる仕組みが実装されています。QUICはRFCや様々なInternet-draftsに記述されていたり、LinuxのTCP実装で流行したTCPのロスリカバリの仕組みを実装しています。この文書はQUICの輻輳制御とロスリカバリについて記述しており、RFC、Internet-draft、学術論文、TCP実装と同様のTCPの性質が適応されます。

# 1.  Introduction
QUICは新しいUDP上の多重化されたセキュアなトランスポートです。QUICは数十年のトランスポートとセキュリティの経験に基づいています。そして、現代の汎用的なトランスポートとして魅力的となる仕組みが実装されています。QUICプロトコルは[[draft-hamilton-quic-transport-protocol]](https://tools.ietf.org/html/draft-hamilton-quic-transport-protocol-00)で記述されています。

QUICはRFCや様々なInternet-draftsに記述されていたり、LinuxのTCP実装で流行したTCPのロスリカバリの仕組みを実装しています。この文書はQUICの輻輳制御とロスリカバリについて記述しており、RFC、Internet-draft、学術論文、TCP実装と同様のTCPPの性質が適応されます。

この文書ではまずQUICの輻輳制御とロスリカバリの仕組みを説明するのに必要な転送機械の部分の記述をします。そしてQUICのデフォルトの輻輳制御とデフォルトのロスリカバリについて記述し、QUICで実装されるロスリカバリのTCPの様々な仕組みの説明が続きます。

# 2.  Design of the QUIC Transmission Machinery
QUICの中での全ての転送はパケットレベルのヘッダを伴って送信されます。これは、パケットのシーケンス番号(以下ではパケット番号と呼ばれます)を含みます。これらのパケット番号はコネクションの生存期間の間繰り返されませんし、単純増加し、重複検出を些細なものにします。この基本的な設計上の決定はTCPのロス検出機構のQUICの解釈から、送信と再送の間のあいまい性をなくし、大きな複雑性を除去します。

すべてのパケットはいくつかのフレームを含むことができます；ロス検出と輻輳制御機構に重要なフレームの概要を以下に示します。

- STREAMフレームはアプリケーションデータを含みます。暗号ハンドシェイクデータもSTREAMデータとして送信され、下にQUICの信頼性の機構を使用します。
- ACKフレームは確認応答情報を含みます。QUICはSACKベースの方式を使用し、それはACKフレームでACKされた最大のパケット番号を明示的に報告します。そして、ACKレンジとして報告される最大のACKされたものよりシーケンス番号を持つパケットは小さいです。ACKフレームは新しくACKされたそれぞれのパケットに対しての受信したタイムスタンプを含みます。
- ACKブロックを送信者によってまだ受信していないものに限定するために、送信者は定期的に 、受信者に指定したシーケンスナンバー以下のパケットへのackを止めるようにシグナルするSTOP_WAITINGフレームを送信し、受信者側で"一番小さいackされてない"パケット番号を引き上げます。そのためACKフレームの送信者は、最小のackをしてないものと、報告されたパケット番号の間のACKブロックのみをレポートします。送信者に、STOP_WAITINGフレームの最小の値としてACKで受信される最近受信した最大のackされたパケットを送信することを推奨します。

## 2.1.  Relevant Differences Between QUIC and TCP
QUICとTCPの間には顕著な違いがあり、それは2つのプロトコルで採用されているロスリカバリの違いが論理的に重要です。簡単にこれらの違いを以下に記述します。

### 2.1.1.  Monotonically Increasing Sequence Numbers
TCPは送信者側の送信シーケンス番号と受信者側の配信シーケンス番号を合成します。これは同じシーケンス番号を持つ同じデータの再送信となり、その結果 "再送の曖昧性"の問題が発生します。QUICではその2つを区別します。QUICはパケット送信番号(パケット番号と呼ばれます)を送信に使用し、受信するアプリケーションに配送されるいかなるデータも、配送順番を決定できるエンコードされたストリームオフセットがSTREAMフレームの中にもち、1つ以上のストリームで送信されます。

QUIKパケットのシーケンス番号は単調増加し、直接転送順を符号化します。高いQUICシーケンス番号はパケットがより後に送信されたことを意味し、低いシーケンス番号は、パケットは先に送信されたことを意味します。

この設計のポイントはQUICのロス検出の仕組みをかなり単純化します。ほとんどのTCPの仕組みは暗黙的にTCPシーケンス番号に基づいて送信順序を推測しようとします。これは非自明な作業であり、TCPのタイムスタンプが利用できない時は特にです。


ACKを受信した時どのパケットが確認されたのかの曖昧性を除去するために、QUICは再送が必要なとき、ロスしたパケットを新しいシーケンス番号で送信します。その結果として、シーケンス番号のみに基づいてより正確なRTT測定が可能であり、スプリアス再送は自明に検出され、Fast Restansmitなどの仕組みが全般的に適応できます。

### 2.1.2.  No Reneging
QUICのACKはTCPのSACKと同等の情報を持ち、QUICはいかなるACKされたパケットも取りやめる事はできず、大幅に両側の実装を簡素化しメモリへの圧迫を減らします。

### 2.1.3.  More ACK Ranges
TCPの3 SACKレンジとは対照的に、QUICは255までのACKレンジをサポートします。高いロス環境においても、素早くリカバリします。

### 2.1.4.  Explicit Correction For Delayed Acks
QUICのACKは明示的に受信者がパケットを受信してから対応するACKを送信するまでに被った遅延を符号化します。経路上のRTTを見積もる時に、ACKの受信者が受信者の遅延に調整できるようになります、具体的にはACKまでの遅延タイマーです。この仕組は受信者が、OSカーネルがパケットを受信してからの遅延を見積もり報告出来るようにします。これは、ユーザスペースのQUICが受信したパケットを処理する前に、コンテキストスイッチのレイテンシなどの遅延を被る受信者にとって有用です。


# 3.  An Overview of QUIC Loss Recovery
QUICのパケット送信、ACKの受信、イベントの期限切れタイマーの挙動について簡単に記述します。

## 3.1.  On Sending a Packet
モードに基づいた再送タイマーがセットされます
- もしハンドシェイクが完了して無ければ、ハンドシェイクタイマーを開始します
  - Exponential Backoffで 1.5 SRTTがセットされます
- もしまだACKされていない未処理のパケットがあれば、ロスタイマーがセットされます。
  - ロス検出の実装に依存する。Early Retransmitの場合、デフォルトは 0.25 RTT
- 2つより少ないTLPが送信されていれば、計算しTLPタイマーをリスタートする
  - 複数のパケットが送信中であれば、タイマーは(10msと2*SRTT)の大きい方がセットされます
  - もし1つのパケットのみが送信中であれば、タイマーは(1.5*SRTT + delayed ack timer, 2*SRTT)の大きい方がセットされます
- もし2つの TLPがすでに送信されていれば、RTOタイマーがセットされます
  - 最初のRTOの後に、タイマーはExponential Backoffで(200ms, SRTT+4*RTTVAR)の大きい方がセットされます

## 3.2.  On Receiving an ACK
ACKが受信された時、以下のステップが実行されます

- ackの検証、順番通りでないackの無視も行う
- RTT測定結果の更新
- 送信者はこのACKフレームでACKされ観測された最大のものより小さいackされてないパケットをACKEDとしてマークします
- 観測された最大のものより小さいパケット番号を持つまだackされてないパケットは、FACKに基づいてインクリメントされるmissing_reportsを持ちます(largest_observed - missing packet number)
- デフォルトで閾値に3がセットされます
- missing_reports > threshold なパケットは再送のためにマークされます。このロジックはFast RetransmissionとFACKに基づいた再送を実装します
- もしパケットが未処理で、観測された最大のパケットが送信された最大のパケットであれば、再送タイマーは 0.25SRTTにセットされ、タイマーをもつEarly Retransmitを実装します。
- 未処理のパケットがなければ、タイマーを停止する

## 3.3.  On Timer Expiration
QUICは1つのロスリカバリタイマを使用し、それはセットされた時、幾つかの状態のうちの一つになります。もしタイマーが期限きれしたとき、状態によって実効できる行動を決定します(TODO:いつタイマーがセットされるか記述する)

- Handshake状態:
  - 未処理のパケットを再送する
- ロスタイマー状態:
  - まだACKされていない未処理のパケットを失う
  - 輻輳制御機にロスを報告する
  - 輻輳制御機が許可するだけの再送する
- TLP状態:
  - 再送可能な、未処理の最小のパケットを再送する
  - ACKが届くまで、いかなるパケットも失ったとしてマークしません
  - TLPとRTOのタイマーをリスタートします
- RTO状態:
  - 再送可能な、最小の2つの未処理のパケットを再送する。
  - ackが受信されRTOがスプリアスでないことを確認するまで、輻輳ウィンドウを下落させない(例: 1パケットをセットする)
  - 次のRTOのためにタイマーをリセットする(exponential backoffする)

# 4.  Congestion Control
(QUICのNewRenoスタイルの輻輳制御を記述する)

# 5.  TCP mechanisms in QUIC
QUICは様々なRFC、Internet Draft、その他のよく知られたロスリカバリの仕組みを実装しています。実装の詳細は、TCP実装によって異なります。

## 5.1.  RFC 6298 (RTO computation)
QUICは標準式によってSRTTとRTTVARを計算します。もし遅延したackの補正が測定されたRTTよりも小さい場合(そうでなければ負のRTTになります)、RTT標本が取得され、ackは新しく大きい最大の観測されたパケット番号を含みます。最小のRTTは観測されたRTTにのみ基づきますが、SRTTは差分を補正する遅延したACKを使用します。

以上で記述されたとおり、QUICは標準的なタイムアウトとCWNDが減少するRTOを実装しています。しかしながら、QUICは再送の曖昧性を持たないので、QUICは最新のよりも最も古いパケットを再送します。QUICはRFCで指定されている1秒ではなく、一般に受け入れられている最小の200 msのRTOを使用します。

## 5.2.  FACK Loss Recovery (paper)
QUICは早いロスリカバリのためにFACKの論文で記述された(そしてLinux Kernelで実装されている)アルゴリズムを実装します。QUICはFACKの並び替えの閾値を測定するのに、パケットシーケンス番号を使用します。現在のQUICは、多くのTCP実装が行う適応的閾値を実装していません。

## 5.3. RFC 3782, RFC 6582 (NewReno Fast Recovery)
QUICはNewReno RFCを踏まえて、輻輳ウィンドウごとに一度そのCWNDを減少させます。 損失が明らかになった時点で未解決の最大のパケットを追跡し、そのパケット番号の前に発生した損失は同じロスイベントの一部とみなされます。
幾つかのTCP実装にはシーケンス番号に基づいてコレを行うことは注目に値します。したがって、一つのパケットの複数回の損失は一回のロスイベントとして考えられます。

## 5.4.  TLP (draft)
RTOがトリガーされる前に、QUICは常に二つのtail loss probesを送信します。ロスが未処理の時はいつでも、QUICはtail loss probeを発動します。これはTCP実装と違います。

## 5.5. RFC 5827 (Early Retransmit) with Delay Timer
QUICはスプリアス再送を最小化するためのタイマーを持つearly retransmitを実装します。最後の未処理のパケットがACKされた後に、タイマーは1/4 SRTTがセットされます。

## 5.6. RFC 5827 (F-RTO)
QUICは後続のACKが受信されRTOがスプリアスではないことを確認するまで、CWND と SSThreshF-RTOを減少させないことでF-RTOを実装します。概念的にはこれは似ていますが、より少ないエッジケースを持つよりクリアな実装になります。

## 5.7.  RFC 6937 (Proportional Rate Reduction)
ロスからリカバリする際に、PRR-SSRB はQUICで実装されます。

## 5.8.  TCP Cubic (draft) with optional RFC 5681 (Reno)
TCP CubicはQUICのデフォルトの輻輳制御アルゴリズムです。Renoは完全に実装されており、コネクションのオプションを通して容易に有効にできます。

## 5.9.  Hybrid Slow Start (paper)
QUICはハイブリッドスロースタートを実装していますが、パケットペーシングを組み合わせた時に謝ってトリガーされることが示されているため、ACK train検出は無効になっています。QUICでは、ハイブリッドスロースタートはデフォルトで有効です。現在の最小遅延増加は4msで、最大は16msです。そしてその範囲内で、コネクションのmin_rttの1/8以上に増加そのラウンドで増加した場合、QUICはスロースタートを終了します。



# 6.  References

## 6.1.  Normative References

   [RFC2119]  Bradner, S., "Key Words for use in RFCs to Indicate
              Requirement Levels", March 1997.

## 6.2.  Informative References

   [draft-hamilton-quic-transport-protocol]
              Hamilton, R., Iyengar, J., Swett, I., and A. Wilk, "QUIC:
              A UDP-Based Multiplexed and Secure Transport", July 2016.

Authors' Addresses

   Janardhan Iyengar
   Google

   Email: jri@google.com


   Ian Swett
   Google

   Email: ianswett@google.com
