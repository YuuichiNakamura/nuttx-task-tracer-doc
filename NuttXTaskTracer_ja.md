NuttX Task Tracer
=================

# 概要

NuttX Task Tracer は、 NuttX カーネル内の各種イベントを収集してそれを出力、グラフィカルに表示させるためのツールです。

以下のようなイベントを収集することができます。

- タスクの起動、終了、切り替え
- システムコールの発行、終了
- 割り込みハンドラの実行、終了
- ユーザプログラムに埋め込んだユーザ定義メッセージ

# 準備

## Trace Compass のインストール

Task Tracer では、トレース結果の表示のために Trace Compass という外部ツールを使用します。
https://www.eclipse.org/tracecompass/ よりダウンロードして、ホスト環境にインストールしてください。

インストール後、起動して Tools -> add-ons メニューを選択し、 Install Extensions で "Trace Compass ftrace (Incubation)" という機能拡張をインストールしてください。

## NuttX カーネルのコンフィグレーション

機能を有効にするためには、NuttX カーネルのコンフィグレーションを変更する必要があります。
menuconfig で設定を変更する場合には、以下のオプションを有効にします。

- `RTOS Features  --->`
  - `Performance Monitoring  --->`
    - `System performance monitor hooks` にチェック `[*]` を入れる
    - `System call monitor hooks` にチェック `[*]` を入れる
    - `Interrupt handler monitor hooks` にチェック `[*]` を入れる
    - `Scheduler tracer` にチェック `[*]` を入れる
    - `Task trace buffer size` には適当なバッファサイズをバイト単位で指定する
    - `Task trace name buffer size` にはタスク名保存用バッファのサイズを指定する
      - (通常はデフォルトの 128 で問題ないです)
- `Device Drivers  --->`
  - `System Logging  --->`
    - `Scheduler tracer driver` にチェック `[*]` を入れる

コンフィグレーション後、カーネルとアプリケーションの両方をビルドします。

カーネルのトレース機能が有効になっていると、NuttShell で "trace" コマンドが利用可能になります。

# 計測方法

NuttShell の trace コマンドによってトレース機能の制御を行います。

## Quick Guide

```
nsh> trace start
```
で、トレースを開始します。

```
nsh> trace stop
```
で、トレースを停止します。

```
nsh> trace dump
```
で、現在蓄積されているトレースデータをコンソールに表示します。
ターミナルソフトのログ機能でファイルに保存することで、Trace Compass への入力として使用することができます。

ターゲットがストレージを持っていて、USB MSC などによってホストへのファイル転送が可能な場合には、
```
nsh> trace dump <ファイル名>
```
によって、トレースデータをファイルに保存することができます。

## trace コマンド詳細

### **trace start**
トレースを開始します
```
trace start [<duration>]
```
- `<duration>` : トレースする期間を秒数で指定します。指定の秒数の経過後にトレースを停止します。指定しなかった場合には、明示的に停止しない限りトレースを継続します。

### **trace stop**
トレースを停止します
```
trace stop
```

### **trace dump**
トレース結果をファイルに保存します。
トレース実行中だった場合には停止してから行います
```
trace dump [<filename>]
```
- `<filename>` : トレース結果を保存するファイル名を指定します。指定しなかった場合は、トレース結果がコンソールに表示されます。

### **trace cmd**
トレースを開始してコマンドを実行し、実行後にトレースを停止します
```
trace cmd "<command>"
```
- `<command>` : nsh のコマンドライン文字列を指定します。複数のパラメータがある場合には引用符で囲む必要があります。

実行例:
```
nsh> trace cmd "sleep 1"
```

### **trace mode**
トレース対象イベントを指定するなどのモード設定を行います
```
trace mode [{+|-}{o|s|a|i}...]
```
- `+o` : Oneshot 機能を有効にします。
トレースバッファはリングバッファになっていて、通常はバッファがいっぱいになると古いデータを上書きしてトレースを継続しますが、Oneshot が有効だとバッファがいっぱいになった時点でトレースを停止します。アプリ実行の起動時のトレースを取りたい場合などに使用します。

- `-o` : Oneshot 機能を無効にします。

- `+s` : システムコールトレース機能を有効にします。
アプリケーションの発行するシステムコールの開始と終了のイベントが記録されます。
デフォルトではすべてのシステムコールが記録の対象となりますが、`trace syscall` コマンドで記録するシステムコールをフィルタすることができます。

- `-s` : システムコールトレース機能を無効にします。

- `+a` : システムコールトレースでの引数取得を有効にします。
システムコール呼び出しで渡された引数をトレースデータに保存するようになります。

- `-a` : システムコールトレースでの引数取得を無効にします。

- `+i` : 割り込みトレース機能を有効にします。
トレース期間中に発生した割り込みハンドラの実行開始と終了のイベントが記録されます。
デフォルトではすべての割り込みが記録の対象となりますが、`trace irq` コマンドで記録する割り込みをフィルタすることができます。

- `-i` : 割り込みトレース機能を無効にします。

パラメータを省略した場合は、現在のモードを以下のように表示します。

実行例:
```
nsh> trace mode
Task trace mode:
 Oneshot                 : off (-o)
 Syscall trace           : on  (+s)
  Filtered Syscalls      : 16
 Syscall trace with args : on  (+a)
 IRQ trace               : on  (+i)
  Filtered IRQs          : 2
```

### **trace syscall**
システムコールトレース機能のフィルタを設定します
```
trace syscall [{+|-}<syscallname>...]
```
- `+<syscallname>` : 指定した名前のシステムコールをフィルタの対象にします。
フィルタされたシステムコールの実行はトレースデータに記録されなくなります。<p>
システムコール名の指定には、ワイルドカード `"*"` を使用することができます。
例えば、`+sem_*` と指定すると、`sem_post()` や `sem_wait()` など、`sem_` で始まるシステムコールすべてがフィルタの対象となります。

- `-<syscallname>` : 指定した名前のシステムコールをフィルタの対象から外します。
システムコール名の指定は、同様にワイルドカード `"*"` を使用することができます。

NuttX のタスク実行中には `getpid()` やセマフォの取得・解放等が頻繁に実行されるため、
システムコールトレースを行う際には
```
trace syscall +getpid +sched_* +sem_* +mq_*
```
を指定しておくのがおすすめです。

パラメータを省略した場合は、現在のフィルタ内容を以下のように表示します。

実行例:
```
nsh> trace syscall
Filtered Syscalls: 16
  getpid
  sem_destroy
  sem_post
  sem_timedwait
  sem_trywait
  sem_wait
  mq_close
  mq_getattr
  mq_notify
  mq_open
  mq_receive
  mq_send
  mq_setattr
  mq_timedreceive
  mq_timedsend
  mq_unlink
```

### **trace irq**
割り込みトレース機能のフィルタを設定します
```
trace irq [{+|-}<irqnum>...]
```
- `+<irqnum>` : 指定した番号の割り込みをフィルタの対象にします。
フィルタされた割り込みハンドラの実行はトレースデータに記録されなくなります。

- `-<irqnum>` : 指定した番号の割り込みをフィルタの対象から外します。

SPRESENSE では IRQ 11 の SVC Call 割り込み、IRQ 15 の SysTick 割り込みが頻繁に発生するため、割り込みトレースを行う際には
```
trace irq +11 +15
```
を指定しておくのがおすすめです。

パラメータを省略した場合は、現在のフィルタ内容を以下のように表示します。

実行例:
```
nsh> trace irq
Filtered IRQs: 2
  11
  15
```

### **ユーザメッセージの記録**
`/dev/tracer/` への書き込みを行うことで、トレースデータにユーザメッセージを記録することができます。

実行例:
```
nsh> echo "message" >/dev/tracer
```

# 計測結果の表示

計測結果をターミナルのログやファイル転送などでホストに転送すると、Trace Compass でその結果を表示することができます。

Trace Compass を起動し、File -> Open Trace でトレースデータのファイル名を指定します。

# トレース制御 API

nsh の trace コマンド以外にも、アプリケーションから API によって直接トレース機能を制御することが可能です。

API を使用する際は、以下のインクルードファイルを include してください。
```
#include <nuttx/sched_tracer.h>
```

## API 詳細

### **sched_tracer_start**
トレースを開始します
```
#include <nuttx/sched_tracer.h>

void sched_tracer_start(void);
```
#### 引数
- なし

#### 戻り値
- なし

### **sched_tracer_stop**
トレースを停止します
```
#include <nuttx/sched_tracer.h>

void sched_tracer_stop(void);
```
#### 引数
- なし

#### 戻り値
- なし

### **sched_tracer_mode**
トレースモードの設定・取得を行います
```
#include <nuttx/sched_tracer.h>

struct tracer_mode_s *sched_tracer_mode(struct tracer_mode_s *newm);
```

#### 引数
- `newm` : トレースモードを指定する `struct tracer_mode_s` 構造体へのポインタ
```
struct tracer_mode_s
{
  unsigned int flag;

#define TRIOC_MODE_FLAG_ONESHOT      (1 << 0)
#define TRIOC_MODE_FLAG_SYSCALL      (1 << 1)
#define TRIOC_MODE_FLAG_SYSCALL_ARGS (1 << 2)
#define TRIOC_MODE_FLAG_IRQ          (1 << 3)
};
```

#### 戻り値
- 現在の設定値を格納した `struct tracer_mode_s` 構造体へのポインタ

### **sched_tracer_syscallfilter**
システムコールトレース機能のフィルタの設定・取得を行います
```
#include <nuttx/sched_tracer.h>

ssize_t sched_tracer_syscallfilter(struct tracer_syscallfilter_s *oldf,
                                   struct tracer_syscallfilter_s *newf);
```
#### 引数
- `oldf` : 取得する設定値を保存する struct tracer_syscall_s 構造体 へのポインタ。
NULL を与えた場合は取得を行いません。
- `newf` : 設定値を格納した struct tracer_syscall_s 構造体 へのポインタ。
NULL を与えた場合は設定を変更しません。

```
struct tracer_syscallfilter_s
{
  int nr_syscalls;
  int syscall[0];
};
```

#### 戻り値
- 設定値を取得するために値の保存に必要な構造体のサイズを返します。
フィルタに設定されているシステムコールの個数によって必要なサイズが変わるため、設定値を取得するには以下の手順で行ってください。
  1. oldf に NULL を指定して API を呼び出し、必要サイズを取得する
  2. 必要サイズ分の領域を確保する
  3. 確保した領域を oldf に渡して API を呼び出す

### **sched_tracer_irqfilter**
割り込みトレース機能のフィルタの設定・取得を行います
```
#include <nuttx/sched_tracer.h>

ssize_t sched_tracer_irqfilter(struct tracer_irqfilter_s *oldf,
                               struct tracer_irqfilter_s *newf);
```
#### 引数
- `oldf` : 取得する設定値を保存する struct tracer_irq_s 構造体 へのポインタ。
NULL を与えた場合は取得を行いません。
- `newf` : 設定値を格納した struct tracer_irq_s 構造体 へのポインタ。
NULL を与えた場合は設定を変更しません。

```
struct tracer_irqfilter_s
{
  int nr_irqs;
  int irq[0];
};
```

#### 戻り値
- 設定値を取得するために値の保存に必要な構造体のサイズを返します。
フィルタに設定されている IRQ 番号の個数によって必要なサイズが変わるため、設定値を取得するには以下の手順で行ってください。
  1. oldf に NULL を指定して API を呼び出し、必要サイズを取得する
  2. 必要サイズ分の領域を確保する
  3. 確保した領域を oldf に渡して API を呼び出す

### **sched_tracer_message**
トレースデータにユーザ定義メッセージを記録します。
アプリケーション中にこの API を実行することで、トレースデータからこの部分を実行したことを知ることができます。
```
#include <nuttx/sched_tracer.h>

void sched_tracer_message(FAR const char *fmt, ...);
```
#### 引数
- `fmt` : printf() 形式のフォーマット文字列。
後続して引数を指定することで、引数の内容が printf() 形式で出力されます。

#### 戻り値
- なし
