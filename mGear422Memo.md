
# [mGear4.2.2](https://mgear4.readthedocs.io/en/latest/)

## インストール

<details><div>

## 重要

mGear はPymelを使用しています。Maya 2022 および 2023 では、Pymel はオプションのパッケージです。
Maya 2022/2023 をインストールするときに必ずインストールしてください。
Maya 2024 の場合、Pymel は pip を使用してインストールする必要があります。
こちらの[公式手順に従ってください](https://help.autodesk.com/view/MAYAUL/2024/ENU/?guid=GUID-2AA5EFCE-53B1-46A0-8E43-4CD0B2C72FB4)

## mGear のインストール

1. 最新の mGear リリースを[ここ](https://github.com/mgear-dev/mgear4/releases)からダウンロードします。
2. 解凍してください
3. mGearリリースフォルダの内容を Maya/modules フォルダにコピーします。
   - Windows: ユーザー<ユーザー名>ドキュメントMayaモジュール
   - Linux: ~/maya/modules
   - Mac OS X: ~/maya/modules
4. Maya を起動する

4.2.2ではMaya2022.3にドラッグアンドドロップでインストールできた。

</div></details>

# 検証内容

1. バインドジョイント直下にメッシュがある場合リグをどうするか :  
control_01コンポーネントのジョイント無しで作成しメッシュとペアレントコンストレインする
1. 持ち物の持ち替えはスペースマネージャーでできるか :  
スペースマネージャーでも問題ないが、control_01コンポーネントを武器コントローラにしてみた。IK Reference Arrayでスペーススイッチ可能。  
武器に手が追従する仕組みも検証確認できた。右手を例にするとグローバル用ウエポンctlを作り、prop_R0_ctlを子にして右手IK Reference Arrayのターゲットにprop_R0_ctlを追加しておく。(prop_R0_ctlを子にもうひとつ子があると握ったまま手首の角度をつけやすくなる)  
arm_R0_ik_cns_parentConstraint1のアトリビュート>prop_R0_ctlのTranslateを 0 0 0 にする。これでグローバルウエポンctlに手が追従するので武器ctlもグローバルウエポンctlに追従させればサイクルしない

1. メッシュ表示替えはでできるか : 機能はなさそう。自前MELを使用する。

1. UVスクロールや切り替え機能はできるか : 機能はなさそう。自前MELを使用する。

1. local_C0_ctl にスペーススイッチ : global_C0_ctl, world_ctlの切り替え可能

1. 既にバインドされたジョイントにリグを入れられるのか :  
mgearでビルドしてジョイントを作りそのジョイントを切り取ってモデルにバインドしてビルド時にリンクするか試して見たがうまくいかない。詳しくは[guide](#guide-settings)のConnect to existing jointsを参照  
mGearジョイントとバインドジョイントをコンストレインするMEL(yjp_ConstraintTableEdit)
mGearのマトリックスペアレントが使えれば処理は軽いが、成功していない。

1. global_C0_ctl にアニメを維持したまま原点固定する機能は存在するのか。:  
なさそうだったので自前MELをpythonにしてみた。global_C0_ctlのStayアトリビュートでglobal_C0_ctlの移動回転スケールのミュートができる

<details>
<summary>python</summary>
<div>

```python
from maya import cmds

Ground_0 = 'global_C0_root'
Ground_sdk = 'global_C0_ik_cns'
Ground_ctrl = 'global_C0_ctl'
cmds.select('global_C0_ctl')

cmds.addAttr( longName='Stay', attributeType='bool', k=1, defaultValue=0, minValue=0, maxValue=1 )

GroundInverse = cmds.createNode( 'inverseMatrix', n='GroundInverse' )
GroundChoice = cmds.createNode( 'choice', n='GroundChoice' )
GroundDecompos = cmds.createNode( 'decomposeMatrix', n='GroundDecompos' )

cmds.connectAttr("{}.matrix".format(Ground_ctrl),"{}.inputMatrix".format(GroundInverse))

cmds.connectAttr("{}.matrix".format(Ground_0),"{}.input[0]".format(GroundChoice))
cmds.connectAttr("{}.outputMatrix".format(GroundInverse),"{}.input[1]".format(GroundChoice))
cmds.connectAttr("{}.Stay".format(Ground_ctrl),"{}.selector".format(GroundChoice))
cmds.connectAttr("{}.output".format(GroundChoice),"{}.inputMatrix".format(GroundDecompos))

cmds.connectAttr("{}.outputTranslate".format(GroundDecompos),"{}.translate".format(Ground_sdk))
cmds.connectAttr("{}.outputRotate".format(GroundDecompos),"{}.rotate".format(Ground_sdk))
cmds.connectAttr("{}.outputScale".format(GroundDecompos),"{}.scale".format(Ground_sdk))
cmds.connectAttr("{}.outputShear".format(GroundDecompos),"{}.shear".format(Ground_sdk))
```

</div></details>

- [カスタムステップ](#Custom-Steps)をやってみる。出来た。

- mGearのマトリックスペアレントノードを使ってみる。ノードを調べたが難しい。

- 袖コントローラは手首のひねりに追従してしまう。ペアレントを変更するカスタムステップ処理が必要

- １シーンに複数体リグは入れられるか:  
ネームスペースエディターでセットカレントでキャラ名を選択してからビルドするとコントローラやジョイントにネームスペースが付く  
一部ネームスペースにネームスペースの子ができる。どんな意味があるか不明。

- 首のリグに関しての情報があったのでメモ  
mgearサンプルではスプラインを使っていたが、個人的にはそんなにジョイントいらない気がしている  
[フォーラム](http://forum.mgear-framework.com/t/lowpo-sagat-work-in-progress/3097/16)  
[動画](https://www.youtube.com/watch?v=6IDNuy_0zAY)

- IKのソフトネスは曲がる強さが増すだけでポップ回避ではない  
処理まで見ていないが差し替えは難しそうな予感

- ボーンダイナミクスをmgaerに組み込んでいる人発見。[boneDynamicsNode](https://github.com/akasaki1211/boneDynamicsNode)  
mgearのスプリングマネージャにコリジョンやグラビティが追加された感じ。  
自前のリグにも組み込める
FKctrlにベイクできるようにしたいので、ジョイントにベイクではなくFKctrlにベイクする  
FKctrlで動かしバインドjointがboneDynamicsで動くようにしたい
以下を使えば設定は簡単。各ジョイント単位でboneDynamicsNodeのパラメータを編集すればOK  
常時処理させると重たいはずなので、必要な時に仕込んでベイクが出来る仕組みがいい  
ジョイント単位のパラメーターを保存しておいてプリセットしておけば使いやすそう
人型の様なリグだけではなくエフェクトでも使えそう

```python
create_dynamics_node('joint1','joint2')
```

- スプラインIKのフリップを要確認。フリップしなければmgear一択になる

- ベンドとロールの分離した制御もなさそう。

- カスタムリグを作っている人を探す　カスタムリグを追加してみる

- 組み込み方がわかれば自前のリグを追加する[フォーラム](http://forum.mgear-framework.com/t/what-component-would-you-recommend-for-this-neck/4064/3)
- mGearジョイントとバインドジョイントをコンストレインするMELを作る。  yjp_ConstraintTableEditで作りyjp_ConstraintBat.pyで実行。

- シンプルリグ調べる

- スケールバグが高確率で発生するので各機能のスケールがまともに動作するか確認

- 自動制御関連の検証が必要 : フォーラム情報だと今の所バグが不安  
目の向きによって瞼の動きを連動させたりしていた

- 四足動物のリグテストが必要。獣と人のセットの場合など順次、特殊形状の生物

- mGearのFBX出力で情報用ロケータは出力されるのか : mGearのFBX出力を使わないかも

- クリップやアニメーションレイヤを使う記述があったのでトラックスエディターは使えそうだが要検証。  

- タイムエディターは問題ないか要検証

- 機能が多くなれば処理が重たくなる。自動化を入れていない簡易リグとフルの２つ使うのはあり。

---

# メリット

- コントローラを右クリックでIKFK切り替えやミラーフリップにアクセスしやすい
- スプリング機能で揺れ物モーションが作れる。手足のFKにも使用できる。
- IKのねじり用中間ジョイントや腕の伸縮などやりやすそう。
- IKアップベクターが自動で動いてくれて便利
- リバースフットがまともに動く
- スペーススイッチもできる
- フェイスリグもある
- リグピッカーもある
- マトリックス系ノードの種類が増えている。
- 処理を軽くする技術はレベルが高いと思う
- ドリブンもある。試して動作確認したが、フォーラムではバグ報告がある。

# デメリット

- IKのソフトネスの挙動がおかしい。結局ポップしている、自前のIKのほうがポップしない[こちらを参考にした](https://www.youtube.com/watch?v=4KE5KyWa_-E)
- 便利な機能多数だがバグなのか設定ミスなのか不明な点もある。  
プロジェクトに使用するのはトラブルが発生した場合に対処できない可能性がある。  
自前のリグだと問題対応や改造もしやすい。
- UVスクロール・UVパターン切り替え・メッシュ切り替えのリグはない。自作のMELを使う
- スプラインIKFKでマスターIKスレーブFKはあるがIKFK切り替え機能がない。
- ベンドとツイストを分離するノードはない
- NURBSによる追従機能がない
- フォーラムでそこそこ問題が上がっている。未解決もある。
- [chain_01でIKFK切り替えの挙動がおかしい](#chain_01)
- [フォーラム記事：ミラー用の文字「L・R」を「i・r」に変更したら動かなくなった](http://forum.mgear-framework.com/t/control-mirroring-broken/3687)
- [フォーラム記事：テールのねじれ問題](http://forum.mgear-framework.com/t/getting-a-good-tail-setup-with-mgear/747)

- [チェーンリグ関連のジョイントの向きがMayaと異なる。ウエイトのつけ方が変わるので注意  
ほかにガイドのルートと末端と同じ位置にジョイントが作られない](#chain_ik_spline_variable_fk_01)  

---

- [日本語のフォーラム](http://forum.mgear-framework.com/c/japanese/13)
- mGear でリグを入れた場合アニメーターはmGearプラグインを入れる必要がある
- リガー用とアニメーター用の説明書に分ける

---

# [クイックスタート](https://mgear4.readthedocs.io/en/latest/quickStart.html)

# GuidManager

<img src="https://furcraea.verse.jp/wp/wp-content/uploads/2021/01/image-4.png" width="50%">

- Guide Tools  
以下のボタンはComponent Listからガイドを作った後に使用するボタンです
  - `Settings` : コンポーネントを選択して押下すると設定ウインドウが表示される。[guide](#guide-settings)ノードはguide_root設定ウインドウが表示される。
  - `Duplicate` : コンポーネントを複製する。
  - `Dupl Sym` : コンポーネントをミラー複製する。
  - `Extr ctl` : コントロールカーブの編集をした後に実行すると保存され、次回ビルド時にカーブが読み込まれる
  - `Build From Selection` : guideルートを選択して、このボタンでリグをビルドします。
  - `Draw Component`  
    Component List で選んでこのボタンを押すとガイドが作られる

---

## Main Settings

各コンポーネントの共通項目  

<img src="http://forum.mgear-framework.com/uploads/default/original/2X/5/581d14905b01b22b7f7bccc5238361bcfda3e490.gif" width="30%">

- `Name` : コンポーネントの基本名。armとかlegとかです。
- `Side` : コンポーネントがどちら側にあるか。Left、Right、Center
- `Component Index` : 基本的には0　同じNameが複数あれば番号が増えていく
- `Connector` : どういった接続をするか❓　基本的にはstanderdでOK
接続したコンポーネントによってプルダウンメニューが増えます。orientation（向き）などもあるようです。

- Joint Settings
  - `Parent Joint Index`:  
    とりあえずOFFで-1のままでOK。  
    コンポーネントをchainなど末端の子にすることが多いが、IKやchainリグの何番目のジョイントの子にするか指定するための設定。
  
  - `Joint Names(None) Configure` :  
    Configureボタンを押す  
    ❓既存のジョイントとリグをコネクトする機能と予想  
    親から順に入力する  
    Hip/Knee/Footで試したがHipしかコネクトされずエラーが出る
  - `Orientation Offset XYZ` : 作成されるジョイントの向きを変更する

- Channel Host Settings
  - `Host` : IK/FK・ブレンド・スペース スイッチなど、このコンポーネントのエクストラアトリビュートを他のコンポーネントに表示させる機能。  
    主に、control_01のコンポーネントarmUIなど用意してから設定するようです。

  - `Custom Ctonrollers Group`:  ❓不明

- Colors Settings
  - 以下コントローラーのカラー設定

---

## Component List 種類

### [一部のコンポーネント解説サイト](https://www.mitsurog.com/cg/mgear%E3%81%AE%E3%82%B3%E3%83%B3%E3%83%9D%E3%83%BC%E3%83%8D%E3%83%B3%E3%83%88%E3%81%BE%E3%81%A8%E3%82%81%E3%80%90%E9%95%B7%E7%89%A9%E7%B7%A8%E3%80%91/)

#### arm_2jnt_01

>ロールボーン用Mayaノード付き2ボーンアーム。肘ピン付き

- Compoent Settings

  - `IK/FK Blend` : FKは0 IKは100（初期値100でOK）
  - `MaxStretch` : デフォルトは1.5倍に伸びる設定になっている。基本的には１でOK
  - `Divisions` : 上腕前腕の間のジョイント数　1 1 もしくは　2　2
  - `IK separated Trans and Rot ctl` : 末端コントローラHandの移動と回転を別Ctlにする
  - `MirrorIK ctl axis behaviour` : IKの移動回転がローカルで左右対称になる。オンでOK
  - `MirrorMid ctl and UPV axis behaviour` : Elbow ctl UPVをローカルで左右対称にする。オンでOK

  - `Add Extra Tweak Ctl` : tweak0_ctl上腕～tweak2_ctl手首（手ではない）が追加されIKの上にFKが乗っている階層。mid_ctlがあるので使う機会は少なそう。

  - `Support Elbow Joint` : Elbowジョイントの前後にジョイントを追加する。ゲームであればオフでOK。利点があるかもしれないので検証が必要。

  - `Squash and Strech Profile` : 拡縮のカーブを編集

  - `IK Reference Array` : スペーススイッチ（親を切り替える）のターゲットリストを追加する項目  
  shoulder_L0_tip local_C0_root body_C0_root spine_C0_eff spine_C0_root global_C0_root  
  などを<<で追加していく

  - `UpV Reference Array` : UpVのスペーススイッチ　Copy from IK Refで上記で設定したターゲットをコピーする

  - `Pin Elbow Reference Array` : 同じくElbowのスペーススイッチ

基本的にはarm_2jnt_01を使用してよさそう  
Arm コンポーネント全てのendジョイントの近くに手首ジョイントが作られる
どうしても減らしたい場合は不要なジョイントを消すpythonを書くか、別ジョイントにペアレントコンストして間接的にmGearを使う。

---

### arm_2jnt_02

>新しいアップベクターロールコントロール。ロールボーン用のMayaノードを持つ2ボーンアーム。肘ピン付き

arm_2jnt_01 とほぼ同じ
ロールプレーンの様なコントローラが追加される

---

### arm_2jnt_03

>新しいアップベクターロールコントロール。ロールボーン用のMayaノードを持つ2ボーンアーム。肘ピン付き

- Compoent Settings
  - `Elbow Thickness` : 肘の厚さ ❓

ロールプレーンの様なコントローラが追加される

---

### arm_2jnt_04

>新しいアップベクターロールコントロール。ロールボーン用のMayaノードを持つ2ボーンアーム。肘ピン付き。手首のアライメントオプション。

- Compoent Settings
  - `Elbow Thickness` : 肘の厚さ ❓
  - `Align wrist to guinde orientation` : 手首をガイドの向きに合わせる❓

---

### arm_2jnt_freeTangents_01

>クラシックなベンディ／ロールアームの2ボーンアーム。肘ピンと中央の接線1本のみ。

- Compoent Settings
  - `Divisions` : 上腕前腕の間のジョイント数　0 0 にはできない　最低 1 1
  - `Align wrist to guinde orientation` : 手首をガイドの向きに合わせる❓

肘に2つのジョイントが作られる

---

### arm_ms_2jnt_01

>ロールボーン用Mayaノード付き2ボーンアーム＋Simage仕様

- Compoent Settings
  - `Elbow Thickness` : 肘の厚さ ❓
  - `Divisions` : 上腕前腕の間のジョイント数　0 0 にはできない　最低 1 1

Support Elbow Jointをオフにできない

---

### cable_01

<img src="https://www.mitsurog.com/wp-content/uploads/2021/07/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88_2021-07-17_11_30_25.png" width="70%">

>2点取り付けでシンプルなケーブルリグを作成

両端を繋いだ橋状のリグ

- Compoent Settings
  - `Divisions` : ケーブルすべてのジョイント数
  - `Tip Reference Array` : 末端のスペーススイッチするターゲットを設定する

---

### chain_01

<img src="https://www.mitsurog.com/wp-content/uploads/2021/07/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88-2021-07-17-11.25.18.png" width="70%">

>シンプルIK/FKチェーン、IKスペーススイッチ付き

- FKチェーン  

各chainコンポーネントはDraw Compornentを押下した後  
Chain Initialaizerウインドウが表示される

![](http://forum.mgear-framework.com/uploads/default/original/2X/8/8d9b8ae9006d63f40d323354d85a4e3c2cd0b268.png)

- Selections Number : Rootを除いたIKの数？  
2で検証:黄色ガイド２個作られる。ビルドするとジョイント２個  
ik_ctlターゲットとその親のikcns_ctl、アップ用upv_ctlが作られる  
末端ジョイントはターゲットに向く  
ターゲットをルートに近づけすぎるとポップするので注意
ガイドの青いブレード方向にupv_ctlが作られて正解だがIK/FKを0～1切り替えると90度回転してしまう。バグっぽい。
- Direction : 先端の向き　XでOK
- Spacing : ガイドコントローラの間隔なので任意。後で調整できる。

- Compoent Settings
  - `Mode` : FK・IK・FK/IKのどれか選択
  - `IK/FK Blend` : FKは0 IKは100（初期値100でOK）
  - `Neutral Pose` : ニュートラルポーズってなに❓
  - `IK Reference Array` : スペーススイッチ（親を切り替える）のターゲットリストを追加する項目

<font color="Red">

FK/IKモードにしてガイドが真直ぐの場合はIK動作はせずルートのみエイム挙動になる。  
ガイドを少し湾曲させて配置させないとIK動作にならない。  
Preferred Angleを設定してもIK動作にはならない。
FKのみで使用するほうがいい

</font>

- FK/IKモードのアトリビュート
  - `position` : 無反応
  - `MaxStretch`: 無反応
  - `MaxSquash` : 無反応
  - `Softness` : 無反応
  - `IK/FK Blende` : IK またはFK によるアニメーションを切り替えるためのIK/FK ブレンディング
  - `Roll` : ハンドルベクトル軸回転　ジョイントX回転ではない
  - `Ik Ref` : スペース スイッチャー。どのスペースに従うかを制御します。

---

### chain_02

>シンプルなIK/FKチェーン、IKスペーススイッチ付き、
各セグメントが子セグメントを狙う感じ

<font color="Red">
IK・FK/IKにするとエラー出る。バグっている可能性大
</font>

- Compoent Settings
  - `Mode` : FK・IK・FK/IKのどれか選択
  - `IK/FK Blend` : FKは0 IKは100（初期値100でOK）
  - `Neutral Pose` : ニュートラルポーズってなに❓
  - `Each segment aims at its child` : ❓各セグメントは子セグメントをエイムする
  - `Mirror Behaviour L and R` : 左右対称にするかどうか

---

### chain_FK_spline_01

<img src="https://www.mitsurog.com/wp-content/uploads/2021/07/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88_2021-07-17_11_56_00.png" width="70%">

>スプライン駆動のFKチェーン。各セグメントに追加IKコントロール

- FKコントローラの上にチェーンIKが乗っている階層構造  
- コントローラとジョイントの位置が一致していないので使いにくい

---

### chain_FK_spline_02

<img src="https://www.mitsurog.com/wp-content/uploads/2021/07/chain_FK_spline_02-002.png" width="70%">

>スプライン駆動のFKチェーン。各セグメントに追加のIKコントロール。各ジョイントに追加の微調整オプションを追加。

- FKコントローラの上にチェーンIKが乗っている階層構造
- 末端ジョイントは回転しない。初期の向き固定

- Compoent Settings
  - `Neutral Pose` : ❓
  - `Keep Length` : 長さを変更する
  - `Overrride Negate Axis Direction For "R" Side` :❓  
  おそらくBehaviourなどシンメトリコントロールにした場合、軸が真逆になるのを防ぐ？
  - `Extra Tweaks` : アトリビュートTweakVis でctl表示してジョイントの位置をさらに調整できる
  - `Leaf Joint` : ❓

---

### chain_FK_spline_variable_IK_01

<img src="https://www.mitsurog.com/wp-content/uploads/2021/07/chain_FK_spline_variable_IK_01-002.png" width="70%">

>スプライン駆動のFKチェーン。そして可変数のIKコントロール。
FKがマスター、IKがスレーブ

FKコントローラの上にチェーンIKが乗っている階層構造  
末端ジョイントの回転はできない。  
Chain InitialaizerのSelection Number : Rootを除いたFKの数  
変更する場合はコンポーネントを作り直す必要がある。  
ikTip_ctl 末端のIKの先に付くコントローラ。使いどころ不明。IKコントローラに近づけたほうがよさそう。

- Compoent Settings  
  - `Neutral Pose` : ニュートラルポーズってなに❓
  - `Keep Length` : 長さを変更する
  - `Overrride Negate Axis Direction For "R" Side` :❓  
  おそらくBehaviourなどシンメトリコントロールにした場合、軸が反転するのを防ぐのかも？
  - `Extra Tweaks` : アトリビュートTweakVis でctl表示してジョイントの位置をさらに調整できる
  - IK Controls
    - `IK Ctl NUmber` : 全てのIKコントローラの数
  - Joint Options
    - `Overrride Joints Number` : ジョイントの数を上書き❓
    - `Joint NUmber` : ジョイントの数

---

### chain_IK_spline_variable_FK_01

<img src="https://www.mitsurog.com/wp-content/uploads/2021/07/chain_IK_spline_variable_FK_01-002.png" width="80%">

<img src="http://forum.mgear-framework.com/uploads/default/optimized/1X/53627f75b4f83bf8f25dd39c879aa4ae8de3ae8c_2_690x438.png" width="80%">

[バグ❓ スプライン コンポーネントが奇妙な関節変形を与える](http://forum.mgear-framework.com/t/mgear-spline-components-give-strange-joint-transforms/1283)  
<font color="Red">__スプラインコンポーネント全体的にこんな感じなのでウエイトのつけ方を変える必要がある。  
子の方向にウエイトをつけるのではなく、ジョイントの前後にグラデをかける。__</font>

<img src="http://forum.mgear-framework.com/uploads/default/optimized/1X/ed1b85681a2f1d8744d8087e4ff785162819f001_2_577x500.png" width="80%">

>スプライン駆動のIKチェーン。そして可変数のFKコントロール。
IKがマスター、FKがスレーブ

- Compoent Settings
  - FK Controls
    - `FK Ctl Number` : FKの数
    - `Overrride Negate Axis Direction For "R" Side` :❓  
    おそらくBehaviourなどシンメトリコントロールにした場合、軸が真逆になるのを防ぐ？
  - `Softness` :❓
  - `Position` : 0で根元詰め 1で末端詰め
  - `Max Stretch` : 伸ばす最大％
  - `Max Squash` : 伸ばした時に細くする％
  - `CTL Orientation` : コントローラーの回転軸が２つしかないかも

---

### chain_IK_spline_variable_FK_stack_01

<img src="https://www.mitsurog.com/wp-content/uploads/2021/07/chain_IK_spline_variable_FK_stack_01-001.png" width="80%">

>スプライン駆動のIKチェーン。そして可変数のFKコントロール。
IKがマスター、FKがスレーブ。IKとFKコントロールのスタック付き
 警告：このコンポーネント・スタックは1レベルのスタックしかサポートしていません。これにより、複雑な接続を避け、コンポーネントを少し軽く保つことができます。マスターの入力が多い場合、スレーブのスレーブは移動しません。直接スレーブのみ

チェーンIKにFKが乗っている階層構造  
根元と末端のFKコントローラがなぜかガイドの位置からずれて配置される  
コントローラのOffset Parent Matrixの値を入力して位置合わせしてもいいかも

- Compoent Settings

  - FK Controls
    - `FK Ctl Number` : FKの数
    - `Overrride Negate Axis Direction For "R" Side` :❓  
    おそらくBehaviourなどシンメトリコントロールにした場合、軸が真逆になるのを防ぐ？
  - `Add Joints` : どこかにジョイントを追加❓
  - `Softness` :❓
  - `Position` : 0で根元詰め 1で末端詰め
  - `Max Stretch` : 伸ばす最大％
  - `Max Squash` : 伸ばした時に細くする％
  - Chain Master
    - `Local` : ❓
    - `Global` : ❓
    - `Connection Offset` : ❓
    - `Only IK Global Master (No FK Ctl and Joints)` : ❓

---

### chain_IK_loc_ori_01

>ロケーターからオリエンテーションを取るシンプルなチェーン

FKの挙動。使用用途不明。

- Compoent Settings
  - `Neutral Pose` : ニュートラルポーズってなに❓
  - `Overrride Negate Axis Direction For "R" Side` :❓  
  おそらくBehaviourなどシンメトリコントロールにした場合、軸が真逆になるのを防ぐ？
  - `Add Joints` : どこかにジョイントを追加❓

---

### chain_net_01

>❓説明なし

FKコントローラの上にチェーンIKが乗っている階層構造  
末端ジョイントの回転はできない。  
よくわかっていない。

- Compoent Settings
  - `Neutral Pose` : ニュートラルポーズってなに❓
  - `Is Only Master (No Joints)` :
  - `Keep Length` : 長さを変更する :
  - `Overrride Negate Axis Direction For "R" Side` :❓  
  おそらくBehaviourなどシンメトリコントロールにした場合、軸が真逆になるのを防ぐ？
  - `Extra Tweaks` : アトリビュートTweakVis でctl表示してジョイントの位置をさらに調整できる
  - Visibility UI Host
    - `Vis. Host` : ?
  - Chain Master
    - `MasterA` : ❓ガイドを選択して<<を押下しても適用されない
    - `MasterB` : ❓ガイドを選択して<<を押下しても適用されない
    - `Bias A/B` : MasterA　MasterB のブレンドだと思う
    - `Connection Offset` : インデックス番号だと思う
  - Joint Options
    - `Override Joints Number` : ジョイント数上書きするか
    - `Joints Number` : ジョイント数

---

### chain_spring_01

>スプリング付きFKチェーン

- `Selection Number` : ジョイントの数

- `Compoent Settings` : 設定項目は無し

アトリビュートに追加される項目

- `Spring chain intensity` : 揺れ機能の有効度 0から1
- `damping_*` : 減衰？　0.5でよさそう
- `stiffness_*` : 硬さ　初期ポーズを維持する強さ 0.2-0.5ぐらい
各ジョイントで設定できる

各コントローラにキーがうてる。  
intensityを0で揺れなくすることはできるが常に揺れる処理が入ってるので重そう。

---

### chain_spring_lite_stack_master_01

>lite_chain_stack のマスターになれるスプリング付き FK チェーン  

lite_stack_master名前からして処理が軽い？　挙動がスタックしない？　上位版？

- `Selection Number` : ジョイントの数

- `Compoent Settings` : 設定項目は無し

- アトリビュートに追加される項目
  - `Spring chain intensity` : 揺れ機能の有効度 0から1
  - `damping_*` : 減衰？　0.5でよさそう
  - `stiffness_*` : 硬さ　初期ポーズを維持する強さ 0.2-0.5ぐらい
  各ジョイントで設定できる

各コントローラにキーがうてる。  
intensityを0で揺れなくすることはできるが常に揺れる処理が入ってるので重そう。

---

### chain_stack_01

>特殊なコネクターで、1本のマスター・チェーンから何本ものチェーンを駆動できるスタッキング・チェーン。当初はアニメスタイルヘアーのためにデザインされましたが、どのような目的にも使用できます。
このコンポーネントは'chain_FK_spline_02'にあります。

---

### chain_variable_IK_01

>IKコントロール数可変のチェーン

チェーンIK　根元と先端はコントローラに追従  
回転はXしかできない。あまり使用しないかも。

- Compoent Settings

  - `Keep Length` : 長さを維持する
  - `Extra Tweaks` : アトリビュートTweakVis でctl表示してジョイントの位置をさらに調整できる
  
  - IK Controls
    - `IK Ctl NUmber` : 全てのIKコントローラの数
  - Joint Options
    - `Overrride Joints Number` : ジョイントの数を上書き❓
    - `Joint NUmber` : ジョイントの数

---

### chain_whip_01

>スライド式ウィップコンポーネント

FKコントローラにチェーンIKが乗っている感じ  
末端ジョイントより先にもコントローラがあるがX回転以外に意味を感じない。  
末端ジョイントのIKコントローラの回転とジョイントは一致しないので注意。

- Compoent Settings
  - `Neutral Pose` : ニュートラルポーズ❓
  - `Keep Length` : 長さを維持する
  - `Overrride Negate Axis Direction For "R" Side` :❓  
  おそらくBehaviourなどシンメトリコントロールにした場合、軸が真逆になるのを防ぐ？
  
  - Joint Options
    - `Overraid joints Number` : 初期設定したジョイント数を上書き
    - `Joint Number` : chain_whip全てのジョイント数

---

### control_01

>スペーススイッチと回転順序選択を備えたシンプルなコントロール装置。
このコンポーネントは、ルート回転を使用してコントロールの向きを配置できます。
注意: MAYA 2018および2018.1には、負のスケールでの動作を壊すバグがあります。これは「ミラー動作オプション」に影響します。

単品のFKコントローラ。MainSettingsは同じ

- Compoent Settings
  - `Joint` : ジョイントを作る
  - `Add Leaf Joint` : 子のジョイントがある❓
  - `is Leaf Joint` : 親のジョイントがある❓
  - `Uniform Scale` : 均一なスケールをする
  - `World Space Orientation Align` :　名称body、センターに使われている。  
  １つのジョイントを仕込む場合　特定の向きを向けたい場合はOFFにする
  
  - `Mirror Behaviour L and R` : 左右対称にするかどうか
  - `ctl Size` : コントローラーサイズ
  - `Control Shape` : カーブコントローラをプルダウンから選択

  - Keyble
    各キーをつけたいチャンネルをチェックする  
    Rotateのroはなんだろう。  
    bodyの設定ではAxisがZXYになっている。

- 前腕に袖ジョイントSleeve_L0_ctlを追加した場合ParentJoint indexを 1 にするとarm_L0_1_jntの子になる  
ただし arm_L0_div2_loc に Sleeve_L0_root がペアレントされてしまうので、ハンドをX回転させると袖も回転してしまう。
回転してほしくない場合は Sleeve_L0_root を arm_L0_div1_loc の子にする必要がある。ガイドにその設定が見当たらない。

- eyeslook_C0_root の IK Reference Arrayリスト
  - neck_C0_head
  - local_C0_root
  - body_C0_root
  - spine_c0_eff

---

### eye_01

>アイコントロールリグ

普通のエイムコンストレイン

- Compoent Settings
  - `Up Vector Direction` : YでOK
  - `IK Reference Array` : サンプルでは eyeslook_C0_root という control_01 が設定されている

---

### foot_bk_01

>フットロールをコントロールするための逆コントローラー付きフット。

- リバースフット
  SectionsNumber : ３で母指球、指、つま先になる
  指まで入れなければ２、[１はバグるフォーラムでは進展無し](http://forum.mgear-framework.com/t/foot-bk-01/762)
- `Direction` : XでOK
- `Spacing` : とりあえず1でOK
- leg_2jnt_02の子の場合connector:standerdをleg_2jnt_01に変える

- Compoent Settings
  - `Use Roll Ctl` :
  - `Default Roll Angle` : つま先が支点になって曲がりだす角度かと思う。いったん-20でOK

---

### hydraulic_01

>機械式リギング用の油圧コンポーネント。

- エイム挙動
- 根元と末端にコントローラが作られる

- Compoent Settings
  - 油圧？

---

### leg_2jnt_01

>ストレッチ、丸み、ik/fk...を持つ2本のボーンレッグ。

- Compoent Settings

❓

---

### leg_2jnt_02

>膝の太さをコントロール。ストレッチ、丸み、ik/fk...を備えた2ボーンレッグ。

- Compoent Settings
  - Mirror Mid Ctl and UPV axis : コントローラのシンメトリ挙動
  - Knee Thickness : 膝を曲げた時に膝の位置を末端へ移動させるアトリビュート。膝移動時ヒップは回転しない。
  - Add Extra Tweaks ctl: アトリビュートTweakVis でctl表示してジョイントの位置をさらに調整できる
  - Support Knee Joints : 膝の近くに２つジョイントが作られる。AAAタイトル、映像ハイモデル用。

<font color="Red">

foot_bk_01仕込む場合、設定connector:standerdをleg_2jnt_01に変えるとOK

</font>
---

### leg_2jnt_freeTangents_01

>オートUPV。ストレッチ、ラウンドレス、IK/FK...クラシックなコアロールを備えた2ボーンレッグ。ニーピンと1本のセンタータンジェント付き。

- Compoent Settings

    ❓

---

### leg_3jnt_01

>四足動物およびその他の動物の骨脚3本

- Compoent Settings

    ❓

---

### leg_ms_2jnt_01

>2ボーンレッグ（ロールボーン用Mayaノード付き）＋Simage仕様

- Compoent Settings

    ❓

---

### lite_chain_01

>超シンプルで軽量なFKチェーン

- Compoent Settings
    ❓

---

### lite_chain_stack_01

>特殊なコネクターにより、1本のマスターチェーンから複数のチェーンを駆動できるスタッキングチェーン。当初はアニメヘアー用にデザインされたが、どのような用途にも使用できる。

- Compoent Settings
    ❓

---

### lite_chain_stack_02

>特殊なコネクターにより、1本のマスターチェーンから複数のチェーンを駆動できるスタッキングチェーン。当初はアニメヘアー用にデザインされたが、どのような用途にも使用できる。
 ローカル接続とワールド接続のブレンドコントロール付き。

❓

- Compoent Settings

---

### meta_01

>中手骨を一括コントロールする

- Compoent Settings
  - `Interpolate Scale` : 補間 ❓ 不明
  - `Interpolate Rotation` : 補間 ❓ 不明
  - `Interpolate Translation` : 補間 ❓ 不明
  - `Joint Chain Connection` : ジョイントをつなげる ❓ 不明
  - `Meta ctl` : 各中手骨コントローラを作る。

  メイン設定は基本と同じ
  キャラの指の根元に使用されている
  Joint Chain ConnectionだけOFF

[フォーラム：中指骨の問題](http://forum.mgear-framework.com/t/meta-01-and-fingers-how-do-they-work/3123)
meta_L0_meta0_ctl を動かしても指が付いてこない。

![meta](http://forum.mgear-framework.com/uploads/default/original/2X/0/0ec3cf14dd9ab9ba7c68bfe6e35f37daac34b54f.png)  
今の所、指が子になっていないので後でペアレントし直すしかない。

<details>
<summary>フォーラムにあるようにビルド後処理するしかない</summary>
<div>

```python
import pymel.core as pm

for side in 'LR':
    for meta in range(4):
        metaCtl = pm.PyNode('meta_{}0_meta{}_ctl'.format(side,meta))
        fingerRoot = pm.PyNode('finger_{}{}_ik_cns'.format(side,meta)).getParent()
        pm.parent(fingerRoot,metaCtl)
```

</div></details>

---

### mouth_01

>口唇と顎

- Compoent Settings
    ❓

---

### mouth_02

>口唇と顎 エクストラオフセット 顎のコントロール

- Compoent Settings
    ❓

---

### neck_ik_01

>ネックIKとFK、オプションのタンジェントコントロール付き

- Compoent Settings
  - `Softness` : ❓
  - `Max Stretch` : 伸ばす倍率　基本的に1.0でOK
  - `Max Squash` : 伸ばした時に補足する値デフォルト0.5
  - `Divisions` : ジョイント数
  - `Tangent Controls` : ❓
  - `IK ctl World Ori` : ❓

  - IK Reference Array
    - `Chicken style IK` : 鳥の様な首❓
    - スペーススイッチ設定

  - Head Reference Array
    - スペーススイッチ設定

[フォーラム記事：IKで首を制御しようとすると首コントロールの回転に従うのではなく、少し移動される](http://forum.mgear-framework.com/t/head-control-translating-instead-of-just-rotating-with-neck-in-ik/3489)

---

### sdk_control_01

> ドライブのキーを設定するコントローラー
> ❓

---

### shoulder_01

> シンプルな肩。
 腕にスペーススイッチ、腕にオービットレイヤー
> エイムリグかな

- Compoent Settings
    ❓

---

### shoulder_02

> シンプルなショルダー。
 このショルダーは3箇所あり、軌道をコントロールの先端から切り離すことができる。

- Compoent Settings
    ❓

---

### shoulder_ms_01

> ms_arm/leg と組み合わせて使用する肩／手足のコネクター

- Compoent Settings

> ❓

---

### spine_FK_01

> IKコントロール付きFKスパイン

- Compoent Settings

> ❓

---

### spine_S_shape_01

>S字型の背骨コンポーネント。特徴は'spine_ik_02'コンポーネントに基づいています。

- Compoent Settings

> ❓

---

### spine_ik_01

>IKスパインにFKコントローラーを重ねたもの。
IKポジションに追従する オプションのオートベンドIKコントロールとセントラル
タンジェント・コントロール。

- Compoent Settings

    IKを動かしてみるとジョイントにスケールアニメが入ってしまう。
    ❓

---

### spine_ik_02

>このバージョンには股関節が追加されている。IKスパインFKコントローラー
IKポジションに追従する オプションのオートベンドIKコントロールとセントラル
タンジェント・コントロール。

- Compoent Settings

> ❓

---

### squash4side_01

>Squash4Sidesコンポーネント。注意：このコンポーネントは、ルートの全回転を使用します。

- Compoent Settings

> ❓

---

### squash_01

>リニア・スカッシュ・コンポーネント

- Compoent Settings

> ❓

---

### tangent_spline_01

>タンジェントコントロールとIKリファレンスアレイを備えたスプライン。
すべてのコントロールはワールド指向

- Compoent Settings

> ❓

---

### ui_container_01

>UIコンテナは、子UIコントローラの周りにシンプルなボックスを作ります。また、アニメーション中に ui をオフセットできるようにコントローラを追加します。

- Compoent Settings

> ❓

---

### ui_slider_01

>インターフェースの構築に使われるシンプルなスライダーコントロール。blendshapesやsetDrivenKeysとのリンクに便利。

- Compoent Settings
  - `Mirror Behaiour L and R` :
  - `Control Size` :
- Range of Motion
  - `TranslateX Negative Positive` : txのレンジで使用する範囲プラスのみならマイナスをオフにする。であってる❓
  - `TranslateY Negative Positive` :tyのレンジで使用する範囲プラスのみならマイナスをオフにする。

---

### EPIC_arm_01

>EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
arm_2jnt_02に基づく

- Compoent Settings

---

### EPIC_arm_02

> 説明なし

Compoent Settings

- `Leaf Jjoint` : ❓
- `Use Wrist Blade` : ❓
- `Rest T Pose` : Tポーズにリセットできる❓

---

### EPIC_chain_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
lite_chain_01がベース。ジョイント名はコンポーネントインスタンス名から取得

- Compoent Settings

---

### EPIC_chain_IK_FK_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
chain_01がベース。ジョイント名はコンポーネントのインスタンス名から取得
シンプルなIK/FKチェーン、IKスペーススイッチ付き

- Compoent Settings

---

### EPIC_control_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
control_01がベース。コンポーネント・インスタンス名から取ったジョイント名

- Compoent Settings

---

### EPIC_foot_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
foot_bk_01 に基づいています。

- Compoent Settings

---

### EPIC_hydraulic_01

> 機械式リギング用の油圧コンポーネント。

- Compoent Settings

---

### EPIC_layered_control_01

> RBFや他のタイプのドライバーを接続するために、複数のレイヤー入力コントロールを持つコントロール

- Compoent Settings

---

### EPIC_leg_01

>EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
leg_2jnt_01 に基づいています。

- Compoent Settings

---

### EPIC_leg_02

> 説明なし

- Compoent Settings

---

### EPIC_leg_3jnt_01

> ゲーム3 四足動物やその他の動物の骨脚

- Compoent Settings

---

### EPIC_mannequin_arm_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
arm_2jnt_02に基づく

- Compoent Settings

---

### EPIC_mannequin_leg_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
leg_2jnt_01 に基づいています。

- Compoent Settings

---

### EPIC_meta_01

> 中手指の広がり

- Compoent Settings

---

### EPIC_neck_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
neck_ik_01 に基づいています。

- Compoent Settings

---

### EPIC_neck_02

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
SplineIKソルバーを使用した曲率

- Compoent Settings

---

### EPIC_shoulder_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
shoulder_01をベースに、コンポーネントインスタンス名から取ったジョイント名

- Compoent Settings

---

### EPIC_spine_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
このコンポーネントはmetaHumanの背骨構造にマッチします。

- Compoent Settings

---

### EPIC_spine_02

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
このコンポーネントはmetaHumanの背骨構造にマッチします。
SplineIKソルバーを使用した曲率

- Compoent Settings

---

### EPIC_spine_cartoon_01

> EPICのUEやその他のゲームエンジン用のゲームレディコンポーネント
spine_S_shape_01 に基づいています。

- Compoent Settings

---

## Draw Component

Component List で選んでこのボタンを押すとガイドが作られる

## Show Setting After Create New Component

チェックされている時にDraw Componentを押すとSettingsも開く

---

## guide settings

- Rig settings
  - `Rig Name` : リグの名称
  - `Debug Mode` :  ❓ 　Finalのままでよさそう
  - `Guid Build Steps` :  ❓ 　Finalizeのままでよさそう  
ビルドを段階的にする項目。デバッグする時に変更する。

- Animation Channles Settings
  - `Add Internal Proxy Channels` :  ❓
  - `Use Classic Channel Names (All channnels will have uniquw names)` :  ❓
  - `Use Component Instanse Name for Attributes Prefix` :  ❓

- Base Rig Control
  - `Use World Ctl or Custom Name` : Ground_ctlなど名称変更したいときはチェックを入れる  

- Skinning Settings
  - `Import Skin` : スキンを読み込む
  - `skin path` : スキンファイルの場所  

- Joint Setteings
  - `Separated Joint Straucture` : 有効にするとjnt_orgグループにジョイントが格納される。無効にするとコントローラの階層にジョイントがぶら下がる。
  - Force uniform scaling in all joints by connection all axis to Z axis<br>
  すべての軸をZ軸に接続することで、すべてのジョイントに均一なスケーリングを強制する。mgear_matrixConstraintを見るとScaleZをSX,SY,SZにコネクトしている。
  - Connect to existing joints.NOTE:joints need to have freezed rotations values.  
  既存のジョイントに接続する。
  注意： ジョイントは、回転値をフリーズする必要がある。  
  検証した所、ルート以外コネクトされない。  
  ガイドから作られたジョイントを使ってバインドしてもコネクトされない。  
  各コンポーネントのConfigureを設定してもルートのみ、(１度成功したが原因不明)  
  ジョイントのAXISを自由に決めたい場合はバインドジョイントとmGearジョイントをペアレントコンストレインしたほうが確実。ただしジョイントが倍になるので処理が気になる。  
  まだ試していないが、mGearジョイントのmgear_matrixConstraintをバインドジョイントとコネクトしても角度が変わってしまう。
  drivenRestMatrixの回転を調整してうまくいったジョイントもあるが、右腕足はだめかもしれない。  
  applyop.pyのgear_matrix_cnsがマトリックスコンストレインをつくる処理  
  右肘で試したがうまくいかず向きがバグる。  

- Post Build Data Collector(Experimental)
  - `Collect Data on External File` : <span style="color: rgb(255,10,10);"> 不明
  - `DataPath` : ファイルの場所
  - Collect Data Embbeded on Root or custom joint(Worning: FBX Ascii format is not supported)  
  ルートまたはカスタムジョイントに埋め込まれたデータを収集する（警告：FBX Asciiフォーマットはサポートされていません。）
  - Custom joint or Transform <span style="color: rgb(255,10,10);">不明

- Synoptic Settings
<span style="color: rgb(255,10,10);">まだ確認していない

  - Rig Tabs
  - Availlable Tabs

- Color Settings  
以下コントローラのカラー設定

---

### Custom Steps

- Pre Custom Step
たぶんリグをビルドする前に実行するpythonを設定する

- Post Custom Step  
リグビルド完成後に実行するpythonを設定する  
yjp_StayCtrlを試してみた。  
キャラクター専用のpyを作ってそこに複数のdefを記述すればOK。
yjp_ConstraintBat(パス)も書いておけばリグジョイントとバインドジョイントをコンストできる  
キャラ名_post.pyを用意 内容は以下

```python
from yjp_rig import yjp_StayCtrl
from yjp_rig import yjp_ConstraintBat

#Stayアトリビュート追加
yjp_StayCtrl.yjp_StayCtrl()
#コンスト追加
yjp_ConstraintBat.yjp_ConstraintBat(csvファイルパス)
```

yjp_ConstraintBatでジョイントとジョイントをコンストしたが、コントローラとジョイントをコンストすればノードが減る  
スケールコンストレインをやっていなかった。結構ノード増えそう。
mgear_matrixConstraintで接続すればスケール込みなのでノードも減るが、検証が必要
metaのペアレント修正なども書いておける。  
武器コンストの値も変更できる
各コンポーネントのアトリビュート値も設定できそう。

メッシュとコントローラのコンストも問題ない。  
リグの両腕の肘を前でくっつけるようなポーズにした時　肘のメッシュがおかしい

---

### Naming Rules

- Controls Naming Rule
デフォルトでとりあえずOK
  - {descrition} LetterCase デフォルトでOK  

- Joints Naming Rule
デフォルトでとりあえずOK
  - {descrition} LetterCase デフォルトでOK  

- Controls Sides Naming 　Joints Sides Naming
どちらもデフォルトでOK  

- index Padding  
強制桁数　とりあえず0でOK  

- Extensions Naming  
デフォルトでOK

- Load Save  
上記の設定を読み込み保存

---

## IKのアトリビュート

- `Ref` : アトリビュートに複数のコンポーネントが並ぶため、ここから下はこのコンポーネントですよという意味だと予想
- `Roll` : 極ベクトルを移動せずに肘の方向を調整できるロールアトリビュート
- `Armpit Roll` : 肩関節周囲のツイストジョイントを手動で調整するための脇ロール
- `Scale` : 腕全体のスケール
- `IK/FK Blende` : IK またはFK によるアニメーションを切り替えるためのIK/FK ブレンディング
- 肩と手首の関節周りの変形を改善するためのツイストジョイント
- `MaxStretch`: IK コントロールが遠すぎて届かない場合に腕を伸ばすためのストレッチ(最大ストレッチ制限を含む)
- `Slide` : 肘のスライディング。上腕を伸ばすと同時に下腕を短縮します。またはその逆も同様です。
- `Softness` : IK アームをまっすぐに伸ばしたときにポップを防ぐ柔らかさ
- `Reverse` : IK 使用時に腕を反対方向に曲げます。
- `Roundness` : ゴムホース風のアニメーションの丸み
- `Volume` : ストレッチ中のボリューム維持  
たとえばキャラクターがテーブルに肘を置いている場合など、肘を正確に位置決めするための肘コントローラー
- `Ik Ref,IkRot Ref, UpV Ref, Elbow Ref` :  
IK、アップ ベクター、エルボー コントローラー用のスペース スイッチャー。どのスペースに従うかを制御します。

---

## GameTools

## FBXExportベータ

<img src="https://mgear4.readthedocs.io/en/latest/_images/fbx_shifter.png" width="50%">

    まだ試していない。

Shifter’s FBX Exporter

- File : ユーザーが設定を再ロードしたり、スクリプト化されたパイプラインで使用したりする場合に、設定をシリアル化できるようにします。

### Source Elements

- `Geo Root` : ジオメトリオブジェクトルートのリスト。キャラクターをどのように構成したかに応じて、ジオメトリが複数ある場合があります。
- `Joint Root`: スケルトンのルート

### Settings

FBXエクスポート設定また、エクスポート時にデータを調整することもできます。

- FBX
  - `Up Axis` : とりあえず Y でよさそう
  - `File Type` : バイナリかアスキー
  - `FBX Version` : FBXバージョン
  - `FBX Preset` : 下記の出力設定を保存するとプルダウンに表示されるので選択する  
MayaのプリファレンスでFile/Projects>File Dialog>OS nativeにしたほうがいいらしい。  
MayaのFile>Export Selection Option>File Type Specific Options>Edit presetを押下して、FBX出力設定を保存する。

- FBX conditioning
  - `Remove Namespace` : ネームスペースの削除
  - `Clean up scene` : FBX内の不要データを削除

### File Path

- `Directory` : エクスポートされた FBX ファイルの場所。
- `File Name`: 生成される fbx ファイルの名前。これは、 Unreal アセットの名前としても使用されます。

### Unreal Engine Import

- `Unreal Engine Import` : これを有効にすると、他の Unreal UI 要素がアクティブになることが可能になります。
また、現在開いている Unreal プロジェクトをクエリすることで、Unreal Skeleton リストを更新します。
- `Directory` : SKM とアニメーションの Unreal でのインポート場所。

<img src="https://mgear4.readthedocs.io/en/latest/_images/fbx_shifter_ue_select_folder.png" width="80%">

1. インポート先の Unreal 内のフォルダーに移動します。
2. Unreal のコンテンツ ブラウザでフォルダーを選択します

3. Shifter UI でフォルダー アイコンをクリックします。

4. ディレクトリへのパッケージ パスは Unreal から取得されます。必要に応じて変更でき、インポート時にフォルダー構造が生成されます。

### Export

#### Skeletal Mesh

スケルトンとジオメトリのエクスポートを可能にします。

<img src="https://mgear4.readthedocs.io/en/latest/_images/fbx_shifter_export_geo.png" width="50%">

- `Skinning`: スキニングデータをエクスポート
- `Blendshapes`: ジオメトリ上に存在するブレンドシェイプをエクスポートします。
- `Partitions`: パーティション化された FBX をエクスポートします。
- `Cull Joints` : 有効にすると、生成された fbx パーティション ファイルから不要なリーフ ノードがすべて削除されます。

- パーティション

ジオメトリデータの分割を実行します。パーティションは、一度エクスポートすれば、パーティションごとに FBX を生成できるように設計されています。生成された各 FBX パーティションには、パーティションに追加されたジオメトリのみが含まれます。

- ジオメトリルートを追加すると、すべてのジオメトリの子オブジェクトがマスターパーティションに追加されます。

<img src="https://mgear4.readthedocs.io/en/latest/_images/fbx_shifter_export_geo_partitions.png" width="50%">

- 「+」ボタンを押してカスタムパーティションを作成します。作成したら、他のジオメトリ オブジェクトをマスター パーティションからカスタム パーティションにドラッグできます。
- パーティションを右クリックして、色の変更、複製、削除を行います。
- パーティション上のボタンを切り替えると、エクスポートが無効になります。

  - `Export Skeletal/SkinnedMesh` : FBX エクスポートを実行します。「Enable Unreal Engine Import」がアクティブな場合、FBX はアクティブな Unreal Engine プロジェクトにインポートされます。

#### Animation

Maya アニメーションを FBX としてエクスポートします。クリップを使用すると、アニメーション レイヤも利用しながら、Maya タイムラインのセクションをエクスポートできます。

- Clip  
クリップを使用すると、Maya タイムライン上の時間のセクションを表す名前付きアニメーションを作成してエクスポートできます。新しいクリップは Maya タイムラインの長さを自動的に読み取り、それを開始フレームと終了フレームとして使用します。

  - `ゴミ箱`: クリップを取り外します。
  - クリップの名前がファイルに追加されます。例えば。BoyA_ROM、BoyA_Clip_2
  - `ドロップダウン`: アニメーションが読み取られ、最終的な FBX としてエクスポートされるアニメーション レイヤを表します。
  - `Start Frame` : アニメーションが開始されるフレーム番号。
  - `End Frame` : アニメーションが終了するフレーム番号。
  - `タイムライン範囲を設定`: 指定された範囲に合わせて Mayas タイムラインを更新しました。
  - `再生ボタン`: クリップをループで再生します。
  - `チェック ボックス`: アニメーション クリップを無効にし、クリップのエクスポートを停止します。

- Animation Layers

<img src="https://mgear4.readthedocs.io/en/latest/_images/fbx_shifter_animation_cb.png" width="50%">

- None : アクティブなアニメーション レイヤと無効なアニメーション レイヤの現在の設定を使用します。シーンに表示されているものをエクスポートします。

- 選択した他のアニメーション レイヤは、アニメーション レイヤとMayas BaseAnimationをエクスポートします。

<img src="https://mgear4.readthedocs.io/en/latest/_images/fbx_shifter_maya_anim_layers.png" width="50%">

[原文FBXExportマニュアル](https://mgear4.readthedocs.io/en/latest/shifterUserDocumentation.html)

---

- Disconnect Joint  ❓
- ConnectJoint  ❓
- DeleteRig + KeepJoints  ❓ 　ジョイントとバインドを残してリグを消すんだとおもう
- GameTool Disconnect + Assembly IO  ❓

[フォーラム記事：FBX エクスポートがジオメトリとジョイントをワールドにペアレント化しない](http://forum.mgear-framework.com/t/fbx-export-not-parenting-geometry-and-joints-to-world/3608)

## Settings2

- コンポーネントとguideルートの設定をする　GuidManagerにもある。

## Duplicate

- コンポーネントを選択して実行すると複製する　GuidManagerにもある。

## Duplicate Sym

- コンポーネントを選択して実行するとミラー複製する　GuidManagerにもある。

## Extract Controls

- コントロールカーブの編集をした後に実行すると保存され、次回ビルド時にカーブが読み込まれる

## Build from Selection

- guideルートを選択して実行するとビルドする。

## Build from Guide Template File

- guideファイルを読み込んでビルド実行する。

## Rig Builder

    ❓

## Import Guide Template

- 作成したガイドを読み込む

## Export Guide Template

- ガイドを保存する

## Extract Guide From Rig

    ❓

## Extract and Match Guide From Rig

    ❓

## Guide Template Samples

- `Biped Template, Y-up` : 人型
- `Quadrped Template, Y-up` : 四足
- `Game Biped Template, Y-up` : ゲーム用の人型　EPICリグのテンプレート

## Match Guide to Selectd joint Hirarchy

    ❓

## Auto Fit Guide (BETA)

    ❓

## Plebes

     メーカー固有のキャラクター ジェネレーターからキャラクターをすばやくリグするための、シンプルなテンプレート ベースのツール。モデルデータを提供された場合は使うかもしれないが、ほぼ使わないと思う。

## Mocap

- `HumanIK Mapper` : ❓
- `Import Mocap Skelton Biped(Legacy)` : ❓
- `Charactrize Biped(Legacy)` : ❓
- `Bake Mocap Biped(Legacy)` : ❓

## Update Guide

- Guideを選択して実行すると何かしている❓不明

## Reload components

- おそらく独自のコンポーネントを追加した時に読み込むのではないかと

## Build Log

- `Toggle Log`:
- `Toggle Debug Mode`:
- `Sifter Log Window`:ビルド実行時のログウインドウを表示する

---

# ueGear

    アンリアル用　保留

# Simple Rig

    アイテムや機械などのリグ❓
<img src="https://user-images.githubusercontent.com/1050212/41524743-6a5defe0-7319-11e8-98ff-d3190b8baa1b.png" width="40%">

- File
  - `Export Config` : SimpleRigの保存❓
  - `Import Config` : SimpleRigの読み込み❓
- Convert
  - `Convert Shifter Rig` :❓
  - `Create Shifter Guide` :❓
- Delete
  - `Delete Pivot` :❓
  - `Delete Rig` :❓
- Auto
  - `Auto Build`　:❓
- Rig
  - `Create Root` :
  - `Create Control` : 名称入力
    - `Side Label` : Center・Left・Rightどれかを選択
    - `Position` : Center to Geometry・Base of Geometry・World Centerどれかを選択
    - `Ctl Shape` : コントローラーシェイプ
    - `Create CTL` : コントローラ作成
  - Edit
    - PIvot
      - `Edit Pivot` :❓
      - `Set Pivot` :❓
      - `Re-Parent Pivot` :❓
    - Elements
      - `( - )` :❓
      - `( + )` :❓
      - `Select Affected` :
  - Auto Rig
    - `Surfix Rule` :名称ルール❓
    - `Auto Build`　:❓

- Extra Config
  - Root
    - `Root Name` : ルートの名称
  - Main Controls(World,Local,Global)
    - `Main controls in World Center` : ❓
    - `Use fix size` : ❓
    - `Fix Size` : サイズ
    - `Local Global Ctl Shape` :コントローラーシェイプ
    - `Create World Ctl` : Worldコントローラの作成をするか
    - `World Ctl Shape` :コントローラーシェイプ
  - Custom Sets
    - `Extra CTL Sets` : ❓

# Skin and Weights

## Copy Skin

## Select Skin Defomers

## Import Skin

スキンの読み込み

## Import Skin Pack

 ❓

## Export Skin

[Maya 2022 の mGear4.0.3 でのスキンのエクスポート エラー](http://forum.mgear-framework.com/t/solved-export-skin-error-with-mgear4-0-3-in-maya-2022/2967)

## Export Skin Pack Binary

## Export Skin Pack ASCII

## Get Names in gSkin File

## Import Deformer Weight Map

## Export Deformer Weight Map

# Rigbits

## AddNPO

- 選択した各オブジェクトの親としてトランスフォームを追加してローカルの値を０にリセット

---

## Gimmick Joints

- Add Blended Joint

>このジョイントは 2 つのジョイント間で 50% 回転

- Add Support Joint

>サポート ジョイントはブレンド ジョイントの下に作成され、変形を助けるように設計されています。通常、この種のジョイントは SDK などによっても駆動されます。

![Gimmick Joints](https://mgear4.readthedocs.io/en/latest/_images/gimmick_joints.png)

---

## MirrorControlsShape

- ❓

## Replace Shape

- 用意したカーブシェイプとctlを差し替える
![a](https://mgear4.readthedocs.io/en/latest/_images/replace_shape.gif)

---

## Match All Transform

- 位置合わせ、Maya標準機能と同じ

## Match Pos with BBox

- バウンディングボックスの中心に位置合わせ

## Align Ref Axis

- 頂点を複数選択して中心にロケータを作成
![a](https://mgear4.readthedocs.io/en/latest/_images/Align_ref_axis.gif)

---

## CTL as Parent

- 選択した各オブジェクトの親として、選択したシェイプのコントロールを作成します

## Ctl as Child

- 選択した各オブジェクトの子として、選択したシェイプのコントロールを作成します

---

## Duplicate Symmetrical

- 選択したオブジェクトとその子を複製してミラーリングします。
    これは、一部の軸のスケーリングを無効にし、値の一部を反転することによって行われます。
    これにより、ミラー動作が提供されます。一部の名前変更も処理します。例: _L から _R まで

---

## RBF Manager

- 足などのジョイントをドライバーに動かされるスカートコントローラをドリブン設定する  
[mGear 2.5: RBF Manager公式動画](https://www.youtube.com/watch?v=VyWCaE-YOwk&list=TLPQMTMwMjIwMjRaMuoKE3vewA&index=3)

[フォーラム：RBF Manager問題とフィードバック](http://forum.mgear-framework.com/t/rbf-manager-issues-and-feedback/3236
)
不安になるほどの長文。自分のMELのほうが安定してるかも。

## RBF Manager2

- RBF Managerの上位版と思われる

[Rbf.ファイルのインポートをカスタムステップに組み込む方法](http://forum.mgear-framework.com/t/rbf/917)

<details>
<summary>python</summary>
<div>

```python
import mgear.shifter.custom_step as cstp
import pymel.core as pm
from mgear.rigbits import rbf_io

class CustomShifterStep(cstp.customShifterMainStep):

    def __init__(self):
        self.name = "rbf_setup"

    def run(self, stepDict):

        rbfPath = THE_PATH_TO_YOUR_FILE

        rbf_io.importRBFs(rbfPath)
```

</div></details>

- １つのコントローラーで複数連動させる機能

## SDK Manager(BETA)

<img src="https://user-images.githubusercontent.com/1050212/74310737-30b5ea80-4db1-11ea-8f6b-bd1beeb97e09.png" width="40%">

[動画](https://www.youtube.com/watch?v=KmbUr_rjZ50)

- 不明

---

## Space Manager

<img src="http://forum.mgear-framework.com/uploads/default/optimized/2X/2/2e585f60990a75754c81c23a3f3e39e6849fc9a3_2_573x499.png" width="90%">

スペーススイッチ設定（親切り替え設定） プロップなどのマルチコンスト的な機能。
リグをビルド後使用する。

- `Import` : テーブルの読み込み
- `Export` : テーブルの保存
- `Run` : テーブル内の設定を実行

- テーブル
  - `Driven` : 子コントローラ　ノードを選択してセルを右クリックしてAdd Selectedすれば置換できる
  - `Drivers` : 複数の親ノード　ノードを複数選択してセルを右クリックしてAdd Selectedすれば置換できる
  - `Constraint Type` : 基本的にはparentConstraint 回転だけ追従したい時はorient
  - `MaintainOffset` : 現状の位置を維持して親子にするか、親ノードの位置に子がスナップするか。
  - `Top Menu Name` : アトリビュートにする名前。SpaceとかfollowParentとか任意
  - `Sub Menu Names` : アトリビュートのプルダウンに表示されるテキスト「,」で区切る
  - `Menu Type` : enum か float を選ぶ。enumでいいかも。floatだと各親分のウエイトアトリビュートが追加される。
  - `Custom UIhost` : アトリビュートが追加されるコントローラ

- `Add driven` : 子コントローラを選択して実行すると行が追加される
- `Add Driven and Drivers` : 子コントローラと親ノードを選択して実行すると行が追加される
- `Add UIhost` : 切り替えアトリビュートを追加するノードを選択して実行すると行が追加される
- `Remove Row` : 選択した行を消す
- SpaceMnagerを開き、子になるコントローラを選びAdddriven押す。
- 親となるジョイントやコントローラを複数選択してDriversを右クリックAddSelectする  

要検証：複数のコントローラを選択して指定レンジで指定スペースに切り替えながらベイクは可能か

---

## Space Jumper

 ❓

## Interpolate Transform

- ２つ選択したノードの 位置 回転 サイズ の平均値を持ったトランスフォームノードを作る。
- ～_INTER_～　というノードが作られる。各ノードの階層の影響なく平均値が取れる

## Connect Local SRT

- 動かされるノード　コントローラノードの順で選択して実行するとSRTがコネクトされる

---

## Spring

 ❓

## Rope

 ❓

---

## Channel Wrangler

 ❓

---

## Facial Rigger

- 顔のリグ

[フォーラム:フェイシャルリガー / eye_Mid_ctl / ミラー問題](http://forum.mgear-framework.com/t/facial-rigger-eye-mid-ctl-mirror/4020)

## Eyelid Rigger 2.0

- 目のリグ

---

## Proxy Geo

- 起動するとカプセルやボックス形状のShapeが作れる様子
- 簡易モデルやコリジョンで使うのかもしれない

## Proxy Slicer

 ❓

## Proxy Slicer Parenting

 ❓

---

# Animbits

 ❓

## Channel Master

<img src="https://www.mgear-framework.com/mgear_dist/_images/channel_master.png" width="30%">

[アニマブログ :リグコントローラーではないアトリビュートの操作経路を確保したい場合  
各項目の色変更や数値の範囲設定、順番変更等](https://www.studioanima.co.jp/Anigon-chat/2022/09/07/post-5770/)

 ❓

## Soft Tweaks

<img src="https://mgear4.readthedocs.io/en/latest/_images/softtweak.gif" width="60%">

- デフォーマー？
  ゲームでは使用しない機能

## Cache Manager

 ❓

## Human IK Mapper

- Human IK とmGearのコントローラの関連付けをしてモーション移植するための準備機能かと予想

## Space Recorder

- Record World Spaces
  - Record Buffer A B C : 選択したノードの動きをワールド空間で保存
- Apply World Spaces
  - Apply Buffer A B C : ❓
- Apply to Selection
  - Apply Sel Buffer A B C : 選択したノードに保存した動きをベイクする

## Smart Reset Attribute/SRT

SRT (選択したオブジェクトのスケール、回転、移動) をリセット0にします
![smartReset](https://mgear4.readthedocs.io/en/latest/_images/smartReset_hotkey.png)

## Spring Manager

[動画](https://www.youtube.com/watch?app=desktop&v=Zr_1KipElk0)

リグを入れた後springManagerで選択したFKコントローラの揺れをパラメーターでつくれる
キーをつけながら調整可能。動きが確定したらベイクする。

- menu
  - `Bake` ベイク
  - `Delete` スプリング設定をctlから消す
  - `Presets` スプリング設定を保存できる。
  - `Select`
- Directions 基本的にX方向が先端
- SpringParameters
  - `Total Intensity` スプリングの強さ ❓
  - `Spring Rig Scale`  ❓
  - `Translational Intensity` 移動もスプリングで動く ❓
  - `Translational Damping`  ❓
  - `Translational Stiffness`  ❓
  - `Rotational Intensity`  ❓
  - `Rotational Damping`  ❓
  - `Rotational Stiffness`  ❓
- presets
  
- [あかさき氏](https://github.com/akasaki1211)の
boneDynamicsNodeがコリジョンも使えてよさそう

## Bake Spring nodes

 ❓

## Clear Baked Spring nodes(Shifter Component)

 ❓

# CFXbits

 ❓

# Anim Picker

  ピッカー

[フォーラム：相対パス?](http://forum.mgear-framework.com/t/animpicker-relative-path/1773/13)

## Edit Anim Picker

 ❓

## Enable opacity passthrough

 ❓

# Synoptic

シノプティックは廃止され、サポートされなくなりました。したがって、問題が発生しても不思議ではないと思います。

# Flex

 ❓

# Utilities

- Reload : Mgearのリロード
<font color="Red">QT UI は正しくリロードされないため、Maya を再起動することをお勧めします。</font>
- Create mGear Hotkeys
- Enable mGear file drop

# Help

- [Web 公式](https://www.mgear-framework.com/)
- [Forum](http://forum.mgear-framework.com/)
- [Documentation 公式マニュアル](https://mgear4.readthedocs.io/en/latest/)

# mGer Viewport Menu

これを有効にすると、コントロールが選択されている場合に、Maya のデフォルトの右クリック メニューが置き換えられます。他のものが選択されている場合、右クリック メニューは通常どおり機能します。

このメニューは選択したコントローラの種類に基づいて変化します。ほとんどのコントローラーとホスト コントローラーでは、次のようになります。  
![menu](https://mgear4.readthedocs.io/en/latest/_images/quickstart_viewportMenu.png)  

## Controller Viewport Menuコントローラービューポートメニュー

- Select host: このボディパーツのホスト コントローラーを選択します。そこから IK/FK ブレンディングなどを行うことができます。

- `Select child controls`: 現在のコントロールの下にある子コントロールを選択します。
- `Reset`: 選択したコントロールをデフォルトの位置にリセットします (バインド ポーズと同様)
- `Reset all below`: まず子コントロールを選択し、次に上記と同様にリセットします。
- `Reset translate/rotate/scale`: 上記の通常のリセットと同じですが、平行移動、回転、スケールに特化しています。

- `Mirror` : 選択したコントローラのポーズをミラーリングします。
- `Mirror all below`: ミラーと同じですが、すべての子コントロールが対象です。
- `Flip` : 選択したコントローラーのポーズを左右反転します。
- `Flip all below`: fliip と同じですが、すべての子コントロールが対象です
- `Rotate Order switch`: アニメーションをそのまま維持しながら、回転順序を変更します。

## Host Viewport Menuホスト ビューポート メニュー

- `Switch arm/leg to Ik/Fk`: ポーズを維持したまま、IK と FK を切り替えます。

- `Switch arm/leg to Ik/Fk + Key`: ポーズを維持したまま、IK と FK を切り替えます。切り替えの前後にキーフレームを追加します。

- `Range switch`：タイムスライダー範囲内のキーをIk/Fk に切り替える

- `Parent [various]` : IK コントローラなどのさまざまなコントローラのドロップダウン メニューで、そのコントローラが従う空間を設定します。たとえば、肩、体、またはグローバル コントローラに続いて腕の IK コントローラを切り替えることができます。コンポーネントにすでにキーフレームがある場合は、切り替えの前後にキーフレームが追加されます。

- `Select all controls`: リグのすべてのコントローラーを選択します。

- `Keyframe child controls`: 現在の選択範囲のすべての子コントロールをキーフレームに設定します。

# リグモジュールの追加

[3DCGMeetup08_MayaRigSystem_mGear](https://www.slideshare.net/slideshow/3dcgmeetup08mayarigsystemmgear/54702295)

(mgearバージョンが古い)
![a](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-34-2048.jpg?cb=1666707212)
![b](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-35-2048.jpg?cb=1666707212)

# ガイドのカスタマイズ

(mgearバージョンが古い)
![a](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-37-2048.jpg?cb=1666707212)

![a](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-39-2048.jpg?cb=1666707212)

![a](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-42-2048.jpg?cb=1666707212)

![a](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-43-2048.jpg?cb=1666707212)

![a](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-44-2048.jpg?cb=1666707212)

![a](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-45-2048.jpg?cb=1666707212)

![a](https://image.slidesharecdn.com/3dcgmeetup08ueta-151103185554-lva1-app6892/75/3dcgmeetup08mayarigsystemmgear-46-2048.jpg?cb=1666707212)

# カスタム コンポーネント

[フォーラム：カスタム コンポーネントのガイド/チュートリアル?](http://forum.mgear-framework.com/t/guides-tutorials-for-custom-components/1352)

[動画　mGear: Rigging Framework Live Stream](https://www.youtube.com/watch?v=2Q_nfNZkNlo)

## 上記フォーラムから引用

>カスタム コンポーネントを作成する手順をまとめ  
シフター クラシック コンポーネント フォルダーで、作成したいコンポーネントに最も似ているコンポーネントを選択します。  
たとえば、独自の arm モジュールを作成し、そのコンポーネントを複製して、ニーズに合った名前に変更したいとします。まず、そのコンポーネント (「rynas_arm_01/__ init__.py」) の __ init__.py で何が起こっているかを編集することから始めます。  
ほとんどのコアライブラリには必要な機能が多数含まれており、独自の機能を簡単に追加できることがわかります。他のコンポーネントを読んで、何を探しているのかを把握し、モジュールがどのように機能するかに慣れてください。  
最初のコンポーネントは、コード、最適化、「ベスト プラクティス」の点で芸術作品とは言えません。少なくとも、慣れ始めるまでは私にとってはそうでした。  
さらに行う必要があることがいくつかあります。たとえば、独自の設定をコンポーネントに追加し、それらの設定をuiに公開したい場合は、 qtなどで ui ファイルを編集する必要があります。デザイナー(Maya がまだインストールされていない場合は、Maya に付属しています) は「C:\Program Files\Autodesk\MayaVersion\bin\designer.exe」にあります。  
もう一度、他のコンポーネントが UI で何を行っているか、およびコンポーネント間の接続がどのように確立されているかを参照してください。ui ファイルを py ファイルに再構築する必要があります。
これを支援するものがコアライブラリにすでにあり、それは pyqt モジュール内にあり、ui2py と呼ばれます。checkbox12 という名前のものを見つけて接続しようとすると、非常にイライラするデバッグが発生する可能性があるため、デザイナーでのウィジェットオブジェクトの名前には注意してください。以下のワークフローのヒントのコード例。  
ワークフローのヒント:  
コンポーネントを素早くリロードするためのシェルフ ボタンがあります。このオプションはシフター メニューからも表示され、数回クリックするだけで済みます。  

```python
from mgear import shifter
shifter.reloadComponents()
```

>また、それを実行してリロードしてビルドすることもできるので、それが使用したいワークフローである場合は、スクリプトの最後にこの行を追加します。これは、によって私に示されました。  
shifter.Rig().buildFromSelection()  
.ui ファイルから .py ファイルを構築します。  

```python
from mgear import core  
core.pyqt.ui2py("path\to\your\ui\file")  
```

>まずはcontrol_01モジュールを試して、それを拡張することから始めます。これが最も簡単に複製できます。複雑なことにいきなり取り組むと、非常にイライラする可能性があります。少なくともこれは私自身の経験で発見したことです。
