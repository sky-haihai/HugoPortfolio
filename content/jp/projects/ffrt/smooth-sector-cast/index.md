---
title: "開発ログ：平滑なセクター放射線キャスティング"
description: "平滑な中心角を持つリアルタイムセクター放射線キャスティング法"
date: "2024-05-16T17:45:58+0000"
type: page
image: "/images/ffrt/fancasticon.png"
---

## 目標

ゲームプロジェクト Fluffy Floaty Rubber Tennis は、物理ベースの固定カメラ宇宙テニスゲームです。プレイヤーの唯一の操作は**ラケットを振る**ことです。

ラケットを振る際にラケットと他の物体がどのように相互作用するかのロジックは、ラケットがヒットした物体に力を加え、プレイヤーに反作用力を加えることです。

## なぜ Physics.SphereCast を使わないのか？

では、なぜ内蔵の Unity 物理関数 `Physics.SphereCast` を使わないのでしょうか？

もしかしたらその背後にある本当のロジックを理解していないだけかもしれませんが、ヒット距離が 0 の時にバグがあると思います。

下のスクリーンショットに示されているように、プレイヤーが左に向かってスイングすると、スフィアキャストはキャラクターの原点で始まり、ラケットより少し先で終わります。予想される衝突点はスフィアキャストの「最初の球体」とその右側の長い障害物の間に発生します。スフィアキャストが障害物に到達するまでの移動距離が 0 であるため、ヒット距離も 0 であると予想されます。

{{< figure src="sphere-cast.png" >}}

この時点までは、すべてが私にとって理にかなっています。実際には、スフィアキャストは予想通りヒット距離が 0 でヒットを返しますが、ヒットポイントは `Vector3.Zero` にあります！

私は複数のテストを行い、結果は一貫しています。ヒット距離が 0 である限り、ヒットポイントは常に `Vector3.Zero` にあります...

## 新しいソリューションの構築

組み込みの関数が期待通りに動作しないため、`Physics.Raycast` を使用して独自のソリューションを構築することにしました。

このメカニズムは、このプロジェクトのニーズに基づいています：ラケットを振ること。

### 直感性

問題を分解すると、これらはスイングを直感的にするための重要なポイントです：

- 快適なスイングには、キャラクターの周りに小さな円の空間を含める必要があります。したがって、ボールがプレイヤーと重なっている場合、プレイヤーはそれを打つことができるはずです。
- スイングは扇形である必要があります。なぜなら、スイングは基本的に腕を回転させてラケットが扇形の領域をカバーするからです。
- スイングは、プレイヤーが狙っている方向またはマウスの位置に最も近い物体を最初にヒットする必要があります。

したがって、打撃領域は次のようにカバーする必要があります：

{{< figure src="new-1.png" height="200">}}

### メカニズム

上記のように、アイデアは同心円を2つ描き、円の中心を通る2本の線を引いて扇形の角度を決定し、ヒットポイントを接続して平滑な中心角を持つ扇形領域を形成することです。

パラメータは以下の通りです：
1. 重なり合う円の半径。
2. 扇形がある円の半径。（スイング時にラケットが届く距離）
3. 扇形の角度。

### レイキャスティング

その後、小さな円から大きな円まで定義されたレイ密度でレイをキャストできます。このプロジェクトでは、

{{< figure src="new-2.png" height="200">}}

### レイのソーティング

直感性セクションで述べたように、レイはプレイヤーがヒットしたい物体をヒットするようにソート/削除する必要があります。

たとえば、プレイヤーが右側を狙っていて、ヒットエリアと重なる緑の円がボールである場合、ソートなしのデフォルトの動作は非常にランダムです。たとえば、表示された画像では、レイは下から上、上から下、さらには中央からエッジに向かってキャストされることがあります。

{{< figure src="new-4.png" height="200">}}

{{< figure src="new-3.png" height="200">}}

この場合、次の3つのソーティング方法を追加することにしました：

1. **距離ソート**：セクターの中心からの距離に基づいてレイをソートします。

2. **角度ソート**：レイと照準方向との角度に基づいてレイをソートします。

3. **最初/最後のヒットソート**：レイがキャストされる順序に基づいてレイをソートします。

## 実装

いくつかのデバッグを経て、結果はかなり良好です。少なくともゲームプレイの観点からはそうです。

これは、さまざまなソート方法の完全な比較です：

<div style="display: flex; flex-wrap: wrap; justify-content: center; gap: 5px;">
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="first-hit.png" alt="first hit" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最初のヒット</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="last-hit.png" alt="last hit" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最後のヒット</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="dist-min.png" alt="distance min" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最小距離</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="dist-max.png" alt="distance max" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最大距離</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="angle-min.png" alt="angle min" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最小角度</div>
    </div>
    <div style="display: flex; flex-direction: column; align-items: center; width: 300px;">
        <img src="angle-max.png" alt="angle max" style="width: 100%; height: auto; aspect-ratio: 1 / 1;">
        <div style="margin-top: 0px; text-align: center;">最大角度</div>
    </div>
</div>

## ゲーム内でこの成果を見たいですか？
自分でゲームを試してみてください！

<iframe frameborder="0" src="https://itch.io/embed/2707388?bg_color=ffffff&amp;fg_color=2d2d2d&amp;link_color=ea5858&amp;border_color=cccccc" width="552" height="167"><a href="https://secondrealmstudio.itch.io/ffrt">Fluffy Floaty Rubber Tennis by Second Realm Studio, Charlotte Crosland, Yyuk1, Wonderboiz, Rina, sky-haihai</a></iframe>