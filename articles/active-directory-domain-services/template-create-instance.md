---
title: テンプレートを使用して Azure DS Domain Services を有効にする | Microsoft Docs
description: Azure Resource Manager テンプレートを使用して Azure Active Directory Domain Services を構成して有効にする方法について説明します。
services: active-directory-ds
author: iainfoulds
manager: daveba
ms.service: active-directory
ms.subservice: domain-services
ms.workload: identity
ms.topic: conceptual
ms.date: 01/14/2020
ms.author: iainfou
ms.openlocfilehash: e63f330d463be21905467869474527fdf9d6abff
ms.sourcegitcommit: dbcc4569fde1bebb9df0a3ab6d4d3ff7f806d486
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 01/15/2020
ms.locfileid: "76030918"
---
# <a name="create-an-azure-active-directory-domain-services-managed-domain-using-an-azure-resource-manager-template"></a>Azure Resource Manager テンプレートを使用して Azure Active Directory Domain Services マネージド ドメインを作成する

Azure Active Directory Domain Services (Azure AD DS) では、Windows Server Active Directory と完全に互換性のあるマネージド ドメイン サービス (ドメイン参加、グループ ポリシー、LDAP、Kerberos 認証、NTLM 認証など) が提供されます。 ドメイン コントローラーのデプロイ、管理、パッチの適用を自分で行わなくても、これらのドメイン サービスを使用することができます。 Azure AD DS は、既存の Azure AD テナントと統合されます。 この統合により、ユーザーは、各自の会社の資格情報を使用してサインインすることができます。また管理者は、既存のグループとユーザー アカウントを使用してリソースへのアクセスをセキュリティで保護することができます。

この記事では、Azure Resource Manager テンプレートを使用して Azure AD DS を有効にする方法について説明します。 サポート リソースは、Azure PowerShell を使用して作成されます。

## <a name="prerequisites"></a>前提条件

この記事を完了するには、以下のリソースが必要です。

* Azure PowerShell のインストールおよび構成。
    * 必要であれば、手順に従って、[Azure PowerShell モジュールをインストールし、Azure サブスクリプションに接続](/powershell/azure/install-az-ps)します。
    * 必ず [Connect-AzAccount][Connect-AzAccount] コマンドレットを使用して Azure サブスクリプションにサインインしてください。
* Azure AD PowerShell をインストールして構成します。
    * 必要であれば、手順に従って、[Azure AD PowerShell モジュールをインストールして Azure AD に接続](/powershell/azure/active-directory/install-adv2)します。
    * 必ず [Connect-AzureAD][Connect-AzureAD] コマンドレットを使用して Azure AD テナントにサインインしてください。
* Azure AD DS を有効にするには、Azure AD テナントに "*全体管理者*" 特権が必要です。
* 必要な Azure AD DS リソースを作成するためには、ご利用の Azure サブスクリプションに "*共同作成者*" 特権が必要です。

## <a name="dns-naming-requirements"></a>DNS の名前付けに関する要件

Azure AD DS インスタンスを作成する際は、DNS 名を指定します。 この DNS 名を選ぶ際のいくつかの考慮事項を次に示します。

* **組み込みドメイン名:** 既定では、ディレクトリの組み込みドメイン名が使用されます ( *.onmicrosoft.com* サフィックス)。 マネージド ドメインに対するインターネット経由での Secure LDAP アクセスを有効にしたい場合、デジタル証明書を作成して、この既定のドメインとの接続をセキュリティで保護することはできません。 *.onmicrosoft.com* ドメインを所有するのは Microsoft であるため、証明機関 (CA) からは証明書が発行されません。
* **カスタム ドメイン名:** 最も一般的な方法は、カスタム ドメイン名を指定することです。一般に、貴社が既に所有していて、なおかつルーティング可能なものを指定します。 ルーティング可能なカスタム ドメインを使用すれば、ご利用のアプリケーションをサポートするために必要なトラフィックを正しく送信することができます。
* **ルーティング不可能なドメインのサフィックス:** 一般に、ルーティング不可能なドメイン名サフィックス (*contoso.local* など) は避けることをお勧めします。 *.local* サフィックスはルーティングできないため、DNS 解決で問題の原因となることがあります。

> [!TIP]
> カスタム ドメイン名を作成する場合は、既存の DNS 名前空間に注意してください。 ドメイン名には一意のプレフィックスを含めることをお勧めします。 たとえば、DNS ルート名が *contoso.com* である場合、*corp.contoso.com* または *ds.contoso.com* というカスタム ドメイン名で Azure AD DS マネージド ドメインを作成します。 オンプレミスの AD DS 環境を持つハイブリッド環境では、これらのプレフィックスが既に使用されている場合があります。 Azure AD DS には、一意のプレフィックスを使用してください。
>
> Azure AD DS マネージド ドメインにルート DNS 名を使用できますが、環境内の他のサービス用にいくつかの DNS レコードを追加作成することが必要になる場合があります。 たとえば、ルート DNS 名を使用するサイトをホストする Web サーバーを実行する場合、名前の競合が発生して、追加の DNS エントリが必要になる可能性があります。
>
> これらのチュートリアルとハウツー記事では、簡略な例として *aadds.contoso.com* というカスタム ドメインを使用しています。 すべてのコマンドで、一意のプレフィックスを含む、独自のドメイン名を指定してください。
>
> 詳細については、[ドメインの名前付けプレフィックスの選択][naming-prefix]に関するページを参照してください。

DNS 名には、次の制限も適用されます。

* **ドメインのプレフィックスの制限:** プレフィックスが 15 文字より長いマネージド ドメインを作成できません。 指定するドメイン名のプレフィックス (たとえば、ドメイン名 *contoso.com* の *contoso* など) は、15 文字以内に収める必要があります。
* **ネットワーク名の競合:** マネージド ドメインの DNS ドメイン名は、まだ仮想ネットワークに存在していない必要があります。 具体的には、名前の競合につながる次のシナリオをチェックしてください。
    * 同じ DNS ドメイン名の Active Directory ドメインが Azure 仮想ネットワーク上に既にあるかどうか。
    * マネージド ドメインを有効にする仮想ネットワークに、オンプレミス ネットワークとの VPN 接続があるかどうか。 このシナリオでは、オンプレミス ネットワークに同じ DNS ドメイン名のドメインがないことを確認します。
    * その名前の付いた Azure クラウド サービスが Azure 仮想ネットワーク上に既にあるかどうか。

## <a name="create-required-azure-ad-resources"></a>必要な Azure AD リソースを作成する

Azure AD DS には、サービス プリンシパルと Azure AD グループが必要です。 これらのリソースにより、Azure AD DS マネージド ドメインはデータを同期し、マネージド ドメインで管理アクセス許可を持つユーザーを定義できます。

まず、[Register-AzResourceProvider][Register-AzResourceProvider] コマンドレットを使用して、Azure AD Domain Services リソース プロバイダーを登録します。

```powershell
Register-AzResourceProvider -ProviderNamespace Microsoft.AAD
```

Azure AD DS が通信し、自身を認証するようにするには、[New-AzureADServicePrincipal][New-AzureADServicePrincipal] コマンドレットを使用して Azure AD サービス プリンシパルを作成します。 ID *2565bd9d-da50-47d4-8b85-4c97f669dc36* を持つ *Domain Controller Services* という名前の特定のアプリケーション ID が使用されます。 このアプリケーション ID は変更しないでください。

```powershell
New-AzureADServicePrincipal -AppId "2565bd9d-da50-47d4-8b85-4c97f669dc36"
```

次に、[New-AzureADGroup][New-AzureADGroup] コマンドレットを使用して *AAD DC Administrators* という名前の Azure AD グループを作成します。 このグループに追加されたユーザーには、Azure AD DS マネージド ドメインで管理タスクを実行するためのアクセス許可が付与されます。

```powershell
New-AzureADGroup -DisplayName "AAD DC Administrators" `
  -Description "Delegated group to administer Azure AD Domain Services" `
  -SecurityEnabled $true -MailEnabled $false `
  -MailNickName "AADDCAdministrators"
```

*AAD DC Administrators* グループが作成されたら、[Add-AzureADGroupMember][Add-AzureADGroupMember] コマンドレットを使用してグループにユーザーを追加します。 まず、[Get-AzureADGroup][Get-AzureADGroup] コマンドレットを使用して *AAD DC Administrators* グループのオブジェクト ID を取得し、次に [Get-AzureADUser][Get-AzureADUser] コマンドレットを使用して目的のユーザーのオブジェクト ID を取得します。

次の例では、UPN が `admin@contoso.onmicrosoft.com` のアカウントのユーザー オブジェクト ID です。 このユーザー アカウントを、*AAD DC Administrators* グループに追加するユーザーの UPN に置き換えます。

```powershell
# First, retrieve the object ID of the newly created 'AAD DC Administrators' group.
$GroupObjectId = Get-AzureADGroup `
  -Filter "DisplayName eq 'AAD DC Administrators'" | `
  Select-Object ObjectId

# Now, retrieve the object ID of the user you'd like to add to the group.
$UserObjectId = Get-AzureADUser `
  -Filter "UserPrincipalName eq 'admin@contoso.onmicrosoft.com'" | `
  Select-Object ObjectId

# Add the user to the 'AAD DC Administrators' group.
Add-AzureADGroupMember -ObjectId $GroupObjectId.ObjectId -RefObjectId $UserObjectId.ObjectId
```

最後に、[New-AzResourceGroup][New-AzResourceGroup] コマンドレットを使用してリソース グループを作成します。 次の例では、リソース グループは *myResourceGroup* という名前が付けられ、*westus* リージョンに作成されます。 独自の名前と希望するリージョンを使用します。

```powershell
New-AzResourceGroup `
  -Name "myResourceGroup" `
  -Location "WestUS"
```

Availability Zones がサポートされているリージョンを選択すると、Azure AD DS リソースが、冗長性強化のために複数のゾーンに分散されます。 Availability Zones は、Azure リージョン内の一意の物理的な場所です。 それぞれのゾーンは、独立した電源、冷却手段、ネットワークを備えた 1 つまたは複数のデータセンターで構成されています。 回復性を確保するため、有効になっているリージョンにはいずれも最低 3 つのゾーンが別個に存在しています。

Azure AD DS を複数のゾーンに分散するために、ご自身で構成するものは何もありません。 Azure プラットフォームでは、ゾーンへのリソース分散が自動的に処理されます。 詳細情報および利用可能なリージョンについては、「[Azure の Availability Zones の概要][availability-zones]」を参照してください。

## <a name="resource-definition-for-azure-ad-ds"></a>Azure AD DS のためのリソース定義

Resource Manager リソース定義の一部として、次の構成パラメーターが必要です。

| パラメーター               | 値 |
|-------------------------|---------|
| domainName              | マネージド ドメインの DNS ドメイン名。プレフィックスの名前付けや競合に関する前のポイントを考慮に入れてください。 |
| filteredSync            | Azure AD DS では、Azure AD に存在する "*すべて*" のユーザーとグループを同期できるほか、特定のグループのみを "*範囲指定*" して同期することもできます。 すべてのユーザーとグループを同期することを選択した場合は、後で、範囲指定された同期のみの実行を選択することはできません。<br /> 範囲指定された同期の詳細については、[Azure AD Domain Services の範囲指定された同期][scoped-sync]に関するページを参照してください。|
| notificationSettings    | Azure AD DS マネージド ドメインで生成されたアラートがある場合は、電子メール通知を送信できます。 <br />Azure テナントの*全体管理者*と *AAD DC Administrators* グループのメンバーは、これらの通知に対して *[有効]* にすることができます。<br /> 必要に応じて、注意を必要とするアラートが発生した場合の通知の受信者を追加できます。|
| domainConfigurationType | 既定では、Azure AD DS のマネージド ドメインが "*ユーザー*" フォレストとして作成されます。 このタイプのフォレストでは、オンプレミスの AD DS 環境で作成されたユーザー アカウントも含め、Azure AD 内のすべてのオブジェクトが同期されます。 ユーザー フォレストを作成するために *domainConfiguration* 値を指定する必要はありません。<br /> "*リソース*" フォレストでは、Azure AD に直接作成されたユーザーとグループだけが同期されます。 リソース フォレストは現在プレビュー段階です。 リソース フォレストを作成するには、この値を *ResourceTrusting* に設定します。<br />リソース フォレストを使用する理由や、オンプレミスの AD DS ドメインを使用してフォレストの信頼を作成する方法など、"*リソース*" フォレストの詳細については、[Azure AD DS リソース フォレストの概要][resource-forests]に関するページを参照してください。|

次の圧縮されたパラメーターの定義は、これらの値がどのように宣言されるかを示しています。 *aadds.contoso.com* という名前のユーザー フォレストは、Azure AD DS マネージド ドメインに同期されている Azure AD のすべてのユーザーで作成されます。

```json
"parameters": {
    "domainName": {
        "value": "aadds.contoso.com"
    },
    "filteredSync": {
        "value": "Disabled"
    },
    "notificationSettings": {
        "value": {
            "notifyGlobalAdmins": "Enabled",
            "notifyDcAdmins": "Enabled",
            "additionalRecipients": []
        }
    },
    [...]
}
```

その後、次の圧縮された Resource Manager テンプレート リソースの種類を使用して、Azure AD DS マネージド ドメインが定義および作成されます。 Azure 仮想ネットワークとサブネットが既に存在するか、または Resource Manager テンプレートの一部として作成される必要があります。 Azure AD DS マネージド ドメインは、このサブネットに接続されます。

```json
"resources": [
    {
        "apiVersion": "2017-06-01",
        "type": "Microsoft.AAD/DomainServices",
        "name": "[parameters('domainName')]",
        "location": "[parameters('location')]",
        "dependsOn": [
            "[concat('Microsoft.Network/virtualNetworks/', parameters('vnetName'))]"
        ],
        "properties": {
            "domainName": "[parameters('domainName')]",
            "subnetId": "[concat('/subscriptions/', subscription().subscriptionId, '/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/virtualNetworks/', parameters('vnetName'), '/subnets/', parameters('subnetName'))]",
            "filteredSync": "[parameters('filteredSync')]",
            "notificationSettings": "[parameters('notificationSettings')]"
        }
    },
    [...]
]
```

次のセクションに示すように、これらのパラメーターとリソースの種類は、マネージド ドメインをデプロイするためのより広範囲の Resource Manager テンプレートの一部として使用できます。

## <a name="create-a-managed-domain-using-sample-template"></a>サンプル テンプレートを使用してマネージド ドメインを作成する

次の完全な Resource Manager サンプル テンプレートでは、Azure AD DS マネージド ドメインとサポートする仮想ネットワーク、サブネット、およびネットワーク セキュリティ グループの規則が作成されます。 ネットワーク セキュリティ グループの規則は、マネージド ドメインをセキュリティで保護し、トラフィックを正常に送受信できることを確認するために必要です。 *aadds.contoso.com* の DNS 名を持つユーザー フォレストが作成され、すべてのユーザーが Azure AD から同期されます。

```json
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "apiVersion": {
            "value": "2017-06-01"
        },
        "domainConfigurationType": {
            "value": "FullySynced"
        },
        "domainName": {
            "value": "aadds.contoso.com"
        },
        "filteredSync": {
            "value": "Disabled"
        },
        "location": {
            "value": "westus"
        },
        "notificationSettings": {
            "value": {
                "notifyGlobalAdmins": "Enabled",
                "notifyDcAdmins": "Enabled",
                "additionalRecipients": []
            }
        },
        "subnetName": {
            "value": "aadds-subnet"
        },
        "vnetName": {
            "value": "aadds-vnet"
        },
        "vnetAddressPrefixes": {
            "value": [
                "10.1.0.0/24"
            ]
        },
        "subnetAddressPrefix": {
            "value": "10.1.0.0/24"
        },
        "nsgName": {
            "value": "aadds-nsg"
        }
    },
    "resources": [
        {
            "apiVersion": "2017-06-01",
            "type": "Microsoft.AAD/DomainServices",
            "name": "[parameters('domainName')]",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[concat('Microsoft.Network/virtualNetworks/', parameters('vnetName'))]"
            ],
            "properties": {
                "domainName": "[parameters('domainName')]",
                "subnetId": "[concat('/subscriptions/', subscription().subscriptionId, '/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/virtualNetworks/', parameters('vnetName'), '/subnets/', parameters('subnetName'))]",
                "filteredSync": "[parameters('filteredSync')]",
                "domainConfigurationType": "[parameters('domainConfigurationType')]",
                "notificationSettings": "[parameters('notificationSettings')]"
            }
        },
        {
            "type": "Microsoft.Network/NetworkSecurityGroups",
            "name": "[parameters('nsgName')]",
            "location": "[parameters('location')]",
            "properties": {
                "securityRules": [
                    {
                        "name": "AllowSyncWithAzureAD",
                        "properties": {
                            "access": "Allow",
                            "priority": 101,
                            "direction": "Inbound",
                            "protocol": "Tcp",
                            "sourceAddressPrefix": "AzureActiveDirectoryDomainServices",
                            "sourcePortRange": "*",
                            "destinationAddressPrefix": "*",
                            "destinationPortRange": "443"
                        }
                    },
                    {
                        "name": "AllowPSRemoting",
                        "properties": {
                            "access": "Allow",
                            "priority": 301,
                            "direction": "Inbound",
                            "protocol": "Tcp",
                            "sourceAddressPrefix": "AzureActiveDirectoryDomainServices",
                            "sourcePortRange": "*",
                            "destinationAddressPrefix": "*",
                            "destinationPortRange": "5986"
                        }
                    },
                    {
                        "name": "AllowRD",
                        "properties": {
                            "access": "Allow",
                            "priority": 201,
                            "direction": "Inbound",
                            "protocol": "Tcp",
                            "sourceAddressPrefix": "CorpNetSaw",
                            "sourcePortRange": "*",
                            "destinationAddressPrefix": "*",
                            "destinationPortRange": "3389"
                        }
                    }
                ]
            },
            "apiVersion": "2018-04-01"
        },
        {
            "type": "Microsoft.Network/virtualNetworks",
            "name": "[parameters('vnetName')]",
            "location": "[parameters('location')]",
            "apiVersion": "2018-04-01",
            "dependsOn": [
                "[concat('Microsoft.Network/NetworkSecurityGroups/', parameters('nsgName'))]"
            ],
            "properties": {
                "addressSpace": {
                    "addressPrefixes": "[parameters('vnetAddressPrefixes')]"
                },
                "subnets": [
                    {
                        "name": "[parameters('subnetName')]",
                        "properties": {
                            "addressPrefix": "[parameters('subnetAddressPrefix')]",
                            "networkSecurityGroup": {
                                "id": "[concat('/subscriptions/', subscription().subscriptionId, '/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/NetworkSecurityGroups/', parameters('nsgName'))]"
                            }
                        }
                    }
                ]
            }
        }
    ],
    "outputs": {}
}
```

このテンプレートは、[Azure portal][portal-deploy]、[Azure PowerShell][powershell-deploy]、CI/CD パイプラインなどの、推奨されるデプロイ方法を使用してデプロイできます。 次の例では、[New-AzResourceGroupDeployment][New-AzResourceGroupDeployment] コマンドレットを使用します。 独自のリソース グループ名とテンプレート ファイル名を指定します。

```powershell
New-AzResourceGroupDeployment -ResourceGroupName "myResourceGroup" -TemplateFile <path-to-template>
```

リソースが作成され、PowerShell プロンプトに制御が戻るまで数分かかります。 Azure AD DS マネージド ドメインは引き続きバックグラウンドでプロビジョニングされ、デプロイが完了するまでには最大 1 時間かかる場合があります。 Azure portal では、お使いの Azure AD DS マネージド ドメインの **[概要]** ページに、このデプロイ ステージ全体の現在の状態が表示されます。

Azure AD DS マネージド ドメインがプロビジョニングを完了したことが Azure portal に示されたら、次のタスクを完了する必要があります。

* 仮想マシンがマネージド ドメインを検出してドメイン参加または認証を行うことができるように、仮想ネットワークの DNS 設定を更新します。
    * DNS を構成するには、ポータルで Azure AD DS のマネージド ドメインを選択します。 **[概要]** ウィンドウで、これらの DNS 設定を自動的に構成するように求められます。
* [Azure AD Domain Services とのパスワード同期を有効にして](tutorial-create-instance.md#enable-user-accounts-for-azure-ad-ds)、エンド ユーザーが会社の資格情報を使用してマネージド ドメインにサインインできるようにします。

## <a name="next-steps"></a>次のステップ

動作中の Azure AD DS マネージド ドメインを確認するために、[Windows VM のドメイン参加][windows-join]、[Secure LDAP の構成][tutorial-ldaps]、および[パスワード ハッシュ同期の構成][tutorial-phs]を実行できます。

<!-- INTERNAL LINKS -->
[windows-join]: join-windows-vm.md
[tutorial-ldaps]: tutorial-configure-ldaps.md
[tutorial-phs]: tutorial-configure-password-hash-sync.md
[availability-zones]: ../availability-zones/az-overview.md
[portal-deploy]: ../azure-resource-manager/templates/deploy-portal.md
[powershell-deploy]: ../azure-resource-manager/templates/deploy-powershell.md
[scoped-sync]: scoped-synchronization.md
[resource-forests]: concepts-resource-forest.md

<!-- EXTERNAL LINKS -->
[Connect-AzAccount]: /powershell/module/Az.Accounts/Connect-AzAccount
[Connect-AzureAD]: /powershell/module/AzureAD/Connect-AzureAD
[New-AzureADServicePrincipal]: /powershell/module/AzureAD/New-AzureADServicePrincipal
[New-AzureADGroup]: /powershell/module/AzureAD/New-AzureADGroup
[Add-AzureADGroupMember]: /powershell/module/AzureAD/Add-AzureADGroupMember
[Get-AzureADGroup]: /powershell/module/AzureAD/Get-AzureADGroup
[Get-AzureADUser]: /powershell/module/AzureAD/Get-AzureADUser
[Register-AzResourceProvider]: /powershell/module/Az.Resources/Register-AzResourceProvider
[New-AzResourceGroup]: /powershell/module/Az.Resources/New-AzResourceGroup
[Get-AzSubscription]: /powershell/module/Az.Accounts/Get-AzSubscription
[cloud-shell]: /azure/cloud-shell/cloud-shell-windows-users
[naming-prefix]: /windows-server/identity/ad-ds/plan/selecting-the-forest-root-domain
[New-AzResourceGroupDeployment]: /powershell/module/Az.Resources/New-AzResourceGroupDeployment
