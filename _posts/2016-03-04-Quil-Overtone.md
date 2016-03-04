---
layout: post
title:  Quil + Overtone
---
先日[Sapporo.clj#9 : ATND](https://atnd.org/events/74457)に参加して、Quilを試してみました。

[Quil](http://quil.info/)はClojureでProcessingを使えるようにしたライブラリで、ClojureでもClojureScriptでも使用することができます。Clojure版はREPLを使うことで動的にスケッチの内容を変更できる点が本家のProcessingに無い大きな特徴でしょう。

Quilの詳細は他にも説明があるようなのでとりあえず置いておいて、今回は音との連携について書いてみたいと思います。

### Overtoneを使って音量を取得する

音量を取得する処理は[Shadertone](https://github.com/overtone/shadertone)に実装があります(Shadertoneはここから更にglslに値を渡す仕組みが用意されています)。以下の箇所ではOvertoneで生成した音のモニター出力を取得しています。

[shadertone/tone.clj#L15-L23](https://github.com/overtone/shadertone/blob/v0.2.5/src/shadertone/tone.clj#L15-L23)

```clojure
;; ----------------------------------------------------------------------
;; Tap into the Overtone output volume and send it to iOvertoneVolume
;; volume tap synth inspired by
;;   https://github.com/samaaron/arnold/blob/master/src/arnold/voltap.clj
(defsynth vol []
  (tap "system-vol" 60 (lag (abs (in:ar 0)) 0.1)))

(defonce voltap-synth
  (vol [:after (foundation-monitor-group)]))
```

このままだとOvertoneで生成した音だけしか取得出来ないので、sound-inを使用してマイクからの入力を取得するようにしたのと、バンドパスフィルターを通すようにしたのが次のコードです。

```clojure
(defsynth sound-in-vol-tap [freq 440.0 rq 1.0]
  (tap "sound-in-vol" 60 (lag (abs (bpf (sound-in:ar 0) freq rq)) 0.1)))
(defonce sound-in-synth440 (sound-in-vol-tap 440))
(defn sound-in-vol
  ([] @(get-in sound-in-synth440 [:taps "sound-in-vol"]))
  ([in-synth] @(get-in in-synth [:taps "sound-in-vol"])))
```

### Quilスケッチ側の実装

Quilのスケッチでは、updateの度に上記の関数を呼んで値を取得します。

```clojure
(defn update-state [state]
  (let
    [vol (voltap/sound-in-vol)]

(..snip..)
```

### 実行例

lein new quilで作成したプロジェクトから弄ったのがこちらです。

<iframe src="https://player.vimeo.com/video/157720429" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/157720429">quil-voltap</a> from <a href="https://vimeo.com/user1962356">kn1kn1</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

コードは[こちら](https://github.com/kn1kn1/quil-voltap)にあります。

[https://github.com/kn1kn1/quil-voltap](https://github.com/kn1kn1/quil-voltap)

LightTableで実行すると2つウィンドウが表示されますが、[ここ](https://github.com/LightTable/LightTable/issues/150)を見ると"quil's fault"ということのようです。(emacsだと2つ出てこないので何とも言えないところですが、、)

やっている内容としては、440Hz, 3000Hz辺りの音量をそれぞれ取得して、背景色、円の大きさなどに反映しています。


### まとめ

Overtoneを使って音とQuilを連携してみました。

* マイク入力から音量を取得できるようにして、Overtone以外の音源に対応した
* バンドパスフィルターを使用して特定周波数近傍の音量のみ取得できるようにした

ことで、QuilでのVJがより容易になったのではないかと思います。
