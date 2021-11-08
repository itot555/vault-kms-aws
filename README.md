# Snapshots - BYOK with Vault

# KMS - AWS

[キーマネージメントシークレットエンジン](https://www.vaultproject.io/docs/secrets/key-management)は、様々なキーマネージメントサービス（KMS）プロバイダーの暗号キーの配布とライフサイクル管理のための一貫したワークフローを提供出来る[Vault Enterprise](https://www.hashicorp.com/products/vault/pricing)の機能になります。

これにより、企業や組織は、KMSプロバイダのネイティブな暗号機能を利用しながらも、Vaultのインターフェースを利用し、キーを集中管理することが出来る様になります。

Vaultのキーマネジメントシークレットエンジンは、キーのオリジナルコピーを生成し、保持します。Vaultの管理者が、サポートされているKMSプロバイダのいずれかでキーの配布とライフサイクル管理を行うと、キーのコピーがKMSプロバイダに配布されます。

Vaultが作成したキーは、サポートされているKMSプロバイダーのキーのインポート仕様に基づいて、常に安全に転送されます。

以下のステップで実施していきます。

- [KMSシークレットエンジンの設定](#KMSシークレットエンジンの設定)
- [テストキーの作成](#テストキーの作成)
- [キーの配布と暗号化復号オペレーション](#キーの配布と暗号化復号オペレーション)
- [キーのローテーション](#キーのローテーション)
- [テストキーの削除](#テストキーの削除)

## KMSシークレットエンジンの設定

ライセンスが有効になっている事を確認します。`features`に、`Key Management Secrets Engine`が含まれている事を確認します。

```
vault read -format=json sys/license/status | jq .
```

KMSシークレットエンジンを有効化します。

```
vault secrets enable keymgmt
vault secrets list
```

バックエンドのAWS KMSとVaultのキーマネジメントシークレットエンジンが連携出来る様に設定します。

```console
$ vault write keymgmt/kms/aws \
      key_collection="ap-northeast-1" \
      provider="awskms" \
      credentials=access_key="${AWS_ACCESS_KEY_ID}" \
      credentials=secret_key="${AWS_SECRET_ACCESS_KEY}" \
      credentials=session_token="${AWS_SESSION_TOKEN}"
Success! Data written to: keymgmt/kms/aws
```

設定しているAWSのシークレットとして、必要な権限は[こちら](https://www.vaultproject.io/docs/secrets/key-management/awskms#authentication)を参考にしてください。

## テストキーの作成

テストキーを作成します。Vault1.8では、AWS KMSでサポートされるキーのタイプは`aes256-gcm96`のみです。詳細は、[Compatibility](https://www.vaultproject.io/docs/secrets/key-management#compatibility)をご参照ください。

```console
$ vault write keymgmt/key/test-key type="aes256-gcm96"
Success! Data written to: keymgmt/key/test-key
```

作成したテストキーの詳細を確認してみます。

```
vault read -format=json keymgmt/key/test-key | jq .
```

- 注: ここで作成したテストキーは共通鍵(symmetric key)になります。

## キーの配布と暗号化復号オペレーション

作成したテストキーをAWS KMSバックエンドにプッシュします。KMSプロバイダにキーを配布する際に利用できるパラメータは[こちら](https://www.vaultproject.io/api/secret/key-management#parameters-9)から確認する事が出来ます。

```console
$ vault write keymgmt/kms/aws/key/test-key \
    purpose=encrypt,decrypt \
    protection=hsm
Success! Data written to: keymgmt/kms/aws/key/test-key
```

`purpose`: 配布するキーが利用する機能を指定します。AWS KMSにおいては、`encrypt`と`decrypt`のみサポートされています。
`protection`: KMSプロバイダ内で配布するキーを利用して、どこで暗号化のオペレーションを行うか指定します。`hsm`、もしくは`software`を指定する事が出来ます。

ランダムに作成されたAWS Key IDを確認します。キーの名前はリージョンで一意である必要があります。

```console
$ export KEY_ID=$(aws kms list-aliases | \
    jq -r '.Aliases[] | select(.AliasName | contains("test-key")).TargetKeyId')
```
```console
$ echo $KEY_ID
a689e544-...
```

配布したキーと`aws` CLIを用いて、プレーンテキストを暗号化してみます。

```console
$ export ENCODED_CIPHERTEXT=$(aws kms encrypt \
    --no-cli-pager \
    --key-id ${KEY_ID} \
    --plaintext $(echo hashicorp | base64) | \
    jq -r .CiphertextBlob)
```

配布したキー(AWS KMSに登録されたカスタマーマネージドキー)を利用して、前のステップでアウトプットされた暗号文を復号します。

```console
$ export DECRYPT_RESPONSE=$(aws kms decrypt \
    --no-cli-pager \
    --ciphertext-blob ${ENCODED_CIPHERTEXT} \
    --key-id ${KEY_ID})
```

`aws` CLIからのレスポンスは、復号された暗号文はbase64エンコードされたJSONオブジェクトになります。レスポンスされたbase64エンコードをデコードします。

```console
$ echo ${DECRYPT_RESPONSE} | jq -r '.Plaintext' | base64 -d
hashicorp
```

ここまでで、AWS KMSの外、Vaultでキーを作成し、それをAWS KMSにカスタマーマネージドキーとして配布し、暗号化と復号のオペレーションが可能であるという事を確認する事が出来ました。

## キーのローテーション

キーをローテーションを行うと、新しいキーマテリアルを含む新しいキーのバージョンが作成されます。キーは、Vaultとそのキーが配布されたKMSプロバイダの両方でローテーションされます。

```
vault write -f keymgmt/key/test-key/rotate
```

ローテーションされたキーの情報を確認します。

```
vault read -format=json keymgmt/key/test-key | jq .
```

キーのIDを確認します。

```console
$ export KEY_ID=$(aws kms list-aliases | \
    jq -r '.Aliases[] | select(.AliasName | contains("test-key")).TargetKeyId')
```
```console
$ echo $KEY_ID
7ac53350-...
```

この状態で暗号文を復号しようとすると、キーがローテションされているため、復号処理が失敗します。

```console
$ export DECRYPT_RESPONSE=$(aws kms decrypt \
    --no-cli-pager \
    --ciphertext-blob ${ENCODED_CIPHERTEXT} \
    --key-id ${KEY_ID})
An error occurred (IncorrectKeyException) when calling the Decrypt operation: The key ID in the request does not identify a CMK that can perform this operation.
```

元のバージョンのキーIDを`KEY_ID`に設定し直して、再度復号処理を行うと成功します。

```console
$ export KEY_ID=a689e544-...
```
```console
$ export DECRYPT_RESPONSE=$(aws kms decrypt \
    --no-cli-pager \
    --ciphertext-blob ${ENCODED_CIPHERTEXT} \
    --key-id ${KEY_ID})
```
```console
$ echo ${DECRYPT_RESPONSE} | jq -r '.Plaintext' | base64 -d
hashicorp
```

利用できるキーのバージョン設定を行うことも可能です。
キーの`min_enabled_version` を設定する事で、作成したキーのバージョンを有効または無効にすることが可能です。`min_enabled_version` 未満のすべてのキーバージョンは、そのキーが配布されたKMSプロバイダでのオペレーションに対して無効になります。`min_enabled_version` を0に設定すると、すべてのキーのバージョンが有効になります。

以下の様に変更を行い、有効なキーバージョンを`2`以降に設定します。

```
vault write keymgmt/key/test-key min_enabled_version=2
```

バージョン1のキーIDを用いて、再度復号処理を行うと、キーが無効化されたため、オペレーションが失敗します。

```console
$ export DECRYPT_RESPONSE=$(aws kms decrypt \
    --no-cli-pager \
    --ciphertext-blob ${ENCODED_CIPHERTEXT} \
    --key-id ${KEY_ID})
An error occurred (DisabledException) when calling the Decrypt operation: arn:aws:kms:.../a689e544-... is disabled.
```

## テストキーの削除

作成したキーを削除します。

```console
$ vault delete keymgmt/kms/aws/key/test-key
Success! Data deleted (if it existed) at: keymgmt/kms/aws/key/test-key
```

もう一度、`aws` CLIで復号の操作をしてみます。既にキーを削除したので、復号の処理が失敗すると思います。

```console
$ aws kms decrypt \
    --no-cli-pager \
    --ciphertext-blob ${ENCODED_CIPHERTEXT} \
    --key-id ${KEY_ID}
An error occurred (KMSInvalidStateException) when calling the Decrypt operation: arn:aws:kms:...:key/... is pending deletion.
```

## Reference

- [AWS KMS](https://www.vaultproject.io/docs/secrets/key-management/awskms)
- [Key Management Secrets Engine](https://www.vaultproject.io/docs/secrets/key-management#kms-providers)
- [Key Management Secrets Engine (API)](https://www.vaultproject.io/api/secret/key-management#key-management-secrets-engine-api)