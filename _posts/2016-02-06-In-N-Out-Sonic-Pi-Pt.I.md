---
layout: post
title:  In 'N Out Sonic Pi - Pt.I
date:   2016-02-06 21:47:04 +0900
---
「これなら私にもできる。久々に音楽を作ってみよう！」と昨年から始めたSonic Piであるが、昨年末に[翻訳で貢献したり](https://github.com/samaaron/sonic-pi/commits?author=kn1kn1)、それからまだ日の目を見ていないけれど[こちら](http://www.sapporo-internationalartfestival.jp/siaflab/sonic-jam-pi/)をお手伝いしたりしていて、順調に音作りから外れてしまったので、「音作りでない部分のSonic Pi」について書いてみたい。

## 全体の構造
ユーザの触る部分のGUIはQtで作られていて、Sonic Pi Serverとも言うべき部分はrubyで書かれている。

GUI (Qt) <-- OSC --> Sonic Pi Server (ruby) <-- OSC --> scsynth (SuperCollider Server)

GUIとSonic Pi ServerはOSCでやりとりをしていて、Sonic Pi ServerはUDP4557番を監視している。

このあたりの話は、つい先日Joe Armstrongが丁寧に書いているとおりである。
https://joearms.github.io/2016/01/29/Controlling-Sound-with-OSC-Messages.html

## コードの実行
上のJoe Armstrongの記事にもあるように、コードを実行させるには、GUI -> Sonic Pi Serverで"/run_code"コマンドを送信する。

```
"/run_code", gui_id, code
```

正確には、我々がCmd-rで「Run」するときには、"/run_code"ではなく"/save-and-run-buffer"を送信しているのであるが、いずれにしても実際にコードを実行しているのは、SonicPi::RuntimeMethods#__spider_eval の以下の箇所である。

[runtime.rb#L687](https://github.com/samaaron/sonic-pi/blob/v2.9.0/app/server/sonicpi/lib/sonicpi/runtime.rb#L687)

```ruby
eval(code, nil, info[:workspace], firstline)
```

[rubyの組込みメソッドeval](http://docs.ruby-lang.org/ja/2.1.0/method/Kernel/m/eval.html)を使用している。第2引数のbinding(評価コンテキスト)がnilなので、SonicPi::RuntimeMethodsと同じコンテキストで実行されている。

## Sonic Piの拡張

ここまでで見たように、バッファに書いて実行されるSonic Piのコードはrubyのevalで実行されるので、基本的にはrubyで書かれたものは何でも実行可能である。

rubyには[loadメソッド](http://docs.ruby-lang.org/ja/2.1.0/class/Kernel.html#M_LOAD)というものがあり、任意のrubyファイルをロードして実行可能である。

このloadメソッドを使って、任意のメソッドを定義したrbファイルを読み込んで実行してもらおうというのがここでの趣旨である。では早速Hello Worldを見てみよう。

### Hello World
以下のようなrbファイルを作成して、ホームディレクトリに保存する。

sp-hello.rb

```ruby
def hello_sp
  puts "Hello Sonic Pi!"
end
```

Sonic Piを起動し、バッファに以下のコードを入力する。

```ruby
load "~/sp-hello.rb"

hello_sp
```

「Run」を実行すると、ログに"Hello Sonic Pi!"が表示されるはずである。

```
=> Starting run 2

{run: 2, time: 0.0}
 └─ Hello Sonic Pi!

=> Completed run 2
```

### バッファ停止を実装してみる

もう少し複雑な例として「バッファ停止」を実装してみたので紹介したい。Sonic Piで複数のバッファのコードを同時に実行していると、ある特定のバッファのコードのみ停止したいと思ったことはないだろうか？今回実装したのは、停止したいバッファの冒頭に以下を追記して再度「Run」を実行すると、バッファで実行されている処理が止まるというものである。他のバッファで実行されている処理は継続したままである。

```
load "~/sp-stop-util.rb"
stop_current_buffer
```

ソースコードは以下のとおりである。スレッドに設定されているワークスペース名(スレッド変数:sonic_pi_spider_job_infoのハッシュに:workspaceで格納されている)をチェックして、カレントスレッドと同じであればスレッドを停止して、最後にカレントスレッドを停止するということをやっている。

SonicPi::RuntimeMethodsと同じコンテキストで実行されるので、require無しでruntime.rbやcore.rbを参照可能である。と同時に、Sonic Piとして公開したつもりでない内部変数や内部メソッドにかなり依存してしまっているので、その点だけは注意が必要である。

[sp-stop-util.rb](https://gist.github.com/kn1kn1/564202800684e362be36)

```ruby
# Stops all threads in the current buffer.
# @example
#   load "~/sp-stop-util.rb"
#   stop_current_buffer
# @note This method is using the Sonic Pi's internal methods and variables.
#   SonicPi::RuntimeMethods#__current_job_info, __stop_job, __current_job_id
#   SonicPi::Runtime#user_jobs, job_subthreads
def stop_current_buffer
  current_job_info = __current_job_info
  current_workspace_name = current_job_info[:workspace] || ''
  puts "current workspace name: #{current_workspace_name}"

  # find sub threads job id
  current_buffer_job_ids = Set.new
  @user_jobs.each_id do |job_id|
    puts "job_id: #{job_id}"
    subthreads = @job_subthreads[job_id]
    next unless subthreads

    subthreads.each do |st|
      info = st.thread_variable_get :sonic_pi_spider_job_info || {}
      puts "info: #{info}"
      workspace = info[:workspace]
      puts "workspace name: #{workspace}"
      current_buffer_job_ids.add job_id if workspace == current_workspace_name
    end
  end

  # kill sub threads
  current_buffer_job_ids.each do |job_id|
    __stop_job job_id
  end

  # kill current thread at last
  __stop_job __current_job_id
end
```

こんな感じでrbファイルを公開することで、「他の人にもこのメソッドを使ってもらえるようにした」=「外側から機能を拡張できた」と言えるのではないだろうか。

今回はとりあえず[gist](https://gist.github.com/kn1kn1/564202800684e362be36)に公開したが、いくつか便利な機能が揃ってきたらgithubにプロジェクトを公開したいと思っている。

上記以外のトピックスとしてlive_loopについても書きたかったけど、長くなったので[Pt.II]({{site.baseurl}}/2016/02/06/In-N-Out-Sonic-Pi-Pt.II.html)に続く。
