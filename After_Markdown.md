承知いたしました。
PowerShellの自作関数を紹介する記事で最も広く読まれ、多くの開発者に利用されているテーマの一つが **「高機能なログ出力関数」** です。特定の「1番読まれている記事」を一つに絞ることは難しいですが、それらの人気記事で紹介されている機能を統合し、実用性を高めた代表的な関数のコードと解説を以下に示します。

この関数は、単にメッセージをファイルに書き出すだけでなく、ログレベル、タイムスタンプ、呼び出し元情報の記録など、本格的なスクリプト開発に不可欠な機能を備えています。

---

### :rocket: 高機能ログ出力関数 `Write-Log`

多くのPowerShellスクリプト開発者が作成する、鉄板とも言える自作関数です。スクリプトの実行状況を追跡し、エラー発生時の原因調査を容易にします。

#### 主な機能

- **ログレベル対応**: `INFO`, `DEBUG`, `WARN`, `ERROR` のレベルを付けてログを記録できます。
- **コンソールとファイルへの同時出力**: 実行中の画面でリアルタイムに進捗を確認しつつ、ファイルにも永続的な記録を残します。
- **自動タイムスタンプ**: 全てのログに `yyyy/MM/dd HH:mm:ss` 形式のタイムスタンプが自動で付与されます。
- **呼び出し元情報の記録**: ログが「どのスクリプトの」「何行目から」呼び出されたかを記録し、デバッグを強力に支援します。
- **色分け表示**: ログレベルに応じてコンソールの文字色が変わるため、警告やエラーを視覚的に素早く認識できます。

### :memo: 関数のコード

<details>
<summary><strong>Write-Log 関数の完全なコードはこちら</strong></summary>

```powershell
function Write-Log {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true, Position = 0)]
        [string]$Message,

        [Parameter(Mandatory = $false, Position = 1)]
        [ValidateSet('INFO', 'DEBUG', 'WARN', 'ERROR')]
        [string]$Level = 'INFO',

        [Parameter(Mandatory = $false)]
        [string]$Path = $Global:LogPath # グローバル変数 $LogPath をデフォルト値として使用
    )

    # 1. ログメッセージの組み立て
    # タイムスタンプ
    $timeStamp = Get-Date -Format "yyyy/MM/dd HH:mm:ss"

    # 呼び出し元の情報を取得 (スクリプト名と行番号)
    # Get-PSCallStack の最初の要素は Write-Log 自体なので、2番目([1])の要素を取得
    $caller = Get-PSCallStack | Select-Object -Skip 1 -First 1
    if ($caller) {
        $callerInfo = "[{0}:{1}]" -f (Split-Path $caller.ScriptName -Leaf), $caller.ScriptLineNumber
    }
    else {
        # PowerShellプロンプトから直接実行された場合
        $callerInfo = "[Console]"
    }

    # 最終的なログ文字列
    $logLine = "[{0}] [{1,-5}] {2} {3}" -f $timeStamp, $Level, $callerInfo, $Message

    # 2. コンソールへの色付き出力
    $color = "White" # デフォルトカラー
    switch ($Level) {
        'DEBUG' { $color = "Gray" }
        'WARN'  { $color = "Yellow" }
        'ERROR' { $color = "Red" }
    }
    Write-Host $logLine -ForegroundColor $color

    # 3. ファイルへの出力
    if (-not ([string]::IsNullOrWhiteSpace($Path))) {
        try {
            # ログディレクトリが存在しない場合は作成
            $logDir = Split-Path -Path $Path -Parent
            if (-not (Test-Path -Path $logDir)) {
                New-Item -Path $logDir -ItemType Directory -Force | Out-Null
            }
            # ファイルに追記 (UTF8 BOMなし)
            Add-Content -Path $Path -Value $logLine -Encoding UTF8 -NoNewline
        }
        catch {
            # ログファイルへの書き込みに失敗した場合のエラー処理
            $errorMessage = "ログファイルへの書き込みに失敗しました。 Path: {0}, Error: {1}" -f $Path, $_.Exception.Message
            Write-Host $errorMessage -ForegroundColor Red
        }
    }
}
```

</details>

### :gear: 関数の解説

#### 1. 基本的な使い方

この関数は、スクリプトの先頭で一度読み込ませるか、PowerShellのプロファイルに登録しておくことで、いつでも呼び出せるようになります。

スクリプト内でログファイルパスをグローバル変数として定義しておくと、毎回パスを指定する手間が省けて便利です。

```powershell
# スクリプトの最初にログファイルのパスを定義
$Global:LogPath = "C:\Logs\MyScript_$(Get-Date -Format 'yyyyMMdd').log"

# --- ここに Write-Log 関数の定義を記述するか、. (ドット) ソーシングで読み込む ---
# . C:\path\to\MyFunctions.ps1

# 関数の呼び出し例
Write-Log "処理を開始します。"
Write-Log "設定ファイルを読み込みました。" -Level DEBUG
Write-Log "対象のファイルが見つかりません。" -Level WARN
Write-Log "データベースへの接続に失敗しました。" -Level ERROR
```

**実行結果（コンソール）**

![コンソールでの色分けされたログ出力のイメージ](https://user-images.githubusercontent.com/1098993/207901179-8898b173-9528-444f-8086-1d18ca80d60c.png)
*(これは出力のイメージです)*

**実行結果（ログファイル）**

```text
[2023/10/27 10:00:00] [INFO ] [MyScript.ps1:10] 処理を開始します。
[2023/10/27 10:00:01] [DEBUG] [MyScript.ps1:11] 設定ファイルを読み込みました。
[2023/10/27 10:00:05] [WARN ] [MyScript.ps1:12] 対象のファイルが見つかりません。
[2023/10/27 10:00:10] [ERROR] [MyScript.ps1:13] データベースへの接続に失敗しました。
```

#### 2. パラメータ詳説

| パラメータ  | 説明                                                                                                              | 型      | 必須 | デフォルト値         |
| :---------- | :---------------------------------------------------------------------------------------------------------------- | :------ | :--- | :------------------- |
| `-Message`  | ログとして記録したいメッセージ。                                                                                  | `string`| **はい** | (なし)               |
| `-Level`    | ログの重要度レベル (`INFO`, `DEBUG`, `WARN`, `ERROR` から選択)。                                                     | `string`| いいえ | `INFO`               |
| `-Path`     | ログを出力するファイルのフルパス。指定しない場合、グローバル変数 `$Global:LogPath` の値が使用されます。 | `string`| いいえ | `$Global:LogPath` |

#### 3. コードのポイント解説

- **`[CmdletBinding()]` と `[Parameter()]`**
  - これらは関数をPowerShellの標準コマンドレット（Cmdlet）のように振る舞わせるための「おまじない」です。パラメータの必須指定や入力値の検証 (`ValidateSet`) が可能になります。

- **`Get-PSCallStack`**
  - この関数の **最も強力な機能** の一つです。このコマンドレットは、現在の場所がどの関数やスクリプトから呼び出されたかの履歴（コールスタック）を取得します。
  - `Select-Object -Skip 1 -First 1` を使うことで、「`Write-Log`関数を呼び出した`1`つ上の階層の情報」だけを抜き出し、呼び出し元のスクリプト名と行番号を取得しています。

- **`switch ($Level)`**
  - `-Level` パラメータの値に応じて、`Write-Host` で表示する文字色を動的に変更しています。これにより、コンソール上での視認性が劇的に向上します。

- **`Add-Content -Encoding UTF8`**
  - ファイルへテキストを追記するコマンドレットです。
  - `-Encoding UTF8` を指定することで、日本語などのマルチバイト文字が文字化けするのを防ぎます。PowerShellのバージョンによってはデフォルトのエンコーディングが異なるため、明示的に指定することが推奨されます。

---

### :thinking: なぜこの関数が人気なのか？

1.  **汎用性の高さ**
    - どのような種類のPowerShellスクリプトにも組み込むことができ、一度作っておけば様々な場面で再利用できます。

1.  **デバッグの劇的な効率化**
    - スクリプトが複雑になると、「どこで」「なぜ」問題が起きたのかを追跡するのが困難になります。この関数を使えば、処理のどの段階でエラーが発生したかが一目瞭然となり、原因究明の時間を大幅に短縮できます。

1.  **スクリプトの品質向上**
    - 適切なログ出力は、自分以外の人がスクリプトをメンテナンスする際の大きな助けとなります。処理の流れが追いやすくなり、属人化を防ぎます。
