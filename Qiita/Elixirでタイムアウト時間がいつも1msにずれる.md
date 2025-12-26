はじめに
----
周期毎(250ms)にフリッカーに使うタイマー処理を作ったんだけど、かなり時間がずれてしまう現象が発生。処理は基本tick 10msをカウントし、指定の時間を作るようにしていたので、そのtickが11msなってしまい、モロに時間が伸びてしまう。
ここではこのズレについて説明します。


時間の待ち処理って
----
Process.sleep(tick)を使う方法とreceiveのタイムアウト(after)を使う方法(下記コード)がある。
```
receive do
  _ -> ...
  after tick -> ...
end
```
実際にどんな処理をしているんだろうと、調べたがさっぱり理解できなかった。しかし、ひとつあれっと思った事があった。

> Process.sleep()の処理って、receiveのタイムアウトと同じだった。
```process.ex
@spec sleep(timeout) :: :ok
def sleep(timeout)
    when is_integer(timeout) and timeout >= 0
    when timeout == :infinity do
    receive after: (timeout -> :ok)
end
```
この時間処理について、調査していたコードは
 - erlang/otp/erts/emulator/beam内のtimer.c:   tickのカウントしているらしい
 - 同じpathにあるprocess.c:  receiveの処理をしている
です。

実験 100ms、1000msと時間を延ばしたらどうなる
----
実験コード
```
  def run(tick) do
    sta = System.os_time(:millisecond)
    Process.sleep(tick)
    time = System.os_time(:millisecond) - sta
    IO.puts("time: #{time}")
    run(tick)
  end
```
Process.sleep(tick)の時間を測定する。
測定は`System.os_time()、OS上でのtickカウンターを使用した。

**結果**
100msの時
```
time: 101
time: 101
time: 101
time: 101
time: 101
```
1000msの時
```
time: 1000
time: 1000
time: 1001
time: 1001
time: 1001
```
10000msの場合
```
time: 10001
time: 10002
time: 10001
time: 10001
time: 10001
```
となった。

> 時間延ばしても1msはズレるんだ。

1ms単位でカウントしているはずだから、不思議じゃない
---
1msでカウントしているはずなので、セットするタイミングもあり、1msぐらいずれるかも、と測定を(ms -> μs)にしてみたら、ほとんど、1msズレが500μs以上となっていた。
つまり、11msは10.5、101は100.5、1001は1000.5となっている。

ちょっと待って、カウント値をセットしてから、次の1msのタイミングで一回目と数えていくはずだから、短くなるのが普通じゃないのかなぁ。もしかしたら、1msのタイミングでカウント値をセットしているのかなぁ。だから、長くなるんだろうか。

まぁ、ごちゃごちゃ言ったものの、これって何も問題はなく、気にすぎた。

> 単純にセットするタイミングと数え始めるタイミングのズレだった


まとめ
----
1msズレみたいな些細な事に疑問をもってしまうという癖がでてしまった。組込みでは、いつも時間を気にしていたのでこれは仕方がない。
しかし、1msって、その間にどれだけ処理ができるか、すごく長い時間だよなぁ。距離で考えると、1m(メートル)で動いているものから見たら、1000km(キロ)みたいな感覚だろう。
と、まとめではない話だったが、結論は「数msなんて誤差範囲、気にするな」ってコト。

