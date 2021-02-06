# Mouse and Keystroke Automation with Grasshopper C#

## 環境

= Windows OS
- Rhinoceros + Grasshopper (Windows Version)

## このハンズオンの目的

Windowsのプログラムで手軽にオートメーションできるようになるのが目的です。基本的にプログラムの自動化を行う場合、利用しているプログラムのSDKを利用して用意されている関数を利用することが主流かと思います。例えばRhinoの場合はRhinoCommonSDKを利用して自動化のプログラムを書くことができます。

ただ、そのプログラムのSDKに関してあまり知識がない状態だと学習コストはそれなりにかかります。マウスやキーボードでやっている作業をくり返すだけでいいんだけど、、、そんな場面を想定しているのが今回のハンズオンです。マウスやキーボードのキーストロークを簡単なプログラムで再現してしまおうというのが今回の目的です。

これを使うことで、UIとして用意されている機能にSDKの知識なくアクセスできるようになり、あたかも人間の手の動きを再現するような形で自動化のプログラムを作れるようになります。特にSDKが公開されていないようなプログラムでは大きな威力を発揮するでしょう。

とはいえ、もしSDKが公開されているなら、そちらをまず使ってみることをおすすめします。UI上には現れない便利なオプションも提供しいていることが多いですし、マウスやキーを使ったUI操作はプログラムアップデート時にUIが変更されて使えなくなる危険性もあったり、あまり安定したオートメーションとはいえないです。

## 基本の流れ

WindowsにはC++で書かれたuser32.dllというライブラリが存在していて、そのライブラリの中にある関数を使うことで、マウスクリックやキーストロークをコントロールすることができます。この関数はC#からInteropServiceというものを介することで利用することができます。これはつまり、WindowsでC#が使える環境であれば、どんなアプリケーション上でもマウスやキーボードのコントロールができるということになります。

今回は、これをGrasshopper上にて使ってみますが、同じことはC#のコンソールアプリでももちろん使えますし、RevitやUnityやAutoCADといったツールでももちろん使えることができます。

C#からC++のコードを呼ぶ部分はモジュールとしてまとめているので、その部分はコピペで持って行って利用します（この部分は長くて複雑ですし、ここを一からやる意味はないと考えて）。今回はどちらかといえば、それをどう利用すればいいのかというハンズオンとなります。なので、最低限のC#の知識があればついてこられるようになっているかと思います。たぶん。

## モジュール

mousekeystroke.cs にマウスやキーのコントロールに必要なコードをまとめています。特にこの中に入っている関数の中で重要なものをピックアップします。

### マウスクリック
**void ClickOnPoint(IntPtr wndHandle, POINT clientPoint, MOUSETYPE mouseType, bool right = false, bool dblclick = false)**

- IntPtr wndHandl: ウィンドウのハンドラー（ウィンドウを示すポインター）。指定のウィンドウ右上からの相対位置を利用してマウス位置を指定したい場合はこれを使う。スクリーンの左上から指定する場合はIntPtr.Zeroを入力する。
- POINT clientPoint: マウスクリックする際のウィンドウあるいはスクリーンの左上からの位置。
- MOUSETYPE mouseType: マウスクリックのタイプ（MOUSETYPE.MOUSEDOWN: マウスダウン, MOUSETYPE.MOUSEUP: マウスアップ, MOUSETYPE.CLICK: マウスクリック）
- bool right: 右クリックかどうか
- bool dblclick: ダブルクリックかどうか（MOUSETYPE.MOUSECLICKの時だけ有効）

### マウスドラッグ
**void Drag(IntPtr wndHandle, POINT srcPoint, POINT tgtPoint, bool right = false)**

- IntPtr wndHandl: ウィンドウのハンドラー（ウィンドウを示すポインター）。指定のウィンドウ右上からの相対位置を利用してマウス位置を指定したい場合はこれを使う。スクリーンの左上から指定する場合はIntPtr.Zeroを入力する。
- POINT srcPoint: ドラッグ開始時のスクリーン、あるいはウィンドウ左上からの位置
- POINT tgtPoint: ドラッグ終了時のスクリーン、あるいはウィンドウ左上からの位置
- bool right: 右ドラッグかどうか

### キー入力（文字）
**void SendKey(string a)**

- string a: キー入力したい文字列

### キー入力（装飾文字）
**void SendKey(ScanCodeShort a)**
**void SendKey(ScanCodeShort[] a)**

- ScanCodeShort a: 装飾文字のキーコード。ScanCodeShort.と入力すると使える装飾文字の候補のリストが現れる。例えばリターンキーの場合は　ScanCodeShort.RETURN　。
- ScanCodeShort[] a: 装飾文字のキーコードの配列。例えばControlキー+Aみたいなキーストロークを使いたい場合は　new ScanCodeShort[]{ScanCodeShort.CONTROL, ScanCodeShort.KEY_A}　をインプットに使います。

この四つの関数を使えばたいていのマウス・キー操作はできるようになります。もっと細かいことをしたい（例えばミドルクリックを使いたい）などあれば、ソースコードをちょっといじることでできるようになります。


## 作法

実際にGrasshopper上でマウスクリックやドラッグ、キー入力をしていて分かったのは、プログラムによってはマウスやキーストローク動作の間に多少の間を置く必要がある場合ということです。例えばドラッグの場合ドラッグ開始と終了の間に最低0.2秒の間隔がないと、GHのウィンドウ上ではうまくドラッグが機能しないことが分かりました。

ということで、プログラム上でこの間を再現する必要があります。C#ではSystem.Threading.Tasksの名前空間を利用することで、スレッド処理（非同期処理）が使えるようになるので、今回はこちらを利用します。

例えば、スクリーンの左上から(50,50)の位置をクリックして、その0.2秒後に(100,100)の位置をクリックしたい場合は次のように書きます。

```csharp
async void SampleClick(){
    ClickOnPoint(IntPtr.Zero, new POINT(50,50), MOUSETYPE.MOUSECLICK, false, false);
    await Task.Delay(200);
    ClickOnPoint(IntPtr.Zero, new POINT(50,50), MOUSETYPE.MOUSECLICK, false, false);
}
```

関数を定義しているvoidの前にasyncをつけます。そのうえで間を置きたいところでawait Task.Delay()を使います。()の中にはミリセカンドで間を置きたい時間を整数で入力します。1000ミリセカンドで１秒です。個人的に色々試した結果、異なるキーストロークやマウスクリックの間には最低200ミリセカンド程間があったほうが良いようです（少なくともGHでは）。

## Windowのハンドラー？

ウィンドウのハンドラーというのは、ウィンドウを示すポインター（住所）です。この情報があるとウィンドウに関わる様々な情報（位置とかサイズとか）を得ることができます。特定のプログラムにおけるこのウィンドウのハンドラーの取得方法に関してはプログラムごとに異なるので、例えばRevitの特定のウィンドウの左上からの相対位置を使いたい場合はそのハンドラーを取得する必要があります。

Grasshopperウィンドウのハンドラーに関してはwindowhandler.cs内に書かれているコードを使います。例によって長いコードなのでこれも基本はコピペで持っていきます。他のプログラムで使いたい場合は次のコードを改変することで使えるようになると思います。

```csharp
public class GrasshopperWindowHandler(){
    IntPtr GetGHWIndowHandle(GH_Document gh){
        // 今開いているすべてのウィンドウの情報を取得
        List<WindowInformation> windowListExtended = WindowList.GetAllWindowsExtendedInfo();
        // Grasshopperのウィンドウ名とマッチするウィンドウを探し出す。
        WindowInformation wi = windowListExtended.Find(
      w => w.Caption.StartsWith((string) ("Grasshopper - " + gh.DisplayName.ToString()))
      );

        // IntPtrとしてアウトプット。整数としてアウトプットする場合は wi.Handle.ToInt64() を使う。
        return wi.Handle;
    }
}
```

特にこの中の 
```csharp
WindowInformation wi = windowListExtended.Find( w => w.Caption.StartsWith((string) ("Grasshopper - " + gh.DisplayName.ToString())));
```
"Grasshopper - " + gh.DisplayName.ToString()"の部分を使いたいウィンドウの名前にかえることで取得できるようになると思います。名前がわからない場合はこの手法は使えないので別の方法で（SDKを利用するなどして）取得してください。


## 利用例

- ウィンドウを最小化し、スクリーン左上に移動する。
```csharp
static async void MoveAndHideWindow(int h, int d){
    ClickOnPoint((IntPtr) h, new POINT(20, -20), MOUSETYPE.MOUSECLICK, false, true);

    await Task.Delay(d);

    ClickOnPoint((IntPtr) h, new POINT(20, -20), MOUSETYPE.MOUSEDOWN, false, false);

    await Task.Delay(d);

    ClickOnPoint(IntPtr.Zero, new POINT(20, 20), MOUSETYPE.MOUSEUP, false, false);

}
```

- すべてのコンポーネントをDisableにする。
```csharp
static async void DisableAllComponents(int h, int d){
    SendKey(new ScanCodeShort[]{ScanCodeShort.CONTROL, ScanCodeShort.KEY_A});

    await Task.Delay(d);

    ClickOnPoint((IntPtr) h, new POINT(20, 250), MOUSETYPE.MOUSECLICK, true);
    await Task.Delay(d);

    for(int i = 0; i < 5; i++){
        SendKey(ScanCodeShort.DOWN);
        await Task.Delay(d);
    }


    SendKey(ScanCodeShort.RETURN);
}
```

- メニュー項目にあるExport Hi-Res Imageを自動で呼び出し、指定のファイル名で高解像度画面キャプチャを書き出したい。

```csharp
static async void SaveImage(int h, string f, int d){
    var p = new POINT(20, 20);

    ClickOnPoint((IntPtr) h, p, MOUSETYPE.MOUSECLICK, false, true);
    await Task.Delay(d);

    p.y = 225;
    ClickOnPoint((IntPtr) h, p, MOUSETYPE.MOUSECLICK);
    await Task.Delay(d);

    SendKey(ScanCodeShort.TAB);
    await Task.Delay(d);
    SendKey(ScanCodeShort.TAB);
    await Task.Delay(d);

    SendKey(f);
    await Task.Delay(d);

    for(int i = 0; i < 5; i++){
        SendKey(ScanCodeShort.TAB);
        await Task.Delay(d);
    }

    SendKey(ScanCodeShort.RETURN);

}
```
