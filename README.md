# ソフトウェア開発においての、ログまわりの整理・まとめ

## はじめに
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-08-32-24.png)
Ubuntuを日常的に使っていると挙動に不審な点があったり決まったアプリケーションが落ちてしまうなどの機会に遭遇します。
そういった時にログを簡単に確認できればいいのですが、ちょっとしたことに時間を取られたくなかったり、そもそもログを確認することが生産性と直結しないなど、なかなかログを確認して対策する習慣がつかないのではないでしょうか？

私は日常的にLinuxで顔認証システムの開発を行っており、普段からUbuntuを使用しています。本記事では、日常の開発や使用時に役立つログ確認のポイントと、効率的にログを確認するためのTipsを紹介します。システム管理者やネットワークエンジニアではありませんので、もしかしたら非効率な点や誤った点があるかも知れません。その場合は恐縮ですがコメントいただけると非常に嬉しいです。

この記事のお伝えしたい章は「[4. 便利なログを確認するアプリケーションの紹介](#4-便利なログを確認するアプリケーションの紹介)」です。
もっと気軽に見やすくログを確認するためのユーティリティ的アプリケーションをご紹介することが目的です。
ですので、従来の基本的なログ確認方法に興味のない方は、その部分を飛ばして4章をご参照ください。

ログの詳しい仕組みやセキュリティに関する詳細はここでは扱いません。(`syslog.conf`とか)
**サーバー監視が目的ではなく、ソフトウェア開発をする実機のトラブルシューティングの為のログ確認が目的です。**

ログ確認ユーティリティの検証環境は`GNOME Boxes`内の`Ubuntu 24.04 LTS`を用いました。

この記事が基本的なログの確認方法について理解を深める一助になればと思います。

![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/eye-catch.png)

## 構成
1. どのようなログの種類があるか
2. ログを確認するケーススタディ
3. 従来のコマンドラインの紹介
4. 便利なログを確認するアプリケーションの紹介
   1. ksystemlog, logs, lnav, frontailなどいろいろ。

- [ソフトウェア開発においての、ログまわりの整理・まとめ](#ソフトウェア開発においてのログまわりの整理まとめ)
  - [はじめに](#はじめに)
  - [構成](#構成)
  - [環境](#環境)
  - [1. どのようなログの種類があるか](#1-どのようなログの種類があるか)
  - [2. ログを確認する（従来の）ケーススタディ](#2-ログを確認する従来のケーススタディ)
    - [一般的なトラブルシューティング](#一般的なトラブルシューティング)
    - [パフォーマンスの監視](#パフォーマンスの監視)
    - [セキュリティの監視](#セキュリティの監視)
  - [3. `dmesg`, `journalctl`](#3-dmesg-journalctl)
    - [`dmesg`](#dmesg)
    - [`journalctl`](#journalctl)
    - [`dmesg`の引数とオプション](#dmesgの引数とオプション)
      - [主要な引数とオプション](#主要な引数とオプション)
      - [`grep`や`less`との組み合わせ](#grepやlessとの組み合わせ)
      - [使用例](#使用例)
    - [`journalctl`の引数とオプション](#journalctlの引数とオプション)
      - [主要な引数とオプション](#主要な引数とオプション-1)
      - [使用例](#使用例-1)
    - [`dmesg`と`journalctl`で重複するログ](#dmesgとjournalctlで重複するログ)
    - [`dmesg`でしか確認できないログ](#dmesgでしか確認できないログ)
    - [`journalctl`でしか確認できないログ](#journalctlでしか確認できないログ)
  - [4. 便利なログを確認するアプリケーションの紹介](#4-便利なログを確認するアプリケーションの紹介)
    - [**GNOME Logs**](#gnome-logs)
      - [特長](#特長)
      - [欠点](#欠点)
      - [ディスプレイマネージャ対応](#ディスプレイマネージャ対応)
    - [**KSystemLog**](#ksystemlog)
      - [特長](#特長-1)
      - [欠点](#欠点-1)
      - [ディスプレイマネージャ対応](#ディスプレイマネージャ対応-1)
    - [**Lnav (Log File Navigator)**](#lnav-log-file-navigator)
      - [特長](#特長-2)
      - [欠点](#欠点-2)
      - [ディスプレイマネージャ対応](#ディスプレイマネージャ対応-2)
    - [**Multitail**](#multitail)
      - [特長](#特長-3)
      - [欠点](#欠点-3)
      - [ディスプレイマネージャ対応](#ディスプレイマネージャ対応-3)
    - [**Logwatch**](#logwatch)
    - [**LOGalyze**](#logalyze)
    - [**Glogg**](#glogg)
      - [特長](#特長-4)
      - [欠点](#欠点-4)
      - [ディスプレイマネージャ対応](#ディスプレイマネージャ対応-4)
  - [最後に](#最後に)
  - [参考文献](#参考文献)


## 環境
```bash
# 実環境
inxi -Sxxx --filter
System:
  Kernel: 6.5.0-35-generic x86_64 bits: 64 compiler: N/A Desktop: Unity
    wm: gnome-shell dm: GDM3 42.0 Distro: Ubuntu 22.04.4 LTS (Jammy Jellyfish)

# アプリケーション検証環境(GNOME Boxes内のUbuntu24.04LTS使用)
inxi -Sxxx --filter
System:
  Kernel: 6.8.0-31-generic arch: x86_64 bits: 64 compiler: gcc v: 13.2.0
    clocksource: kvm-clock
  Desktop: GNOME v: 46.0 tk: GTK v: 3.24.41 wm: gnome-shell
    tools: gsd-screensaver-proxy dm: GDM3 v: 46.0 Distro: Ubuntu 24.04 LTS
    (Noble Numbat)

```

## 1. どのようなログの種類があるか
※ ログファイルは（ほとんど）すべて`/var/log/`ディレクトリ以下に存在します。
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-31-20-44-21.png)

| カテゴリ（ファシリティ）        | ログファイル名       | 説明                                                                                          |
|------------------|------------------|---------------------------------------------------------------------------------------------|
| システム全般のログ | `syslog`         | システム全般のメッセージや各種デーモンのメッセージ。                                          |
|                  | `kern.log`       | カーネル関連のメッセージ、ハードウェアやカーネルモジュールのメッセージ。                        |
| 認証関連のログ    | `auth.log`       | 認証関連のメッセージ、ログインや認証の成功・失敗。                                            |
| アプリケーション固有のログ | `apport.log`     | クラッシュレポート、アプリケーションのクラッシュ詳細。                                          |
|                  | `dpkg.log`       | パッケージのインストールや削除に関するメッセージ。                                              |
| Xorg関連のログ    | `Xorg.0.log`     | Xサーバーのメッセージ、ディスプレイ関連の問題。                                                  |
| その他のログ      | `faillog`        | ログイン失敗の情報。                                                                            |
|                  | `lastlog`        | 最後のログイン情報。                                                                            |
|                  | `wtmp`           | 現在ログインしているユーザー情報。                                                              |
| ブートログ        | `boot.log`       | システムのブートプロセスに関するメッセージが記録されます。システムの起動時に何が起こったかを確認するために使用します。 |
| デーモンのログ    | `daemon.log`     | 各種デーモン（バックグラウンドサービス）のメッセージが記録されます。特定のデーモンに関連する問題をトラブルシュートするために使用します。 |
| メッセージログ    | `messages`       | システム全般の重要なメッセージが記録されます。システムの運用に関連する重要なイベントを追跡するために使用します。       |
| cronログ         | `cron.log`       | cronジョブ（スケジュールされたタスク）の実行に関するメッセージが記録されます。cronジョブが正しく実行されているかを確認するために使用します。 |

## 2. ログを確認する（従来の）ケーススタディ
以下に昔から使われるコマンド例を紹介します。
後述しますが、現在はもっと便利で見やすいユーティリティがあります。そちらを知りたい方はここを飛ばして「## 4. 便利なログを確認するアプリケーションの紹介」をご覧ください。
### 一般的なトラブルシューティング
- **システムがクラッシュした場合**: `syslog`や`kern.log`を確認。
  - コマンド例:
    - `tail -f /var/log/syslog`: リアルタイムで`syslog`の末尾を表示。
    - `grep 'error' /var/log/syslog`: `syslog`内の特定のキーワード（ここでは'error'）を含むログを検索。
    - `less /var/log/kern.log`: `kern.log`をページごとに表示。

- **認証エラーが発生した場合**: `auth.log`を確認。
  - コマンド例:
    - `tail -f /var/log/auth.log`: リアルタイムで`auth.log`の末尾を表示。
    - `grep 'failed' /var/log/auth.log`: `auth.log`内の特定のキーワード（ここでは'failed'）を含むログを検索。
    - `less /var/log/auth.log`: `auth.log`をページごとに表示。

- **アプリケーションがクラッシュした場合**: `apport.log`を確認。
  - コマンド例:
    - `tail -f /var/log/apport.log`: リアルタイムで`apport.log`の末尾を表示。
    - `grep 'CRITICAL' /var/log/apport.log`: `apport.log`内の特定のキーワード（ここでは'CRITICAL'）を含むログを検索。
    - `less /var/log/apport.log`: `apport.log`をページごとに表示。

### パフォーマンスの監視
- **サーバー（or 実機）のパフォーマンスを監視する際**: `syslog`やアプリケーション固有のログを定期的に確認。
  - コマンド例:
    - `tail -f /var/log/syslog | grep 'performance'`: `syslog`内のパフォーマンス関連のメッセージをリアルタイムで表示。
    - `less /var/log/syslog`: `syslog`をページごとに表示。

### セキュリティの監視
- **不正ログインを監視する際**: `auth.log`や`faillog`を確認。
  - コマンド例:
    - `tail -f /var/log/auth.log | grep 'invalid'`: `auth.log`内の不正ログイン試行をリアルタイムで表示。
    - `faillog`: `faillog`を表示し、ログイン失敗の情報を確認。

- **システムの整合性を確認する際**: `lastlog`や`wtmp`を確認。
  - コマンド例:
    - `lastlog`: 最後のログイン情報を表示。
    - `who`: 現在ログインしているユーザーを表示。
    - `last`: `wtmp`を基に、ログイン・ログアウトの履歴を表示。

## 3. `dmesg`, `journalctl`
`dmesg`と`journalctl`は、ログの確認を効率化するユーティリティです。
### `dmesg`
`dmesg`は、"display message"の略。初期のUnixシステムから存在する伝統的なユーティリティです。
カーネルはブート時や動作中に生成するメッセージをリングバッファに保存します。
システム運用中に発生するカーネルエラーやハードウェアエラーの確認に役立ちます。
特定のキーワードを含むメッセージの検索や、特定の条件でフィルタリングして使います。
### `journalctl`
`journalctl`は、`systemd`のジャーナルログを管理・表示するためのコマンドです。
従来の`SysVinit`にかわり2010年ごろから`systemd`が導入され始めました。従来のテキストファイルに保存されていたログを、バイナリ形式で効率的に管理・検索するためのユーティリティが`journalctl`です。新し目のユーティリティゆえに、条件設定やカラーハイライトが充実しています。
`dmesg`と`journalctl`はどちらもシステムログを表示するためのコマンドですが、扱うログの範囲と詳細に違いがあります。（後述）
### `dmesg`の引数とオプション
フィルタリングや検索を行うためには、`grep`や`less`などの他のコマンドと組み合わせて使用します。
以下に`dmesg`の主要な引数とオプションをいくつか紹介します。
#### 主要な引数とオプション
- `-H`または`--human`: 人間が読みやすい形式で出力します。
  ```bash
  dmesg -H
  ```
- `-T`: タイムスタンプを人間が読みやすい形式で表示します。
  ```bash
  dmesg -T
  ```
- `-l`または`--level`: 指定したログレベルのメッセージのみを表示します。
  ```bash
  dmesg -l err  # エラーレベルのメッセージのみ表示
  ```
#### `grep`や`less`との組み合わせ
- 特定のキーワードを含むメッセージを検索するには、`grep`を使用します。
  ```bash
  dmesg | grep 'error'
  ```
- ページごとに表示するには、`less`を使用します。
  ```bash
  dmesg | less
  ```
#### 使用例
```bash
dmesg -T | grep 'error'  # タイムスタンプ付きでカーネルメッセージ内のエラーを検索
```

### `journalctl`の引数とオプション
#### 主要な引数とオプション
- `-e`または`--pager-end`: 最後のメッセージから表示します。
  ```bash
  journalctl -e
  ```
- `-f`または`--follow`: リアルタイムでログをフォローします（`tail -f`に似ています）。
  ```bash
  journalctl -f
  ```
- `-u`または`--unit`: 特定のサービスユニットのログを表示します。
  ```bash
  journalctl -u sshd.service
  ```
- `-p`または`--priority`: 指定した優先度のメッセージのみを表示します。
  ```bash
  journalctl -p err  # エラーレベルのメッセージのみ表示
  ```
- `--since`および`--until`: 特定の時間範囲内のログを表示します。
  ```bash
  journalctl --since "2023-05-01" --until "2023-05-02"
  ```
#### 使用例
```bash
journalctl -xe  # 詳細なsystemdジャーナルログを表示
journalctl -u sshd.service | grep 'Failed'  # SSHサービスの認証失敗メッセージを検索
journalctl --since "2023-05-01" --until "2023-05-02"  # 特定の時間範囲のログを表示
```

### `dmesg`と`journalctl`で重複するログ
- **カーネルメッセージ**:
  - **`dmesg`**: カーネルメッセージを表示。
  - **`journalctl -k`**: `systemd`ジャーナル内のカーネルメッセージを表示。
### `dmesg`でしか確認できないログ
- **ブートローダーメッセージ**:
  - `dmesg`はシステムの起動直後からカーネルが生成するメッセージを表示するため、ブートローダー（例えばGRUB）の後すぐのメッセージ、例えばカーネルの初期化・ハードウェアの検出などのログを確認できます。カーネルメッセージやハードウェアに関連するログに特化していると認識してます。
  - `journalctl`は`systemd`によって管理されるジャーナルログを参照するので、`systemd`がロードされる前の状態は（あまり）関与しません。（カーネルのログはみれます）
### `journalctl`でしか確認できないログ
- **systemdサービスのログ**:
  - `journalctl`は`systemd`のジャーナルを表示するため、各種サービスユニットのログも含まれます。これには、`sshd`、`nginx`、`apache2`などのサービスのログが含まれます。特定のサービスのログは以下のように確認します。
    ```bash
    journalctl -u sshd.service
    ```
- **ユーザーセッションログ**:
  - `journalctl`はユーザーごとのログも管理しています。例えば特定のユーザーセッションに関連するログは以下のように確認します。
    ```bash
    journalctl --user
    ```
- 便利機能:
  - `journalctl`は、ログエントリに対して詳細なタイムスタンプやフィルタリングを行うことができます。
    ```bash
    journalctl --since "2023-05-01" --until "2023-05-02"
    ```
    ログレベルは以下から選べます。
    - 0: emerg (緊急)
    - 1: alert (警報)
    - 2: crit (重大)
    - 3: err (エラー)
    - 4: warning (警告)
    - 5: notice (通知)
    - 6: info (情報)
    - 7: debug (デバッグ)
    例えばエラー以上のログを表示したい場合は以下のようにします。
    ```bash
    journalctl --user -p err
    ```
    結構たくさんエラーが出てくるものです。
    そこで特定のユニットなどに対してフィルターをかけたくなります。
    ```bash
    journalctl --user -u gnome-shell -p warning..err
    ```
    ただ雑感としては、`snap`や`flatpak`などのパーミッション関連、`GNOME-shell`機能拡張の問題などが多い印象です。

参考：[【2023年1月版】 journalctl 覚えておきたい使い方メモ](https://qiita.com/nouernet/items/c60ff2621385f4d8f7b6)


以下は`journalctl`コマンドのヘルプの翻訳です。

```bash
journalctl [オプション...] [マッチ...]

ジャーナルをクエリします。

オプション:
     --system                システムジャーナルを表示
     --user                  現在のユーザーのためのユーザージャーナルを表示
  -M --machine=CONTAINER     ローカルコンテナで操作
  -S --since=DATE            指定した日付より新しいエントリを表示
  -U --until=DATE            指定した日付より古いエントリを表示
  -c --cursor=CURSOR         指定したカーソルからエントリを表示
     --after-cursor=CURSOR   指定したカーソルの後のエントリを表示
     --show-cursor           すべてのエントリの後にカーソルを表示
     --cursor-file=FILE      ファイルのカーソルの後のエントリを表示し、ファイルを更新
  -b --boot[=ID]             現在のブートまたは指定したブートを表示
     --list-boots            記録されたブートの簡潔な情報を表示
  -k --dmesg                 現在のブートのカーネルメッセージログを表示
  -u --unit=UNIT             指定したユニットのログを表示
     --user-unit=UNIT        指定したユーザーユニットのログを表示
  -t --identifier=STRING     指定したsyslog識別子のエントリを表示
  -p --priority=RANGE        指定した優先度のエントリを表示
     --facility=FACILITY...  指定したファシリティのエントリを表示
  -g --grep=PATTERN          パターンに一致するメッセージのエントリを表示
     --case-sensitive[=BOOL] 大文字小文字の区別を強制
  -e --pager-end             ページャーの最後にジャンプ
  -f --follow                ジャーナルをフォロー
  -n --lines[=INTEGER]       表示するジャーナルエントリの数
     --no-tail               フォローモードでもすべての行を表示
  -r --reverse               最新のエントリを最初に表示
  -o --output=STRING         ジャーナル出力モードを変更（short, short-precise, short-iso, short-iso-precise, short-full, short-monotonic, short-unix, verbose, export, json, json-pretty, json-sse, json-seq, cat, with-unit）
     --output-fields=LIST    verbose/export/jsonモードで表示するフィールドを選択
     --utc                   時間を協定世界時（UTC）で表示
  -x --catalog               可能な場合はメッセージの説明を追加
     --no-full               フィールドを省略
  -a --all                   長いフィールドや印刷不可能なフィールドを含むすべてのフィールドを表示
  -q --quiet                 情報メッセージと特権警告を表示しない
     --no-pager              出力をページャーにパイプしない
     --no-hostname           ホスト名フィールドを抑制
  -m --merge                 利用可能なすべてのジャーナルからエントリを表示
  -D --directory=PATH        ディレクトリからジャーナルファイルを表示
     --file=PATH             ジャーナルファイルを表示
     --root=ROOT             ルートディレクトリ以下のファイルで操作
     --image=IMAGE           ファイルシステムイメージのファイルで操作
     --namespace=NAMESPACE   指定した名前空間からジャーナルデータを表示
     --interval=TIME         FSSシーリングキーの変更時間間隔
     --verify-key=KEY        FSS検証キーを指定
     --force                 --setup-keysでFSSキーのペアをオーバーライド

コマンド:
  -h --help                  このヘルプテキストを表示
     --version               パッケージバージョンを表示
  -N --fields                現在使用されているすべてのフィールド名をリスト
  -F --field=FIELD           指定したフィールドが取るすべての値をリスト
     --disk-usage            すべてのジャーナルファイルのディスク使用量を表示
     --vacuum-size=BYTES     指定したサイズ以下にディスク使用量を削減
     --vacuum-files=INT      指定した数のジャーナルファイルを残す
     --vacuum-time=TIME      指定した時間より古いジャーナルファイルを削除
     --verify                ジャーナルファイルの整合性を確認
     --sync                  書き込み中のジャーナルメッセージをディスクに同期
     --relinquish-var        ディスクへのログ記録を停止し、一時ファイルシステムにログ
     --smart-relinquish-var  似たような操作だが、ログディレクトリがルートマウント上にある場合はNOP
     --flush                 /runからすべてのジャーナルデータを/varにフラッシュ
     --rotate                ジャーナルファイルの即時回転を要求
     --header                ジャーナルヘッダー情報を表示
     --list-catalog          カタログ内のすべてのメッセージIDを表示
     --dump-catalog          メッセージカタログのエントリを表示
     --update-catalog        メッセージカタログデータベースを更新
     --setup-keys            新しいFSSキーのペアを生成

詳細はjournalctl(1)のmanページを参照してください。
```

## 4. 便利なログを確認するアプリケーションの紹介
さて、ここまでは従来のログ確認方法を紹介してきました。
ここまでの内容はシステム管理者の方やネットワークエンジニアの方には物足りなく、ソフトウェア開発者の方には退屈だったと思います。
この章では、もっと気軽に見やすくログを確認するためのユーティリティ的アプリケーションをご紹介します。
### **GNOME Logs**
- システムログを簡単に表示およびフィルタリング可能。
- インストール: `sudo snap install gnome-logs`
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-09-11-37.png)
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-09-59-55.png)
Ubuntu標準のログ確認ツールです。
#### 特長
- GNOMEデスクトップ環境に統合されており、使いやすいUI
- ファシリティや重要度でフィルタリング可能
- キーワード検索機能
#### 欠点
- エラーメッセージや警告メッセージのカラーリング機能なし
- 公式のドキュメントが貧弱すぎる
#### ディスプレイマネージャ対応
- GNOME環境で動作するため、X11およびWaylandの両方に対応

### **KSystemLog**
- インストール: `sudo apt install ksystemlog`
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-09-14-33.png)
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-09-23-25.png)
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-09-24-29.png)
#### 特長
- ファシリティのフィルタリングや検索
- ビュー設定をカスタマイズ可能
- プライオリティ別のカラーリング対応
- タブ表示可能
- エントリの追加機能
- 多機能ながら見やすく使いやすい設計
- [ドキュメント](https://docs.kde.org/stable5/en/ksystemlog/ksystemlog/index.html)がしっかりしている
#### 欠点
- GNOME環境であれば多くの依存ファイルをインストールしなくてはならない
- といってもGNOME環境で普通にKDEアプリケーションを使っているなら問題にならない
- コマンドラインに慣れていると、あれみたいのにどうやるの？的な逆転現象が起きる
#### ディスプレイマネージャ対応
- X11およびWaylandの両方に対応

### **Lnav (Log File Navigator)**
- 複数のログファイルを単一のインターフェースで表示、エラーや警告を自動的にハイライト。
- インストール: `sudo apt install lnav`
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-09-26-58.png)
参考：
- [Features: official](https://lnav.org/features)
- [コンソール上でログをカラフルに見やすく表示してくれるログビューアコマンド『lnav』](https://orebibou.com/ja/home/201503/20150310_001/)
#### 特長
- 複数のログファイルをリアルタイムで監視可能
- 正規表現を用いたフィルタリングや検索が可能
- ログのエラー、警告、その他の重要なメッセージを色分けして表示
#### 欠点
- CLIで好みが分かれる
- 学習曲線が少しだけ高い
- 実機のログをみたいだけならオーバーキル気味。
#### ディスプレイマネージャ対応
- CLIゆえ、どちらにも対応

### **Multitail**
  - 複数のログファイルを同時に表示、フィルタリング機能付き。
  - インストール: `sudo apt install multitail`
参考：
- [MultiTailで効率的にログを見る](https://qiita.com/Brutus/items/3e6425c84142902da376)
- [MultiTail: See two, three and more logs in real time at the same time](https://blog.desdelinux.net/en/multitail-sees-two-three-and-more-logs-in-real-time-at-the-same-time/)
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-09-49-03.png)
#### 特長
- 複数のログファイルを同時に表示し、リアルタイムで監視可能
- 若干のカラーリング
#### 欠点
- 多数のログファイルを同時に監視するとリソース消費が増加
#### ディスプレイマネージャ対応
- CLIゆえ、どちらにも対応

### **Logwatch**
- インストール: `sudo apt install logwatch`
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-12-52-06.png)
- 診断メールを送ってくれる
- 設定ができなかったため省略

### **LOGalyze**
- deb, snap, flatpakいずれでもインストールできなかったため省略

### **Glogg**
- インストール: `sudo apt install glogg`
![](https://raw.githubusercontent.com/yKesamaru/how_to_logging/master/assets/2024-05-28-13-26-03.png)
#### 特長
- 自分が注目するログファイルを手動でオープンするタイプ
- タブ追加ができる
- 正規表現を使用可能(`grep` + `less`という感じ)
- GNOME LOGSを野暮ったくした印象
- ファイルの自動リロードや複数ファイルの同時閲覧が可能
- [ドキュメント](https://glogg.bonnefon.org/documentation.html)がちゃんとある
#### 欠点
- ある程度勘所がわかっている人でないと使いにくそう。
#### ディスプレイマネージャ対応
- Qtベースのため、waylandに対応していると思うが未検証。

## 最後に
わたしの雑感ですが、基本は`KSystemLog`で確認、詳しくみたければコマンドラインで、という方法が一番しっくりくる印象です。多分やり方はあるのだと思うのですが、あのログ見たいけど`KSystemLog`の設定だとどこいじればいいの？という場面がありました。そこらへんが解決できればこれ一択になるような気がしました。

以上です。ありがとうございました。

## 参考文献
- [15 Best Log Viewers and Log Analysis Tools for Linux](https://www.ubuntupit.com/best-linux-log-viewer-and-log-file-management-tools/)
- [The KSystemLog Handbook](https://docs.kde.org/stable5/en/ksystemlog/ksystemlog/index.html)