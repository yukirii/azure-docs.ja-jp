---
title: Azure Network Watcher の IP フロー検証の概要 | Microsoft Docs
description: このページでは、Network Watcher の IP フロー検証機能の概要を説明します。
services: network-watcher
documentationcenter: na
author: damendo
ms.service: network-watcher
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: infrastructure-services
ms.date: 11/30/2017
ms.author: damendo
ms.openlocfilehash: 69aca5e0901a0da8aa98fe310ac220898bf650b2
ms.sourcegitcommit: 5d6ce6dceaf883dbafeb44517ff3df5cd153f929
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 01/29/2020
ms.locfileid: "76845008"
---
# <a name="introduction-to-ip-flow-verify-in-azure-network-watcher"></a>Azure Network Watcher の IP フロー検証の概要

IP フロー検証は、仮想マシンから送受信されるパケットの許可または拒否の状況を検証します。 この情報は、方向、プロトコル、ローカル IP、リモート IP、ローカル ポート、リモート ポートで構成されます。 パケットがセキュリティ グループにより拒否された場合、そのパケットを拒否した規則の名前が返されます。 管理者は、任意の送信元または送信先の IP を選択でき、IP フロー検証を使用してインターネットまたはオンプレミス環境との接続の問題をすばやく診断できます。

IP フロー検証では、サブネットや仮想マシン NIC など、ネットワーク インターフェイスに適用されるすべてのネットワーク セキュリティ グループ (NSG) に対するルールが調べられます。 次に、そのネットワーク インターフェイスに関する構成済みの設定に基づいてトラフィック フローが検証されます。 IP フロー検証は、仮想マシンとの間のイングレスまたはエグレス トラフィックがネットワーク セキュリティ グループ内の規則によってブロックされているかどうかを確認するのに役立ちます。

Network Watcher のインスタンスは、IP フロー検証を実行する予定のすべてのリージョンに作成する必要があります。 Network Watcher はリージョン別のサービスであり、同じリージョン内のリソースに対してのみ実行できます。 NIC またはサブネットに関連付けられたルートは返されるため、使用されるインスタンスは IP フロー検証の結果には影響しません。

![1][1]

## <a name="next-steps"></a>次のステップ

ポータルを使用して特定の仮想マシンでのパケットの許可または拒否の状況を確認する方法については、次の記事をご覧ください。 [ポータルを使用して VM でトラフィックが許可されているかどうかを IT フロー検証で確認する方法](diagnose-vm-network-traffic-filtering-problem.md)

[1]: ./media/network-watcher-ip-flow-verify-overview/figure1.png

