---
layout: post
title:  MacでSonic PiとORCAをMIDIで連携させる
---

1/18(土)に64 No.6に出演しました。今回はSonic PiをORCAとMIDIで連携させてみました。OSCでの連携方法はよく見かけたのですが、MIDIでの連携方法(Mac)をあまり見なかったので、そのメモです。

結論としては、以下のポストの情報が有用でした。感謝。

[https://rbnrpi.wordpress.com/2017/07/19/sonic-pi-3-0-arrives-get-going-with-its-midi-and-osc-commands/](https://rbnrpi.wordpress.com/2017/07/19/sonic-pi-3-0-arrives-get-going-with-its-midi-and-osc-commands/)

ポイントは、"Audio Midi設定.app", "MIDIスタジオを表示", "装置はオンライン"。

以下、手順です。

### Audio Midi設定.app の設定
#### Audio Midi設定.app の起動
Audio Midi設定.app を起動します。以下の画面が出てくると思います。

<img src="{{site.baseurl}}/images/ORCA_SP_0.png">

#### [ウィンドウ] - [MIDIスタジオを表示] でMIDIスタジオを起動
分かりにくいが、メニューから辿ります。

<img src="{{site.baseurl}}/images/ORCA_SP_1.png">

#### IACドライバ の設定
何も設定していない状態で、"IACドライバ"(もしくはそれっぽい名前のもの)があると思いますので、それをダブルクリックします。

<img src="{{site.baseurl}}/images/ORCA_SP_2.png">

以下の画面が出てくると思います。"装置名"に識別しやすい名前(ここでは"IACdriver")を設定し、"装置はオンライン"にチェックを入れ、”ポート”に識別しやすい名前のポート(ここでは"port1")を設定する。

<img src="{{site.baseurl}}/images/ORCA_SP_3.png">

MIDIスタジオの"IACドライバ"のアイコンが明るい色に変わったと思います。これで設定完了です。

<img src="{{site.baseurl}}/images/ORCA_SP_4.png">

### Sonic Pi からの見え方
環境設定画面の”入出力”でMIDIポートが見えていればOKです。見えてなければ、"MIDIをリセットする"を実行すると良いかもしれません。

<img src="{{site.baseurl}}/images/ORCA_SP_6.png">


### ORCA からの見え方
画面下のほうに、"IACdriver port1"出てくると思います。出てなかったら、[Midi] - [Refresh Devices] 等を実行すると良いかもしれません。

<img src="{{site.baseurl}}/images/ORCA_SP_5.png">


### 確認

設定は以上です。以下、ソースコードを使用して確認します。

[https://gist.github.com/kn1kn1/8944657e1cc62b2947e54044da465ec3](https://gist.github.com/kn1kn1/8944657e1cc62b2947e54044da465ec3)

のgistに置いてあります。

#### Sonic Pi側
[https://gist.github.com/kn1kn1/8944657e1cc62b2947e54044da465ec3#file-9-rb](https://gist.github.com/kn1kn1/8944657e1cc62b2947e54044da465ec3#file-9-rb)

以下の箇所で、ORCAからのMIDIを受け付けています。`iacdriver_port1`の部分が、上で設定した"装置名"と"ポート"になります。

```
  note, velocity = sync "/midi/iacdriver_port1/0/2/note_on"
```

#### ORCA側
[https://gist.github.com/kn1kn1/8944657e1cc62b2947e54044da465ec3#file-orca-20b06-910764-orca](https://gist.github.com/kn1kn1/8944657e1cc62b2947e54044da465ec3#file-orca-20b06-910764-orca)

こちらは特に設定内容に依存するところはありません。


