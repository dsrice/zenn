---
title: "開発環境の整備のためのホットリロード"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang","HotReload", "air"]
published: false
---

## 概要

EchoのAPI開発環境の整備のためにホットリロードを導入する。
また、WindowsでのDocker環境を使ったため少々特殊な事象が発生したのでそちらについてもまとめる。

## ホットリロードの必要性

GoのWebアプリケーションの開発環境において、ホットリロードが欲しくなるケースはある。
それは、一度実行されてしまうと状態を保持してしまうためだ。
何かしらの修正を行った際に、実行しなおさないと修正内容が正しく反映されているか確認がとれない。
開発時もそうだし、バグの検証などでデバックコードを埋め込んだりすることにおいて実行しなおしは効率が下がる要因になりやすい。
そこで、ホットリロードを採用して開発環境において、修正した際に都度再実行（リロード）してもうらようにしてこの問題を解消を目指します。
注意点としては、修正に対して自動で再実行されてしまうので、状態を保持したかった債などは注意が必要になる。特にIDEで自動保存が有効になっていると意図していない状態でリロードされる場合があるので気をつける必要がある。

## airの導入

### 前提

今回は以下の環境が構築済みである前提で話をすすめる。
ホストPCはWindowsだが、Docker環境を用意し、コンテナ内で作成しているGoのアプリケーションを開発を行う。
Docker環境の管理はdocker-ccomposeを使って管理をする。
すでにdocker-compose.ymlが起動確認まで取れている状態のもの状態とします。

### 導入

Dockerfileに以下の内容を追記します。

``` Dockerfile
FROM golang:1.21-alpine

RUN apk update && apk add git

RUN apk --update add tzdata && \
    cp /usr/share/zoneinfo/Asia/Tokyo /etc/localtime && \
    apk del tzdata && \
    rm -rf /var/cache/apk/*

WORKDIR /go/src/app

# ログに出力する時間をJSTにするため、タイムゾーンを設定
ENV TZ /usr/share/zoneinfo/Asia/Tokyo

RUN go install github.com/volatiletech/sqlboiler@latest
RUN go install github.com/volatiletech/sqlboiler/drivers/sqlboiler-mysql@latest
RUN go install -tags mysql github.com/golang-migrate/migrate/v4/cmd/migrate@latest
RUN go install github.com/cosmtrek/air@latest　（追加）
CMD ["air"]（追加）
```

この状態で`docker compose build`を行いコンテナを作り直します。
ビルドができたらコンテナを立ち上げ、Goアプリケーションのコンテナに入ります。

先のDockerfileの編集が問題なければ、airに関するコマンドを扱えるようになっています。
airのイニシャルコマンドを実行します。

``` 
air init
```

実行するとアプリケーションのルートディレクトリに.air.tomlファイルができる。
中身は以下のものになっている。

``` toml
root = "."
testdata_dir = "testdata"
tmp_dir = "tmp"

[build]
  args_bin = []
  bin = "./tmp/main"
  cmd = "go build -o ./tmp/main ."
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor", "testdata"]
  exclude_file = []
  exclude_regex = ["_test.go"]
  exclude_unchanged = false
  follow_symlink = false
  full_bin = ""
  include_dir = []
  include_ext = ["go", "tpl", "tmpl", "html"]
  include_file = []
  kill_delay = "0s"
  log = "build-errors.log"
  poll = false
  poll_interval = 0
  post_cmd = []
  pre_cmd = []
  rerun = false
  rerun_delay = 500
  send_interrupt = false
  stop_on_error = false

[color]
  app = ""
  build = "yellow"
  main = "magenta"
  runner = "green"
  watcher = "cyan"

[log]
  main_only = false
  time = false

[misc]
  clean_on_exit = false

[screen]
  clear_on_rebuild = false
  keep_scroll = true
```

#### tomlファイルの構成

- root
  - アプリケーションルートの指定になります。
- testdate_dir
  - テストデータに関するディレクトリを指定してください。
  - 必須ではないので実在していなくても問題ありません。
- tmp_dir
  - airではビルドしたあとに起動しています。ビルド結果の保管先をしてします。
  - このフォルダはGit管理外にすることをお勧めします。
- build
  - args_bin
    - 実行時の引数指定になります。
  - bin
    - ビルド結果の出力先
  - cmd
    - ビルドコマンドを指定してください。
  - delay
    - ファイル変更から再ビルドまでの遅延時間[ms]になります。
  - exclude_dir
    - 変更検知の除外フォルダ指定になります。
    - 特にビルド結果を置くフォルダは必ず指定しましょう。
  - exclude_file
    - 変更検知の除外ファイル指定になります。
    - 指定がなくても問題ありません。
  - exclude_regex
    - 正規表現での除外ルールになります。
    - デフォルトでテストコードが対象になっています。
  - exclude_unchanged
    - 未変更ファイルの除外設定になります。
    - デフォルトはfalseになっています。
  - follow_symlink
    - ディレクトリのシンボリック有効設定になります。
    - デフォルトはfalseになっています。
  - full_bin
    - 実行時のコマンド指定になります。
  - include_dir
    - 変更検知に含めるディレクトリ指定になります
  - include_ext
    - 変更検知に含める拡張子指定になります。
  - include_file
    - 変更検知に含めるファイル指定になります。
  - kill_delay
    - 実行中のアプリケーションのダウンタイムになります
  - log
    - ビルド結果のログ出力先になります。
  - poll
    - ファイルポーチング設定になります。
    - デフォルトではfsnotifyを使用して変更検知をしています。しかし、Windows環境ではfsnotifyでは変更検知ができないため、MacのようなLinux環境以外では使用する必要があります。
  - poll_interval
    - ポーリング時のインターバル設定になります。
  - post_cmd
    - 実行後の動作確認コマンド指定になります。
  - pre_cmd
    - 実行前の動作確認コマンド指定になります。
  - rerun
    - 再実行の設定になります。
  - rerun_delay
    - 再実行のディレイタイム設定になります。
  - send_interrupt
    - 実行中のプロセスのキル中の割り込み信号の送信設定
    - Windowsでは本機能のサポートをしていないので注意が必要
  - stop_on_error
    - ビルドエラー時に実行中のプロセスを停止する設定になります。
- color
  - アプリケーションのログのカラー設定になります。
  - app = ""
  - build = "yellow"
  - main = "magenta"
  - runner = "green"
  - watcher = "cyan"
- log
  - main_only = false
    - メインのみのログにする設定になります
  - time = false
    - 時刻出力設定になります。
- screen
  - clear_on_rebuild = false
    - リビルト時のクリア設定になります。
  - keep_scroll = true
    - 自動スクロール設定になります。

#### Windows固有の対応

ホストPCがWindowsの場合、初期状態ではホットリロードが効かない（はず）です。
これはホストPCとコンテナ間のボリュームマウントが関係する問題で、airの設定を直す必要があります。
先ほどの説明にも書きましたが、

``` toml
poll=true
```

にしましょう。
Macなら不要です。

＊経験則ですが、Docker Desktopなどでなく、wsl2を使った環境でコンテナ管理を行っている場合は、Linux環境でのボリュームマウントになりますのでこの設定は不要です。

## 最後に

今回はホットリロードを採用するためにairを導入しました。
Golangの開発環境ではわりかし一般的な手法になるとおもいます。
次はこの環境の上にさらにdelveを入れでデバック環境を用意して不具合などの調査がしやすい環境にしたいと考えています。