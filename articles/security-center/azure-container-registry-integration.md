---
title: Azure Security Center と Azure Container Registry
description: Azure Security Center と Azure Container Registry の統合について説明します
services: security-center
documentationcenter: na
author: memildin
manager: rkarlin
ms.service: security-center
ms.devlang: na
ms.topic: conceptual
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 11/19/2019
ms.author: memildin
ms.openlocfilehash: 2d588d2707c267097e25176997e58f9573017582
ms.sourcegitcommit: 5b073caafebaf80dc1774b66483136ac342f7808
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 01/09/2020
ms.locfileid: "75780047"
---
# <a name="azure-container-registry-integration-with-security-center-preview"></a>Azure Container Registry と Security Center の統合 (プレビュー)

Azure Container Registry (ACR) は、Azure デプロイ用のコンテナー イメージを中央のレジストリに格納して管理する、プライベートなマネージド Docker レジストリ サービスです。 これは、オープンソースの Docker Registry 2.0 を基にしています。

Azure Security Center の 標準レベルを使用している場合には、Container Registries のバンドルを追加できます。 このオプションのフィーチャーを使用すると、レジストリ内のイメージの脆弱性をより詳しく把握できます。 サブスクリプション レベルでバンドルを有効または無効にし、サブスクリプション内のすべてのレジストリをカバーします。 このフィーチャーは、[価格のページ](security-center-pricing.md)に示されているように、スキャンごとではなくイメージごとに課金されます。 

Container Registries のバンドルを有効にすると、Security Center は、レジストリにプッシュされたイメージをスキャンできるようになります。 このスキャンはイメージレベルで実行されます。Security Center はレジストリをスキャンするのではなく、レジストリに格納されたイメージをスキャンしています。 

イメージがレジストリにプッシュされるたびに、Security Center はそのイメージを自動的にスキャンします。 イメージのスキャンをトリガーするには、イメージをリポジトリにプッシュしてください。


スキャンが完了すると (通常は約 10 分後)、次のような Security Center の推奨事項で結果が得られます。

[![Azure Container Registry (ACR) でホストされるイメージで検出された脆弱性に関する Azure Security Center の推奨事項のサンプル](media/azure-container-registry-integration/container-security-acr-page.png)](media/azure-container-registry-integration/container-security-acr-page.png#lightbox)

## <a name="benefits-of-integration"></a>統合の利点

Security Center では、サブスクリプション内の ACR レジストリが識別されて、次のものがシームレスに提供されます。

* プッシュされたすべての Linux イメージに対する **Azure ネイティブの脆弱性スキャン**。 Security Center では、業界をリードする脆弱性スキャン ベンダーである Qualys のスキャナーを使ってイメージのスキャンを行います。 このネイティブ ソリューションは、既定でシームレスに統合されています。

* 既知の脆弱性がある Linux イメージの**セキュリティに関する推奨事項**。 Security Center には、報告された各脆弱性の詳細と、重要度の分類が表示されます。 また、レジストリにプッシュされた各イメージで検出された特定の脆弱性を修復する方法についてのガイダンスも提供されます。

![Azure Security Center と Azure Container Registry (ACR) の概要](./media/azure-container-registry-integration/aks-acr-integration-detailed.png)

## <a name="next-steps"></a>次のステップ

Security Center のコンテナーのセキュリティ機能の詳細については、以下をご覧ください。

* [Azure Security Center とコンテナーのセキュリティ](container-security.md)

* [Azure Kubernetes Service との統合](azure-kubernetes-service-integration.md)

* [仮想マシンの保護](security-center-virtual-machine-protection.md) - Security Center の推奨事項について説明します。
