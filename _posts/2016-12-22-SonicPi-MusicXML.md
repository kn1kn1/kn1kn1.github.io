---
layout: post
title:  Sonic Pi MusicXML Player
---

Sonic PiでMusicXMLプレーヤーを作ってみたので、そのお話をしたいと思います。

## MusicXML

MusicXMLについては、[roombaさんの記事](http://roomba.hatenablog.com/entry/2016/02/03/150354)がとても分かりやすかったので、そちらをご参照ください。

MusicXMLの内容をSonic Piで演奏するという試みは、[こちら](https://rbnrpi.wordpress.com/2016/08/19/converting-musicxml-or-midi-files-to-work-with-sonic-pi/)で既に行われているのですが、外部プログラム(Processing)が必要だったりしたので、本記事ではrubyのみで実装したいと思います。

## 実装

出来上がったコードは、↓にあります。

[https://github.com/kn1kn1/sonic-pi-musicxml-player/blob/master/musicxml-player.rb](https://github.com/kn1kn1/sonic-pi-musicxml-player/blob/master/musicxml-player.rb)

おおまかな流れとしては、

1. 入力されたMusicXMLを解析してScoreモデルを生成
1. Scoreモデルの内容をもとにSonic Piで演奏

という感じになっています。

### XMLパーサーについて

rubyでXMLを扱うにはいくつか方法があるようですが、今回は標準ライブラリのrexmlを使います。ソースコードの冒頭に以下を追加するだけで使用可能です。

```
require 'rexml/document'
```

rexmlにはDOMスタイルとSAXスタイルのパーサーがあるのですが、今回はパフォーマンス的にそれほどシビアではないので、DOMパーサーを使います。

```ruby
doc = REXML::Document.new(open(File.expand_path(path)))
doc.elements.each('score-partwise/part') do |e|
  : (snip)
```

といったように、XPath指定での要素の取得が可能です。

### モデルについて

モデルは、MusicXMLの構造に準拠してScore(楽譜), Part(パート), Measure(小節), Note(音符)をそれぞれクラスとして定義しました。

「ScoreはいくつかのPartを持ち、PartはいくつかのMeasureを持ち、MeasureはいくつかのNoteを持つ」という構造になっています。

Measureはこの実装の中核となるクラスで、notes_table(どのステップにどの音符・休符を持つかのテーブル)を持つ他に、steps(「音符・休符の最小長さ」をいくつ持っているか)や、step_dur(「音符・休符の最小長さ」の持続時間)といった情報を持たせています。

最終的にSonic Piで演奏する箇所は以下のようにしています。まず楽譜内のパート毎にスレッドを起動し、小節をイテレートして行きます。そして小節内では、ステップ毎に音符があるかどうかチェックして、音符がある場合にはスレッドを起動して`play`で音符を演奏しています。そして、音符・休符の有無に関わらずstep_dur(「音符・休符の最小長さ」の持続時間)だけ`sleep`しています。

```ruby
score.parts.each do |part|
  in_thread do
    part.measures.each do |measure|
      measure.steps.times do |step|
        notes = measure.notes_table[step]
        if notes
          notes.each do |n|
            in_thread do
              play n.notes, release: n.duration
            end
          end
        end
        sleep measure.step_dur
      end
    end
  end
end
```

### Measureモデルについて

XMLを解析してMeasureモデルを作るところがなかなか厄介だったので、少々解説します。

基本的には、[roombaさんの記事](http://roomba.hatenablog.com/entry/2016/02/03/150354#%E3%83%91%E3%83%BC%E3%83%88)と同じように、measure要素内のattributes要素の内容やnote要素を解析しています。また凶悪なbackup要素についても実装しています。

まずコンストラクタですが、これは3つの引数(first, previous, element)を取っています。それぞれ、「最初のパートの同じ小節」、「同じパートの直前の小節」、「該当の小節の要素」です。何故「最初のパートの同じ小節」や「同じパートの直前の小節」が必要かと言うと、楽譜によっては、measure要素に明示的にbpmやbeats(4/4拍子の分子), beat-type(4/4拍子の分母)などが指定されていない場合があるからです。コンストラクタの前半部分でこれらの値を引き継いでおいてから、該当の小節の要素をparseメソッドで解析してメンバ変数へ設定するようにしています。

```ruby
def initialize(first, previous, element)
  @notes_table = {}

  if first
    # 最初のパートの同じ小節のプロパティを引き継ぐ
    @bpm = first.bpm
    @divisions = first.divisions
    @beats = first.beats
    @beat_type = first.beat_type
    @steps = first.steps
    @step_dur = first.step_dur
    @bar_dur = first.bar_dur
  elsif previous
    # 同じパートの直前の小節のプロパティを引き継ぐ
    @bpm = previous.bpm
    @divisions = previous.divisions
    @beats = previous.beats
    @beat_type = previous.beat_type
    @steps = previous.steps
    @step_dur = previous.step_dur
    @bar_dur = previous.bar_dur
  else
    # default values
    @bpm = 120
    @divisions = 1
    @beats = 4
    @beat_type = 4
    @steps = 4
    @step_dur = 0.5
    @bar_dur = 2
  end

  parse(element)
end
```

続いてparseメソッドでは、bpm, step_dur, stepsを設定した後、note要素とbackup要素を順に処理して、current_stepをキーにしてNoteオブジェクトをnotes_tableに格納しています。

```ruby
def parse(element)
  bpm = Measure.extract_bpm(element)
  @bpm = bpm if bpm

  : (snip)

  @step_dur = (60.0 / @bpm) * (4.0 / @beat_type) / @divisions
  @steps = (@beats * @divisions).to_i

  current_step = 0
  note = nil
  element.elements.each do |nb|
    # puts nb
    if nb.name == 'note'
      : (snip)

      note = Note.new
      note.notes.push(note_sym)

      : (snip)

      add_to_notes_table(current_step, note)
      current_step += duration.text.to_i
    elsif nb.name == 'backup'
      : (snip)
      current_step -= duration.text.to_i
    end
  end
end
```

backup要素が出現した場合には、current_stepを巻き戻す処理を入れています。

こうして出来上がったnotes_tableは以下のように、{ステップ=>Noteオブジェクトの配列}のレコードが格納されたものになっています。これにより、再生の際にステップをキーにしてNoteオブジェクトの配列を取り出して、`play`メソッドを呼べるようにしています。

```
{0=>[#<Note:0x007f8f44124640 @notes=[:A4], @duration=0.5>],
 1=>[#<Note:0x007f8f43834dc8 @notes=[:A4], @duration=0.5>],
 2=>[#<Note:0x007f8f4408f298 @notes=[:B4], @duration=1.0>]}
```

### その他

XMLパーサーところで「パフォーマンス的にそれほどシビアではない」と書いたのですが、その辺りの話を。

Sonic Piは演奏が時間通りに行われているか常にチェックしていて、ある出音が極端に遅いと例外を吐いてストップしてしまいます。

今回のプログラムでも、XMLの解析に時間が掛かるので、最初の1音目の出音で例外が出てしまいます。これを回避するために、音を出す前に`set_sched_ahead_time!`を呼び出しています。

`set_sched_ahead_time!`は、(内部関数っぽい見た目に反して)Sonic Piの標準関数として定義されていて、ヘルプの記載もあります。

> Specify how many seconds ahead of time the synths should be triggered. This represents the amount of time between pressing 'Run' and hearing audio. A larger time gives the system more room to work with and can reduce performance issues in playing fast sections on slower platforms. However, a larger time also increases latency between modifying code and hearing the result whilst live coding.

これにより、XMLの解析に掛かった時間を差し引いてSonic Piに実行させるようにしています。

## 最後に

お約束のジングルベルをお聞きください。

sonic-pi-musicxml-playerプロジェクトをホームディレクトリにgit cloneし、

```shell
$ git clone https://github.com/kn1kn1/sonic-pi-musicxml-player.git
```

musicxml-player.rbの中身をSonic PiのBufferにコピーして`Run`を叩いてみてください。

↓のような感じで再生されると思います。

<iframe src="https://player.vimeo.com/video/194818681" width="640" height="359" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/194818681">sonic-pi-musicxml-player</a> from <a href="https://vimeo.com/user1962356">kn1kn1</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

上手くファイルが読み込めないなどの場合は、

[https://github.com/kn1kn1/sonic-pi-musicxml-player/blob/master/musicxml-player.rb#L212](https://github.com/kn1kn1/sonic-pi-musicxml-player/blob/master/musicxml-player.rb#L212)

```ruby
play_musicxml('~/sonic-pi-musicxml-player/lg-641011115129979680.xml')
```

あたりがファイルを指定している部分なので、ここを変更してみてください。

MusicXMLファイルは、[https://musescore.com/user/16083/scores/27630](https://musescore.com/user/16083/scores/27630) から持ってきています。

[https://musescore.com/](https://musescore.com/) にはたくさんの楽譜が公開されているので、色々とダウンロードして試してみると面白いかもしれません。

では！
