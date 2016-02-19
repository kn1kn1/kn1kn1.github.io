---
layout: post
title:  In 'N Out Sonic Pi - Pt.II
date:   2016-02-06 21:51:04 +0900
---
[Pt.I]({{site.baseurl}}/2016/02/06/In-N-Out-Sonic-Pi-Pt.I.html)からの続きですが、live_loopだけ知りたい人はPt.I読まずにここから読んでもたぶん理解できると思います。

## live_loop

live_loopについては、[Sam Aaron自身が2015年のStrange Loopで解説している](https://www.youtube.com/watch?v=YlRTTzlhquo)。最初から見るのは辛いという方は、[25分あたり](https://youtu.be/YlRTTzlhquo?t=1493)から見ると良いだろう。

ここでやっているのは、:fooというメソッドと:mainという名前のin_threadを定義して実行しておいた状態で、

```ruby
define :foo do
  sample :bd_haus
  sleep 0.5
end

in_thread(name: :main) do
  loop do
    foo
  end
end
```

:fooの中身を書き換えて再実行するというものである。

```ruby
define :foo do
  # sample :bd_haus
  sample :ambi_choir
  sleep 0.5
end

in_thread(name: :main) do
  loop do
    foo
  end
end
```

実際にSonic Piでやってみると、ビートを止めずにサンプリング音が変わってくれるだろう。

きちんとした説明や理論的な背景は、[Temporal semantics for a live coding languageという論文](https://www.cl.cam.ac.uk/~dao29/publ/farm14-aaron.pdf)にあるらしい(読んでない)のだが、大雑把に言うとlive_loopは「in_threadとdefineの組合せのシンタックス・シュガー」ということになる。

では実際にcore.rbに書かれているlive_loopの中身を見てみよう。

[https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L733-L787](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L733-L787)

以下の部分で、live_loopで渡されたblockの処理をdefineし、

[core.rb#L752-L763](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L752-L763)

```ruby
case block.arity
when 0
  define(ll_name) do |a|
    block.call
  end
when 1
  define(ll_name) do |a|
    block.call(a)
  end
else
  raise "Live loop block must only accept 0 or 1 args"
end
```

次の部分で、名前付きin_threadを起動し、

[core.rb#L765](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L765)

```ruby
in_thread(name: ll_name, delay: delay, sync: sync_sym) do
```

in_threadのループ内で、[rubyのsend](http://docs.ruby-lang.org/ja/2.1.0/method/Object/i/__send__.html)を使用してdefineしたメソッドを呼んでいる。

[core.rb#L777](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L777)

```ruby
res = send(ll_name, res)
```

## 名前付きin_thread

とりあえずここまでは理解したけど、in_threadのブロックはどうやって変えてるの？先に実行されているスレッドの停止はどうやって待っているの？など疑問が尽きないのでin_threadの中身を見てみた。

[https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L2521](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L2521)

実行中のスレッドを停止したり、joinしたりという処理は見つけられず、？？となったが、そういった処理は実際にはやっていなかった。

順に見ていこう。

まず、in_threadの以下の箇所で新しいスレッドを作成して実行している。

[core.rb#L2557](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L2557)

```ruby
t = Thread.new do
```

その中で、上で作成・実行したスレッドを第二引数にして、job_subthread_addメソッドを呼んでいる。

[core.rb#L2616](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L2616)

```ruby
job_subthread_add(job_id, Thread.current, name)
```

job_subthread_add→job_subthread_add_unmutexedと進んで行き、最終的に以下の箇所で、「既に同じ名前のスレッドがある場合には、引数として指定されたスレッドを実行せずにそのままkillする」ということをやっている。つまり、上で作成・実行したスレッドはここで停止されてしまうということである。

[runtime.rb#L770-L777](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/runtime.rb#L770-L777)

```ruby
if name
  if @named_subthreads[name]
    #Don't delay following message, as this method is used for worker thread impl.
    __info "Thread #{name.inspect} exists: skipping creation"

    t.kill
    job_subthread_rm_unmutexed(job_id, t)
    return false
  else
```

Sonic Piで以下のコードを実行してみよう。ログに"foo"という文字列が出力され続けるはずである。

```ruby
in_thread(name: :main) do
  loop do
    puts "foo"
    sleep 0.5
  end
end
```

続いて、以下のように:mainを変更して再実行してみよう。

```ruby
in_thread(name: :main) do
  loop do
    # puts "foo"
    puts "bar"
    sleep 0.5
  end
end
```

ログには、"bar"の文字列が表示されるのではなく、"Thread :main exists: skipping creation"が表示された後、"foo"の文字列が表示され続けるだろう。

```
=> Starting run 3

=> Thread :main exists: skipping creation

=> Completed run 3

{run: 2, time: 5.0, thread: "main"}
 └─ foo

{run: 2, time: 5.5, thread: "main"}
 └─ foo
```

若干不可解かもしれないが、再実行の際にin_threadのブロックを変更することもないし、先に実行されているスレッドを止めることもないということである。

ソースコードをよくよく読んでいくと、実はin_threadのヘルプドキュメントに記載があった。"the second re-run will not create a new similarly named thread. "ということだそうである。

[core.rb#L2748-L2764](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L2748-L2764)

```ruby
# Named threads work well with functions for live coding:
define :foo do  # Create a function foo
 play 50       # which does something simple
 sleep 1       # and sleeps for some time
end
in_thread(name: :main) do  # Create a named thread
 loop do                  # which loops forever
   foo                    # calling our function
 end
end
# We are now free to modify the contents of :foo and re-run the entire buffer.
# We'll hear the effect immediately without having to stop and re-start the code.
# This is because our fn has been redefined, (which our thread will pick up) and
# due to the thread being named, the second re-run will not create a new similarly
# named thread. This is a nice pattern for live coding and is the basis of live_loop.
```

ちなみに名前無しin_threadの場合は、再実行の度に次々と新しいスレッドが生成され、元のスレッドもそのまま実行される。いずれにせよ既に実行されているin_threadのブロックの内容(スレッドの内容)を変更するということはやっていない。仕様としては我々の感覚に反するものかもしれないが、スレッドをイミュータブルにする(中身を変更しない)というのは複雑さを回避するためのベストプラクティスなのかもしれない。

## stop

最後にv2.6から導入されたstopを見てみよう。

[core.rb#L264-L268](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/lang/core.rb#L264-L268)

```ruby
def stop
  # Schedule messages
  __schedule_delayed_blocks_and_messages!
  raise SonicPi::Stop
end
```

SonicPi::RuntimeMethods#__stop_jobで停止していると思いきや、[SonicPi::Stopという例外](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/runtime.rb#L44)を投げているだけだった。

私がv2.6以前に手動でやっていたのと大して変わらなかった。
https://twitter.com/kn1kn1/status/626900980085321728

```
kn1kn1
‏@kn1kn1

2.6でlive_loopをstop出来るようになったのが良いな。前はわざとruntime error出して止めてた。
16:43 - 2015年7月30日
```

例外に関して言うと、昔々「[例外状況である場合のみ例外を使用する](https://books.google.co.jp/books?id=ka2VUBqHiWkC&lpg=PP1&dq=effective%20java&hl=ja&pg=PA241#v=onepage&q&f=false)」と習ったのだが今は違うのかもしれない。

というわけでPt.2でこのシリーズは一旦終了です。次やるとしたらcueとsyncあたりかな。
