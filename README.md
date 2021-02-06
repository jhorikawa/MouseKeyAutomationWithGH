# Mouse and Keystroke Automation with Grasshopper C#

## 環境

- Windows OS
- Rhinoceros + Grasshopper (Windows Version)

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

関数を定義しているvoidの前にasyncをつけます。そのうえで間を置きたいところでawait Task.Delay()を使います。()の中にはミリセカンドで間を置きたい時間を整数で入力します。1000ミリセカンドで１秒です。

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

- メニュー項目にあるExport Hi-Res Imageを自動で呼び出し、指定のファイル名で高解像度画面キャプチャを書き出したい。

```csharp
static async void SaveImage(int h, string f){
    var p = new POINT(20, 20);

    ClickOnPoint((IntPtr) h, p, MOUSETYPE.MOUSECLICK, false, true);
    await Task.Delay(400);

    p.y = 225;
    ClickOnPoint((IntPtr) h, p, MOUSETYPE.MOUSECLICK);
    await Task.Delay(400);

    SendKey(ScanCodeShort.TAB);
    await Task.Delay(400);
    SendKey(ScanCodeShort.TAB);
    await Task.Delay(400);

    SendKey(f);
    await Task.Delay(400);

    for(int i = 0; i < 5; i++){
        SendKey(ScanCodeShort.TAB);
        await Task.Delay(400);
    }

    SendKey(ScanCodeShort.RETURN);

}
```
