# WPF=Windows Presentation Foundation

ベクターベースのレンダリング エンジンを使用し、最新のグラフィックス ハードウェアを活用するために構築された UI フレームワーク。

WPF公式フレームワークには、実装のための仕組みは最低限しか組み込まれていない。
各自で好きにカスタマずして使えということらしい。
一から作っても良いが、すでにいくつかの実績あるヘルパーフレームワークが出ているので、そちらを使うのが良い。

* Prism 元々 Microsoft の patterns & practices で開発された複合アプリケーションを作成するためのフレームワーク
* Livet　国産の MVVM フレームワーク　2019、ReactiveProperty のメンテナーがメンテを引きついだ
* MVVM light toolkit　？
[https://elf-mission.net/programming/wpf/getting-started-2020/step02/](https://elf-mission.net/programming/wpf/getting-started-2020/step02/)からの情報

## Prism（ヘルパーフレームワーク）
こちらを利用する。
メニューバーから`ツール＞拡張機能と更新プログラム`を選び、`Prism Template Pack`をインストールする。
インストールが完了するとPrismのプロジェクトテンプレートが選択できるようになる。
必要なら`拡張機能>拡張機能の管理`で「機能拡張の自動更新」は止める。

* [公式サンプル](https://github.com/PrismLibrary/Prism-Samples-Wpf)
  Prism が提供する個々の機能を簡単に理解することができる優秀なサンプル。良い。

[https://elf-mission.net/programming/wpf/getting-started-2020/step03/](https://elf-mission.net/programming/wpf/getting-started-2020/step03/)からの情報

### 新規プロジェクト
* プロジェクトテンプレートにはPrism Blank App (WPF)を選ぶと良い
* そのさい、DIコンテナに何を使うか確認されるので、Unityを選択する
* プロジェクトができたら、ソリューションコンテキストメニューから`NuGetパッケージの復元`を選択しPrismパッケージを復元しておく

### Viewの構造

* ViewオブジェクトはContentオブジェクトを1つ持つ
* ContentにはViewの内容部となるViewを指定する
* ボタンやテキストボックスといったコントロール系のViewと、それらのView群をまとめ、レイアウト配置するコンテナ系がある
* コンテナ系：Grid、StackPanel
* コントロール系：Button、TextBox
* 通常WindowのContentにはコンテナ系のView（Grid、StackPanel）を指定し、そのViewの管理するView群としてコントロール系Viewを配置する

#### Prismでの拡張
ContentControlに`prism:RegionManager.RegionName`プロパティが提供され、設定した名前を配置場所として、ダイナミックに表示するViewを変更できる。
```
<ContentControl prism:RegionManager.RegionName="ContentRegion" />
```
配置するViewはあらかじめ登録しておく必要がある。
Appの場合
```
        public void OnInitialized(IContainerProvider containerProvider)
        {
            var regionMan = containerProvider.Resolve<IRegionManager>();
            regionMan.RegisterViewWithRegion("ContentRegion", typeof(InsertView));
        }
```
ViewModelの場合
```
        public ViewModel(IRegionManager regMan)
        {
            regMan.RegisterViewWithRegion("ContentRegion", typeof(InsertView));
        }
```

## ReactiveProperty （ヘルパーライブラリ）
データバインディングに利用する。
NuGetでインストールする。reactivepropertyという名前のパッケージ。

### ReactiveProperty
必要最小限機能でパフォーマンスを上げたReactivePropertySlimもある。
int型のプロパティをバインドする場合
```
ReactiveProperty<int> rp {get} = new(0);
```
で用意し、XAML側でrp.Valueをバインドする。
これでView、ViewModelのバインドが完成する。
ViewModelとModel側もバインドしたい場合はModel側も
```
ReactiveProperty<int> rp {get} = new(0);
```
とし、ViewModel側はコンストラクタで
```
ReactiveProperty<int> rp {get}
ViewModel(Model model)
{
this.rp = model.rp.ToReactivePropertyAsSynchronized(x => x.Value);
```
とバインドする。
このように、宣言ではなくコンストラクタで作成した場合、かつ、Subscribなどを利用した場合、メモリリークが発生する。
これを防止するために、IDestructible、CompositeDisposableを使った廃棄処理を追加する。
 ```
public class ViewModel : BindableBase, IDestructible
{
        private CompositeDisposable disposables = new CompositeDisposable();
        ReactiveProperty<int> rp {get}
        ViewModel(Model model)
        {
                rp = model.rp.ToReactivePropertyAsSynchronized(x => x.Value).AddTo(disposables);
        ・・・

        public void Destroy()
            => this.disposables.Dispose();
}
```
### ReactiveCommand
必要最小限機能でパフォーマンスを上げたReactiveCommandSlimもある。
```
public ReactiveCommand cmd { get; }

ViewModel()
{
            cmd = new ReactiveCommand().WithSubscribe(() => {...}).AddTo(disposables);
```
ICommandのCanExecute への対応は IObservable<bool> から
```
CompositeDisposable disposables = new CompositeDisposable();
ReactiveProperty<int?> Id {get}
ReactivePropertySlim<bool> hasId;
public ReactiveCommand cmd { get; }
public ReactiveProperty<string> NowDateTime { get; }

ViewModel(Model model)
{
hasId = new ReactiveProperty<bool>(false).AddTo(this.disposables);
Id = model.Id.ToReactivePropertyAsSynchronized(x => x.Value).WithSubscribe(v => hasId.Value = v.HasValue).AddTo(this.disposables);
NowDateTime = new ReactiveProperty<string>().AddTo(this.disposables);
cmd = hasId.ToReactiveCommand().WithSubscribe(() => {
NowDateTime.Value = DateTime.Now.ToString("yyyy/MM/dd HH:mm:ss");
}).AddTo(this.disposables);
```
とする。
ReactiveCommand は IObservable<bool> も継承しているので、次のような連携もできる。
この場合、cmdが実行されるたびにValueが更新されるReactivePropertyが作成される。
```
public ReactiveCommand cmd { get; }
public ReactiveProperty<string> NowDateTime { get; }
・・・
cmd = new ReactiveCommand().AddTo(this.disposables);
NowDateTime = cmd.Select(_ => DateTime.Now.ToString("yyyy/MM/dd HH:mm:ss")).ToReactiveProperty().AddTo(disposables);
```
### CommandParameter
同じReactiveCommandをXAMLで複数のButtonとバンドする場合、CommandParameterを使い、ボタンを識別するためのパラメータを送ることができる。
 ```
 <Button Command="{Binding PastTimeClick}" CommandParameter="10"/>
 ```
これでstringを受け取ることができる。
```
public ReactiveCommand<string> cmd { get; }
...
cmd = new ReactiveCommand<string>().WithSubscribe((s) => {
                ...
            });
}).AddTo(this.disposables);
```
### AsyncReactiveCommand 
ReactiveCommandではボタンの２度押しなどの対応が面倒。
代わりにAsyncReactiveCommand を使う。
本来の目的は非同期処理が終わるまで待つというものだが、
```
            cmd = new AsyncReactiveCommand().WithSubscribe(() => {
Task.Run(() =>
            {
                ...
            });
}).AddTo(this.disposables);
```
## ReactiveProperty.WPF （ヘルパーライブラリ）
データバインディングに利用する。
NuGetでインストールする。reactiveproperty.WPF。

### 画面LoadといったイベントをCommandで受け取る
EventToReactiveCommandで変換。
```
    <bh:Interaction.Triggers>
        <bh:EventTrigger EventName="Loaded">
            <rp:EventToReactiveCommand Command="{Binding ViewLoaded}" />
        </bh:EventTrigger>
    </bh:Interaction.Triggers>
```
EventArgs も取得するならXAMLで
```
        <bh:EventTrigger EventName="MouseMove">
            <rp:EventToReactiveProperty ReactiveProperty="{Binding MousePoint}" />
            <rp:EventToReactiveCommand Command="{Binding MouseMove}" />
        </bh:EventTrigger>
```
ViewModelで
```
        public ReactiveCommand<MouseEventArgs> MouseMove { get; }
public ReactivePropertySlim<string> CurrentMousePoint { get; }
・・・
CurrentMousePoint = new ReactivePropertySlim<string>(string.Empty).AddTo(this.disposables);
MouseMove = new ReactiveCommand<MouseEventArgs>()
                .WithSubscribe(args =>
{
 var pos = args.GetPosition(null);
            this.CurrentMousePoint.Value = $"(X: {pos.X} Y: {pos.Y})";
        }
}
)
                .AddTo(this.disposables);
                ```
## Xaml.Behaviors.Wpf（ヘルパーライブラリ）
Interaction.Triggersを使いたい場合、NuGetでインストールする。Xaml.Behaviors.Wpfという名前のパッケージ。

## ライブラリとフレームワーク
いずれも他のアプリでの再利用を前提にしたソフトウエア一式を指す。
おおざっぱにライブラリと言ってもかまわない。
ライブラリの中で、提供されるクラスやインスタンス、メソッド等が、相互に関係しながら動作する形態のライブラリであることを強調したい時にフレームワークと呼ぶ。

