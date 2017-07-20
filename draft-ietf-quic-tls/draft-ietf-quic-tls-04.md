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

