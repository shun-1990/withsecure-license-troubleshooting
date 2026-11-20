# WithSecureセットアップ・ライセンス未承認の切り分けと再実行

## 目的・いつ使うか

サーバーの初期キッティングで、PowerShellからWithSecure Elements Agentを入れたのに、アプリだけ追加されてライセンス（サブスクリプションキー）が承認されていないときに使う。

想定している表示の例：

- `Your subscription is not activated. Please enter a valid subscription key to start using the product`

## 結論（先に判断）

- **原則：アンインストールしない。アプリが残っている状態でキーを再登録する。**
- 理由：再インストールすると新しいUUIDが生成され、Elementsポータルに同じ端末が重複登録される。通常のアンインストールではポータルから自動削除されるが、キッティングのたびに繰り返すと台帳が汚れる。
- アンインストール→再インストールは、キー再登録でも直らない場合の最終手段（手順C）。

> 注意：このページにはライセンスキー実物、顧客名、ホスト名、IPアドレスを書かない。キーは `<SUBSCRIPTION_KEY>` と表記する。

## 切り分け：どこを確認するか（上から順に）

止まった原因は大きく6つ。確認した順に潰すと、再実行が一発で通る。

### 1. ライセンスキー自体（ポータル側）

確認場所：Elements Security Center ポータル > **Management > Subscriptions**

- [ ] 対象キーの **Expiration** が期限内か
- [ ] 製品が合っているか（サーバー用 EPP for Servers のキーか。PC用キーをサーバーに入れていないか。キーが製品を決める仕様）
- [ ] 空きライセンス数（使用数上限）に達していないか
- [ ] スクリプト側のキーに打ち間違い、全角・前後の空白、改行が混ざっていないか

判定：ここがNGなら端末側を何度流しても直らない。キーの差し替え・更新が先。

### 2. ポータル側の端末状態（削除・ブロックリスト）

確認場所：ポータル > **Environment > Devices**、および Subscriptions

- [ ] 対象端末が Devices に出ているか、出ていないか
- [ ] 過去にその端末（または同じキー）がポータルから削除されていないか

削除された端末はブロックリストに入り、同じキーで再登録できなくなることがある。その場合のエラー例：

- `You cannot use the subscription key you entered for this computer anymore.`

判定：このエラーが出たら手順B（ブロックリスト解除）が必須。端末の再インストールだけでは直らない。

### 3. ポータルへの通信

確認場所：端末上で接続チェックツール実行

```powershell
& "C:\Program Files (x86)\F-Secure\PSB\ui\wsconnectionchecker.exe"
```

- [ ] WithSecureバックエンド（`*.fsapi.com` 系）への接続がすべて成功するか
- [ ] インターネットに出るのにHTTPプロキシが要る環境なのに、プロファイル／接続設定にプロキシが入っていないか
- [ ] キッティング用ネットワークが一時的にしか疎通していない状態で流していないか

判定：通信NGなら、キー登録は必ず失敗する。ネットワーク／プロキシを直してから手順Aへ。

### 4. PowerShellスクリプトの流し方

- [ ] 管理者権限で実行したか
- [ ] インストーラーの終了コードを見ているか（`$LASTEXITCODE` や `Start-Process -Wait -PassThru` の `ExitCode` を確認せずに「成功」としていないか）
- [ ] **EXEインストーラーのファイル名を変えていないか**（EXEインストーラーはファイル名にサブスクリプションキーを含む形式がある。リネームするとキーが渡らない。サイレント実行 `--silent` も、ファイル名にキーが入っていることが前提）
- [ ] 既にアプリが入っている状態でインストーラーを再実行していないか（インストール済み判定でキー登録部分が素通りすることがある）

判定：スクリプトが終了コードを握りつぶしていると、失敗しても気づけない。以後のキッティング用に、下記「スクリプト改善版」を使う。

### 5. 端末側の状態

- [ ] 再起動待ち（キッティング中の他アプリ／Windows Update起因）が残っていないか。一度再起動してから表示が変わらないか
- [ ] Elements AgentのUIで **Status details** を開き、どの項目が緑でないか
- [ ] **Settings > Central Management** の **Device ID（UUID）** を控え、ポータルの Devices 検索でそのUUIDを引けるか（ポータル側の重複・別人登録の切り分けに使う）

### 6. ログ・診断情報（上記で絞れないとき）

- [ ] WithSecure Support Tool で診断ファイル（wsdiag / fsdiag.tar.gz）を取得する。デスクトップに作成される
- [ ] サポートへ渡す場合は診断ファイルを添付する。自分で見る場合も、まず 1〜5 のどれに当たるかを診断前に当たりを付ける

## 問題がなければ：実行手順

### 手順A：キー再登録（第一選択・アンインストールなし）

前提：切り分け 1〜3 がすべてOK。

管理者のコマンドプロンプト／PowerShellで実行する。

```powershell
cd "C:\Program Files (x86)\F-Secure\PSB\"
.\ws_oneclient_logout.exe --keycode <SUBSCRIPTION_KEY>
echo $LASTEXITCODE
```

- このツールは現在のサブスクリプションを解除し、指定キーで即座に再登録する。再登録だけなら `--keycode` なしの実行や `--nokeycode` は使わない（キーを外すだけになる）
- 戻り値 **0 = 成功**。0以外（ネットワーク不可、キー不正など）の場合、Agentは未承認状態のままになるので、数字を控えて切り分けに戻る
- フォルダが見つからない場合は `C:\Program Files (x86)\F-Secure\` 直下も探す（バージョンにより配置が異なる）

確認方法（うまくいった判定）：

1. 数分待ってAgent UIの **Status details** がすべて緑になる
2. Settings > Central Management の Device ID（UUID）でポータルの **Environment > Devices** を検索し、対象端末が `Protected` で出る

### 手順B：ブロックリスト解除（手順Aで「そのキーはもう使えない」と出たとき）

1. ポータル > **Subscriptions** を開く
2. 対象キーの右端の「…」から **Restore devices from blocklist** を実行する
   - 別経路：Devices 側の **Manage removed devices** から対象端末を **activate** する
3. そのあと手順Aをもう一度実行する

### 手順C：アンインストール→再インストール（最終手段）

手順A・Bでも直らないときだけ。

1. 通常手順でアンインストールし、再起動する
2. ポータルの Devices から旧端末が消えているか確認する（残っていたら重複の元になる）
3. 再インストールする。重複登録を防ぎたい場合は識別子を固定する
   - EXEの場合：`--use_smbios_guid` パラメータを付ける
   - MSIの場合：`UNIQUE_SIGNUP_ID=smbios` を指定する
4. 新しいキーで再インストールすると、ポータルには別端末として新規作成される点に注意する

## スクリプト改善版（以後キッティングで流す形）

ポイントは3つだけ：通信を先に確認する／インストーラーのファイル名を変えない／終了コードで止める。

```powershell
$ErrorActionPreference = 'Stop'
$Key = '<SUBSCRIPTION_KEY>'  # 実キーはGit・チャットに残さない。配布時は安全な方法で渡す

# 1) 既に入っている場合は、インストーラー再実行ではなくキー再登録を行う
$Tool = 'C:\Program Files (x86)\F-Secure\PSB\ws_oneclient_logout.exe'
if (Test-Path $Tool) {
    $p = Start-Process $Tool -ArgumentList "--keycode $Key" -Wait -PassThru -NoNewWindow
    if ($p.ExitCode -ne 0) { throw "WithSecure key registration failed. ExitCode=$($p.ExitCode)" }
    Write-Host 'WithSecure: key re-registered.'
    return
}

# 2) 新規の場合は元のファイル名のままインストーラーを実行し、終了コードを確認する
#    （EXEはファイル名にキーを含む形式があるため、リネームしない）
$Installer = '.\<ORIGINAL_INSTALLER_FILENAME>.exe'
$p = Start-Process $Installer -ArgumentList '--silent --use_smbios_guid' -Wait -PassThru -NoNewWindow
if ($p.ExitCode -ne 0) { throw "WithSecure installer failed. ExitCode=$($p.ExitCode)" }
Write-Host 'WithSecure: installed.'
```

流したあとは必ず、手順Aの「確認方法」2点（UIが緑／ポータルにUUIDで出る）まで見てキッティング完了とする。「アプリが入った」だけでは完了にしない。

## ハマりどころ・失敗したら

- インストーラーを流し直しても直らない → インストール済み判定でキー登録が素通りしている可能性。手順Aを使う
- キーを入れ直しても未承認のまま → 戻り値が0かを確認する。0でなければ通信（切り分け3）かキー（切り分け1）
- ポータルに端末が2つ出る → 再インストールでUUIDが変わった重複。古い方をポータルから整理してから納品する
- 何度やっても同じキーで弾かれる → ブロックリスト（手順B）を疑う。再インストールの繰り返しでは直らない

## 出典

- WithSecure公式サポート記事集（Endpoint Protection）：「subscription is not activated」時の `ws_oneclient_logout.exe --keycode` による再入力、接続チェックツール `wsconnectionchecker.exe` の場所、ブロックリストからの復旧、再インストール時のUUID重複と `--use_smbios_guid` / `UNIQUE_SIGNUP_ID=smbios`
- 確認日：2026-11-20
