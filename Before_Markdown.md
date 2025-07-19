ご質問ありがとうございます。

「PowerShellの自作関数を紹介している記事で1番読まれている記事」を特定することは、検索エンジンの順位変動や集計サイトの有無から非常に困難です。しかし、多くのPowerShell入門記事やTIPS記事で「便利で実用的な自作関数の例」として、必ずと言っていいほど登場する**「ログ出力関数」**が存在します。

この関数は、汎用性が非常に高く、スクリプト開発の基本となるため、多くの開発者に読まれ、参考にされています。ここでは、その代表的なログ出力関数 `Write-Log` のコードとその解説を紹介します。

---

### 最も代表的な自作関数例： `Write-Log`

この関数は、スクリプトの実行状況をタイムスタンプ付きでファイルに記録するためのものです。

#### 関数のコード

```powershell
<#
.SYNOPSIS
    タイムスタンプとメッセージレベルを付けて、指定したファイルにログを追記します。

.DESCRIPTION
    スクリプトの実行状況を記録するために使用します。
    メッセージ、ログファイルのパスを必須パラメータとします。
    ログレベル（INFO, WARN, ERROR）を指定でき、デフォルトはINFOです。
    ファイルが存在しない場合は自動的に作成されます。

.PARAMETER Message
    ログに記録するメッセージ。

.PARAMETER Path
    ログを記録するファイルのフルパス。

.PARAMETER Level
    ログのレベルを指定します。'INFO', 'WARN', 'ERROR' から選択できます。

.EXAMPLE
    PS C:\> Write-Log -Message "処理を開始します。" -Path "C:\logs\script.log"
    # C:\logs\script.log に "[2023-10-27 10:00:00] [INFO] 処理を開始します。" と記録されます。

.EXAMPLE
    PS C:\> Write-Log "データベースへの接続に失敗しました。" "C:\logs\script.log" -Level ERROR
    # C:\logs\script.log に "[2023-10-27 10:01:00] [ERROR] データベースへの接続に失敗しました。" と記録されます。
#>
function Write-Log {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true, Position = 0)]
        [string]$Message,

        [Parameter(Mandatory = $true, Position = 1)]
        [string]$Path,

        [Parameter(Mandatory = $false)]
        [ValidateSet('INFO', 'WARN', 'ERROR')]
        [string]$Level = 'INFO'
    )

    # タイムスタンプを "yyyy-MM-dd HH:mm:ss" 形式で取得
    $timeStamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'

    # ログに出力する文字列を組み立てる
    $logLine = "[$timeStamp] [$Level] $Message"

    # ログファイルが存在するディレクトリを確認し、なければ作成する
    $logDirectory = Split-Path -Path $Path -Parent
    if (-not (Test-Path -Path $logDirectory)) {
        New-Item -Path $logDirectory -ItemType Directory | Out-Null
    }

    # Add-Content コマンドレットでファイルに追記する
    # -Encoding UTF8 を指定して文字化けを防ぐ
    try {
        Add-Content -Path $Path -Value $logLine -Encoding UTF8 -ErrorAction Stop
    }
    catch {
        # ログ書き込み自体に失敗した場合、コンソールにエラーを出力する
        Write-Error "ログファイルへの書き込みに失敗しました。 Path: $Path, Error: $($_.Exception.Message)"
    }
}
```

---

### 自作関数の解説

この `Write-Log` 関数がなぜ多くの記事で紹介されるのか、そのポイントを解説します。

#### 1. 関数の概要
この関数は、以下の3つの情報を組み合わせて、指定されたログファイルに1行のログを追記します。
*   **実行時刻（タイムスタンプ）**: いつ実行されたか
*   **ログレベル**: メッセージの重要度（情報、警告、エラーなど）
*   **メッセージ**: 何が起こったか

**例：** `[2023-10-27 10:00:00] [INFO] 処理を開始します。`

これにより、スクリプトの実行履歴を後から簡単に追跡できるようになります。

#### 2. 使い方（具体例）
関数をPowerShellのプロファイルに登録するか、スクリプトの先頭で読み込むことで、以下のように簡単に呼び出せます。

```powershell
# 基本的な使い方
Write-Log -Message "バックアップ処理を開始します。" -Path "C:\temp\backup.log"

# ログレベルを 'ERROR' にして記録
Write-Log -Message "ファイルのコピーに失敗しました。" -Path "C:\temp\backup.log" -Level ERROR

# パラメータ名を省略した使い方（Position属性のおかげ）
Write-Log "処理が正常に完了しました。" "C:\temp\backup.log"
```

#### 3. コードのポイント解説

この関数には、PowerShellの関数を作成する上での重要なテクニックが詰まっています。

*   **コメントベースのヘルプ (`<# ... #>`)**
    *   関数の先頭にあるコメントブロックは、`Get-Help Write-Log` コマンドで関数の使い方やパラメータの説明を表示させるためのものです。本格的な関数には必須の要素です。

*   **`[CmdletBinding()]`**
    *   これを付けることで、自作関数がPowerShellの標準コマンドレット（`Get-ChildItem`など）と同じように振る舞うようになります。例えば、`-Verbose` や `-Debug` などの共通パラメータが自動的に使えるようになります。

*   **`[Parameter(Mandatory = $true)]`**
    *   この属性が付いたパラメータ（`$Message`と`$Path`）は必須項目となります。指定し忘れると、PowerShellが自動的に入力を促してくれるため、エラーを防げます。

*   **`[ValidateSet('INFO', 'WARN', 'ERROR')]`**
    *   `$Level` パラメータに指定できる値を `'INFO'`, `'WARN'`, `'ERROR'` の3つに制限します。これにより、意図しない値（例: "error", "information"）が入力されるのを防ぎます。また、Tabキーによる入力補完も効くようになり、非常に便利です。

*   **ディレクトリの自動作成**
    *   `Split-Path` と `Test-Path` を使ってログファイルの保存先ディレクトリが存在するかを確認し、なければ `New-Item` で自動的に作成します。これにより、利用者は事前にフォルダを用意する手間が省けます。

*   **`Add-Content -Encoding UTF8`**
    *   `Add-Content` はファイルにテキストを追記するコマンドレットです。`-Encoding UTF8` を指定することで、日本語メッセージが文字化けするのを防ぎます。

*   **`try { ... } catch { ... }`**
    *   ログファイルの書き込みに失敗した場合（例：アクセス権がない）に備え、エラーハンドリングを行っています。これにより、スクリプト全体が停止するのを防ぎ、コンソールにエラー内容を明確に表示できます。

### まとめ

この `Write-Log` 関数が人気なのは、以下の理由からです。

1.  **実用性が高い**: どんなスクリプトでもログ出力はほぼ必須の機能です。
2.  **汎用性がある**: 特定の処理に依存せず、どこでも使い回せます。
3.  **学習効果が高い**: 関数の基本構造、パラメータ属性、エラー処理など、PowerShellの重要な概念を学ぶための最高の教材となります。

多くのPowerShellユーザーが、まずこの関数を自分用にカスタマイズして「マイ関数ライブラリ」に追加することから始めています。
