---
title: Azure AD DS で SharePoint ユーザー プロファイル サービスを有効にする | Microsoft Docs
description: SharePoint Server のプロファイルの同期をサポートするように Azure Active Directory Domain Services のマネージド ドメインを構成する方法について説明します
services: active-directory-ds
author: iainfoulds
manager: daveba
ms.assetid: 938a5fbc-2dd1-4759-bcce-628a6e19ab9d
ms.service: active-directory
ms.subservice: domain-services
ms.workload: identity
ms.topic: conceptual
ms.date: 01/21/2020
ms.author: iainfou
ms.openlocfilehash: f51744d79d34b734c9cc24cf72785a076a91a4f5
ms.sourcegitcommit: 38b11501526a7997cfe1c7980d57e772b1f3169b
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 01/22/2020
ms.locfileid: "76509055"
---
# <a name="configure-azure-active-directory-domain-services-to-support-user-profile-synchronization-for-sharepoint-server"></a>SharePoint Server のユーザー プロファイルの同期をサポートするように Azure Active Directory Domain Services を構成します。

SharePoint Server には、ユーザー プロファイルを同期するサービスが含まれています。 この機能を使用すると、ユーザー プロファイルを中央の場所に格納し、複数の SharePoint サイトおよびファームにアクセスできるようになります。 SharePoint Server ユーザー プロファイル サービスを構成するには、Azure Active Directory Domain Services (Azure AD DS) のマネージド ドメインで適切なアクセス許可を付与する必要があります。 詳細については、[SharePoint Server のユーザー プロファイルの同期](https://technet.microsoft.com/library/hh296982.aspx)に関する記事を参照してください。

この記事では、SharePoint Server ユーザー プロファイル同期サービスを許可するように Azure AD DS を構成する方法について説明します。

## <a name="before-you-begin"></a>開始する前に

この記事を完了するには、以下のリソースと特権が必要です。

* 有効な Azure サブスクリプション
    * Azure サブスクリプションをお持ちでない場合は、[アカウントを作成](https://azure.microsoft.com/free/?WT.mc_id=A261C142F)してください。
* ご利用のサブスクリプションに関連付けられた Azure Active Directory テナント (オンプレミス ディレクトリまたはクラウド専用ディレクトリと同期されていること)。
    * 必要に応じて、[Azure Active Directory テナントを作成][create-azure-ad-tenant]するか、[ご利用のアカウントに Azure サブスクリプションを関連付け][associate-azure-ad-tenant]ます。
* Azure AD テナントで有効化され、構成された Azure Active Directory Domain Services のマネージド ドメイン。
    * 必要に応じて、[Azure Active Directory Domain Services インスタンスを作成して構成する][create-azure-ad-ds-instance]チュートリアルを完了します。
* Azure AD DS マネージド ドメインに参加している Windows Server 管理 VM。
    * 必要に応じて、[管理 VM を作成する][tutorial-create-management-vm]チュートリアルを完了します。
* Azure AD テナントの *Azure AD DC administrators* グループのメンバーであるユーザー アカウント。
* ユーザー プロファイル同期サービスの SharePoint サービス アカウント。
    * 必要に応じて、「[SharePoint Server の管理アカウントとサービス アカウントを計画する][sharepoint-service-account]」を参照してください。

## <a name="service-accounts-overview"></a>サービス アカウントの概要

Azure AD DS マネージド ドメインでは、 **[AAD DC Service Accounts]\(AAD DC サービス アカウント\)** という名前のセキュリティ グループが *[ユーザー]* 組織単位 (OU) の一部として存在します。 このセキュリティ グループのメンバーには、次の特権が委任されます。

- ルート DSE に対する**ディレクトリの変更のレプリケート**特権。
- *[構成]* 名前付けコンテキスト (`cn=configuration` コンテナー) に対する**ディレクトリの変更のレプリケート**特権。

**[AAD DC Service Accounts]\(AAD DC サービス アカウント\)** セキュリティ グループは、組み込みグループの **Pre-Windows 2000 Compatible Access** のメンバーでもあります。

このセキュリティ グループに追加されると、SharePoint Server ユーザー プロファイル同期サービスのサービス アカウントに、正しく機能するために必要な特権が付与されます。

## <a name="enable-support-for-sharepoint-server-user-profile-sync"></a>SharePoint Server ユーザー プロファイルの同期のサポートを有効にする

SharePoint Server のサービス アカウントには、変更をディレクトリにレプリケートし、SharePoint Server ユーザー プロファイルの同期を正しく機能させるために適切な特権が必要です。 これらの特権を付与するには、**AAD DC Service Accounts** グループに、SharePoint のユーザー プロファイルの同期に使用するサービス アカウントを追加します。

Azure AD DS 管理 VM から、次の手順を実行します。

> [!NOTE]
> Azure AD DS マネージド ドメインでグループのメンバーシップを編集するには、*AAD DC 管理者*グループのメンバーであるユーザー アカウントにサインインする必要があります。

1. スタート画面で **[管理ツール]** を選択します。 [管理 VM を作成する][tutorial-create-management-vm]ためのチュートリアルでインストールされた使用可能な管理ツールの一覧が表示されます。
1. グループのメンバーシップを管理するには、管理ツールの一覧から **[Active Directory 管理センター]** を選択します。
1. 左側のウィンドウで、Azure AD DS マネージド ドメイン (*aadds.contoso.com* など) を選択します。 既存の OU とリソースの一覧が表示されます。
1. **[ユーザー]** OU を選択し、 *[AAD DC Service Accounts]\(AAD DC サービス アカウント\)* セキュリティ グループを選択します。
1. **[メンバー]** を選択し、 **[追加]** を選択します。
1. SharePoint サービス アカウントの名前を入力し、 **[OK]** を選択します。 次の例では、SharePoint サービス アカウントに *spadmin* という名前が付けられています。

    ![SharePoint サービス アカウントを [AAD DC Service Accounts]\(AAD DC サービス アカウント\) セキュリティ グループに追加する](./media/deploy-sp-profile-sync/add-member-to-aad-dc-service-accounts-group.png)

## <a name="next-steps"></a>次のステップ

詳細については、「[SharePoint Server でプロファイルを同期するために Active Directory Domain Services のアクセス許可を付与する](https://technet.microsoft.com/library/hh296982.aspx)」を参照してください

<!-- INTERNAL LINKS -->
[create-azure-ad-tenant]: ../active-directory/fundamentals/sign-up-organization.md
[associate-azure-ad-tenant]: ../active-directory/fundamentals/active-directory-how-subscriptions-associated-directory.md
[create-azure-ad-ds-instance]: tutorial-create-instance.md
[tutorial-create-management-vm]: tutorial-create-management-vm.md

<!-- EXTERNAL LINKS -->
[sharepoint-service-account]: /sharepoint/security-for-sharepoint-server/plan-for-administrative-and-service-accounts
