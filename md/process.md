# ロボットシステム学第5回

上田 隆一

千葉工業大学

---

## 今日の内容

* プロセス等、Linuxの仕組みについて

---

## プロセス

* プログラム実行の一単位
  * 実行中のプログラム＋OSが準備した付帯情報
  * プロセスID、ユーザ、親プロセス<br />$ $
* 普段はps(1)やtop(1)で調査（次のページ）
  * CPUやメモリ使用量、ゴミプロセスがないか調査、等<br />$ $

---

## <span style="text-transform:none">psとtop</span>

* `$ ps aux`
  * 項目の意味
    * USER: ユーザ, PID: プロセスID, %CPU: CPU 使用率,%MEM: メモリ使用率, VSZ: 仮想メモリのサイズ,RSS: 使用している物理メモリ量, TTY: 端末, STAT: プロセスの状態, START: 起動した時間, TIME: 使ったCPU 時間, COMMAND: コマンド名、カーネルスレッド名
  * STATの意味
    * R: Run, S: Sleep, D: Disk Sleep, T: Stopped, Z: Zombie, +: forground, <: high priority, s: session leader
* `$ top -c`
  * 項目の意味
    * PID: プロセスID, USER: ユーザ, PR: 優先度, NI: nice 値, VIRT: 仮想メモリ使用量, RES: 物理メモリ使用量, SHR: 共有メモリ使用量, S: プロセスの状態,%CPU: CPU 使用率, %MEM: 物理メモリ使用率, TIME+:CPU 使用時間, COMMAND: コマンド

---

## カーネルから見たプロセス

* リソースを割り当てる一単位
  * 番号（プロセス番号、プロセスID、PID）を付けて管理<br />　
* プロセスに対する仕事
  * 生成と消去（メモリの割り当て）
  * CPU の利用時間の管理（タイムシェアリング）
    * 優先度の高低を考慮
      * 端末とのやり取りが多いものほど高頻度
      * <span style="color:red">↑ロボットを動かす時に問題となる</span>
  * ファイルなどのプロセス外のリソースの提供
  * プログラムが自身のプロセス外のメモリを参照しないよう保護
    * 仮想アドレス空間（後述）

---

## プロセスツリー

* プロセスはプロセスから生まれる
  * <span style="color:red">親子関係がある</span>
  * 家系図（プロセスツリー）ができる
    * pstree(1)で調査を
    * systemdあるいはinitというプロセスからぶら下がる<br />$ $

なんでこんな仕組みなのか？

---

## <span style="text-transform:none">fork</span>とプロセスの親子関係

* fork: あるプロセスがふたつに分裂する仕組み
    * あるプロセスがforkすると、プロセス番号を除いて全く同じプロセスがふたつ同時に走るようになる
    *  <span style="color:red">これだけ見るとなんのためにあるのか意味不明</span><br />　
* fork後のプロセス番号
    * 一方は元のプロセス番号（親）
    * もう一方は新たにプロセス番号をもらう（子）<br />　
*  [forkを実行するコード](https://gist.github.com/ryuichiueda/9593919)

---

## <span style="text-transform:none">exec</span>

* プロセスの中身が他のプログラムに変わる仕組み
    * 例: 次のプログラムは`exec sleep 100`の瞬間に`sleep`に化ける

```
#!/bin/bash

echo bashです
sleep 10          #bashがsleepを実行
exec sleep 100    #このプロセス自体がsleepになる
echo これは実行されない
```

これも単体で見ると意義がよく分からない

---

## シェルがコマンドを実行する<br />仕組み

* forkとexecを組み合わせる
    1. シェル自体がforkする
    2. 子のプロセスが`exec`を実行してコマンドに化ける<br />　
* 何が便利か？
  * コマンドの出力が端末に出てくる
    * 端末の入出力の口がforkでコピーされて共有されている
  * 環境が引き継がれる
    * シェルが日本語環境だとコマンドも日本語環境で動く<br />　
* 考慮しておくべき点
    * コマンドを立ち上げるたびにシェルがコピーされる

---

## <span style="text-transform:none">forkとパイプ

* シェルが自分に入出力する口を作ってforkするという処理を繰り返していく
    * →コマンドが数珠つなぎに

<img width="35%" src="./md/images/pipe.png" />

---

## プロセスとメモリ

* プロセスは、基本的に他のプロセスが使っているメモリの中身を見ることができない
  * **見ることができたら事故**
* プロセス間でメモリが見えないようにする仕組み: 仮想記憶
  * 問題: 図のような1列のメモリをどのように複数のプロセスに割り当てる？

<img width="22%" src="./md/images/mem_sequence.png" />

---

## 仮想記憶（ページング方式）

* アドレス空間を二種類用意
  * 物理アドレス空間（DRAMやその他を直接指す）
  * 仮想アドレス空間（プロセスごとに準備）
* アドレス空間を「ページ」に分割
* 仮想のページと物理ページを対応付け

<img width="90%" src="./md/images/page.jpg" />

---

## 仮想記憶で可能となること（1/2）

* fork後も参照しているアドレスが変わらない<br />$ $
* 別のプロセスのメモリ番地が見えない<br />$ $
* lazyな物理メモリ割り当て
  * プログラムが割り当てのないページの番地にアクセスした時に、物理メモリのページを割り当て
  * 割り当てのないページの番地にアクセスすることを「ページフォルト」と言い、これが起こると割り当てが起こる

---

## 仮想記憶で可能となること（2/2）

* スワップ
  * メモリが不足時にページ上のデータをストレージ上のページに追い出せる（スワップアウト）
  * 仮想アドレスの先が物理メモリである必要がなくなる<br />$ $
* キャッシュの管理が簡単に
  * プロセスが使用していない物理メモリのページに読み書きしたファイルのデータを記憶
  * キャッシュが有効だとHDDの読み書き回数を減らすことができる

---

## ここまでのまとめ

大きなシステムを作るときに大いに参考になる

* fork-exec
  * 親のプロセスを丸コピーするという単純で巧妙な方法で環境やリソースの引き継ぎが簡単に<br />　
* 仮想記憶
  * こちらも単純で巧妙な仕組みで効果的なメモリ利用を実現<br />　


以後は雑多な知識です

---

## プロセス番号等の観察

* `$ ps -eo command,pid,ppid,pgid,sid`
  * pid: プロセス番号
  * ppid: 今の親のプロセス番号
    * 親がいなくなると1番にぶら下がる
  * pgid: プロセスグループID: 同じジョブ（後述）の下にいるプロセスが共有するID
  * sid: セッションID: 一つの端末にぶら下がっているプロセスが共有するID<br />$ $
* 参考
    * https://linuxjm.osdn.jp/html/procps/man1/ps.1.html


---

## ジョブ

* シェルがプロセスを管理する塊
* 操作で把握しましょう

```bash
$ sleep 1000000 | cat | cat     #後ろのcatは特に意味はない
Ctrl+Z
 $ sleep 200000 | sleep 200000
Ctrl+Z
$ jobs

[1]-  停止                  sleep 1000000 | cat | cat
[2]+  停止                  sleep 200000 | sleep 200000
$ fg 2      これでjob2がフォアグラウンドに

sleep 200000 | sleep 200000
Ctrl+Z
[2]+  停止                  sleep 2000000 | sleep 2000000
$ kill %1        #job1を殺す

$ jobs

[1]  Terminated              sleep 1000000 | cat | cat
[2]+  停止                  sleep 200000 | sleep 200000
$ bg 2       #job2をバックグラウンド起動

[2]+ sleep 200000 | sleep 200000 &
$ jobs

[2]+  実行中               sleep 200000 | sleep 200000 &
$ fg 2    #job2をフォアグラウンドへ

sleep 200000 | sleep 200000
Ctrl+C
```

---

## シグナル

* プロセス間通信の一種
* あるプロセスから他のプロセスへの「合図」
  * ジョブのコントロールでやったCtrl+ZやCtrl+Cでも送られている
* シグナルの一覧
  * `$ kill -l`
* killコマンドで送ることができる
  * `$ kill -KILL 12345`     #SIGKILL（後述）をPID12345に
  * `$ kill 12345`           #SIGTERMをPID12345に
* 発展
  * `trap`を使うとシェルスクリプト内でシグナルを捕捉して割り込み処理が書ける

---

## 主なシグナル（1/2）

* SIGHUP（1番）
  * HUP: ハングアップ（電話の切断）
  * 使われ方
    * 端末が切れたときに関連するプロセスを止める
      * セッションリーダにSIGHUPが飛ぶ
      * セッションリーダー: セッションIDの持ち主
    * セッションリーダーがいなくなるとカーネルからSIGHUPがセッションのプロセスに送られる
* SIGINT（2 番）
  * INT: interrupt（割り込み）
  * 使われ方
    * 端末でCtrl+c を押したときに端末からセッショングループのフォアグラウンドプロセスに送られる

---

## 主なシグナル（2/2）

* SIGKILL（9 番）
  * プロセスを強制終了するときに使われる
  * プログラム側で後始末できない
  * 後始末はカーネルに任せる<br />$ $
* SIGSEGV（11 番）
  * メモリのセグメンテーションフォルト<br />$ $
* SIGPIPE（13 番）
  * 読み書きしていたパイプの切断<br />$ $
* SIGTERM（15番）
  * 終わってくれてというシグナル。プログラムは速やかに終わらないといけない


---

## プロセスとファイル

* 一つのプロセスについて調査したければ `/proc/<プロセス番号>` を見る
  * プロセス情報もファイルで提供される。なんでもファイル


---

## 終了ステータス

* プロセスが終わるときに親に返す番号
    * C言語やC++でプログラミングするときに`return 0`あるいは`exit(0)`と書いているアレ
    * `echo $?`あるいは（bashなら）`echo ${PIPESTATUS[@]}`で確認<br />　
* 用途
    * シェルスクリプトでの条件分岐
    * ソフトウェアの自動テスト
        * 全テストをパスしたら終了ステータス0を返す→GitHubなどにテスト成功と表示される