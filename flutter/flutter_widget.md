- [TOP](./README.md)
- [このメモ・独自表記について](../README.md)


# WIP: 随時更新
* このメモは執筆中のため随時更新 

# ドキュメント
* ウィジェットカタログ
    * https://docs.flutter.dev/ui/widgets
* ウィジェットの基礎（ベーシックなウィジェット、ウィジェットのビルドの仕組み）
    * https://docs.flutter.dev/development/ui/widgets-intro

# ウィジェットにおける責務
* Flutterでは可能な限り継承ではなく、Compositionを使っている。
* たとえば"padding"というものをTextウィジェットに持たせるのではなく、Paddingというウィジェットを使い責務を分けている。
* IMO
    * 内部実装では継承はかなり多い。
    * ただ、責務が異なる機能はCompositionの使用、同じ責務のクラスの拡張は継承という使い分けがされている。

# ウィジェットは@immutable
* Widgetクラスは@immutableになっている
    ```
    @immutable
    abstract class Widget extends DiagnosticableTree {
        //...
    }
    ```
* したがってconstコンストラクタではない場合はアナライザによって警告が表示される。
    * https://dart.dev/tools/diagnostic-messages#must_be_immutable

    ```
    class A extends StatelessWidget {
    A({super.key, this.a = true});
    bool a;

    @override
    Widget build(BuildContext c) {
        return const Text("a");
    }
    }
    // This class (or a class that this class inherits from) is marked as '@immutable', but one or more of its instance fields aren't final: A.adartmust_be_immutable
    ```

# ステートレス、ステートフルなウィジェット
* ステートレスなウィジェット
    * Icon, IconButton, Text 等
    * Stateless widgets は StatelessWidget のサブクラスである。
* ステートフルなウィジェット
    * Checkbox, Radio, Slider, InkWell, Form, TextField 等
    * Stateful widgets は StatefulWidget のサブクラスである。


# レイアウトの基本的なウィジェット
* Align
* Center
* SizedBox
    * SizedBox.shrink()
        * https://api.flutter.dev/flutter/widgets/SizedBox/SizedBox.shrink.html
        * 親が許す限り限りなく小さくなる
* Spacer
* Divider
* Padding
* Border
* TabBar, TabBarView
* Table, DataTable
* Card
## Container
* https://api.flutter.dev/flutter/widgets/Container-class.html
* サイズ
    * Containerはそれぞれ独自のレイアウト動作を持つウィジェットを多数組み合わせているため、レイアウトの動作が複雑となる。
        * 要約: alignmentを尊重し、子に合わせてサイズを調整し、幅、高さ、制約を尊重し、親に合わせて拡大し、できるだけ小さくしようとする。
        * 関連する要素 child, alignment, width, height, constraints, 親与えられるconstraintes
        * 詳細はAPIドキュメントを参照
* 主なプロパティ
    * width, height
    * decoration
    * padding, margin
    * alignment
    * constraints
* Container + Rowで全画面とする
    ```dart
    Container(
        height: double.infinity,
        decoration: const BoxDecoration(
            color: Color.fromRGBO(0, 0, 0, 0.6),
        ),
        child: const Row(
            mainAxisSize: MainAxisSize.max,
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
                CupertinoActivityIndicator(radius: 30.0, color: Colors.black)
            ],
        ),
    )
    ```
* Container + Rowで全画面とする
    ```dart
    Container(
        width: double.infinity,
        decoration: const BoxDecoration(
            color: Color.fromRGBO(0, 0, 0, 0.6),
        ),
        child: const Column(
            mainAxisSize: MainAxisSize.max,
            mainAxisAlignment: MainAxisAlignment.center,
            children: <Widget>[
                CupertinoActivityIndicator(radius: 30.0, color: Colors.black)
            ],
        ),
    )
    ```
## Flex派生ウィジェット(Column, Row)とFlexible派生ウィジェット
* Column, Rowは、子のRenderObject.parentData.flexの値によって子へ渡すConstraintsと自身のSizeの設定内容が異なる。
    * Flexibleは子孫のこのflexの値を上書きすることができる
* クラス図と内部の動作の仕組み
    * [コードリーディング](./flutter_arch_code_reading.md) を参照
* Column
    * レイアウト
        * https://api.flutter.dev/flutter/widgets/Column-class.html
        * flexが設定されていないまたは0の子については、無制限の高さの制約となる。幅の制約は親から受け取った制約(incoming constraints)に従う
            * crossAxisAlignmentにstretchが設定されている場合、incoming constraintsにおいて最大幅となるtight constraintsを制約にする。
        * 残りの垂直スペースを子のflexに応じて割り当てる。例えばflexが2.0の場合はflexが1.0の子の2倍のスペースが割当される。
        * 整列
            * mainAxisAlignment はデフォルトでstart
            * crossAxisAlignment はデフォルトでcenter
        * Colomnの幅は子のうち、最大の幅となったものの幅となる。
        * Columnの高さはmainAxisSizeによって決まり、デフォルトは取りうる最大の大きさになろうとする。
    ```
    Center getCenterlingColumn(List<Widget> children, [double padding = 32.0]) {
        return Center(
            child: Padding(
                padding: EdgeInsets.symmetric(horizontal: padding),
                child: Column(
                    mainAxisSize: MainAxisSize.min,
                    children: children,
                ),
            ),
        );
    }
    ```
* Row
    * https://api.flutter.dev/flutter/widgets/Row-class.html
* Flexible, Expanded
    * https://api.flutter.dev/flutter/widgets/Flexible-class.html
    * Row, Column, またはFlexの子孫のflexを制御するウィジェット
    * ParentDataWidgetの派生クラスであり、内部処理としてParentDataWidget.applyParentData()によって最も近い子孫のRenderObjectのparentData.flexやfixを上書きする。
        * 内部処理の詳細はコードリーディングのメモを参照
    * FlexibleウィジェットはRow, Column, またはFlexの子孫である必要がある。
## Stack, Positioned
* https://api.flutter.dev/flutter/widgets/Positioned-class.html
* 例えば、Stack内で「右下」に配置したい場合に、下記のようにAlignを利用しても期待した動作とはならない。
    ```
    Stack
        Container
            ...
        Align// Expandedでラップしたとしても変わらない。
            alignment: Alignment.bottomRight
            child: ...
    ```
* Stack内部で指定の位置に配置したい場合はPositionedを利用する。
## 制約
* ConstrainedBox
    * 子に追加の制約を課すウィジェット
    ```
    ConstrainedBox(
        constraints: BoxConstraints.tight(const Size(200.0, 150.0)),
        //constraints: const BoxConstraints.expand(),
        child: const ColoredBox(color: Colors.red),
    )
    ```
* UnconstrainedBox
* OverflowBox
* LimitedBox
* FittedBox
## その他(TODO)




# material
* Scaffold
    * Scaffoldはよく使われるhelpfulなwidget
    * 以下を提供する
        * a default color
        * has API for 
            * adding drawers, 
            * snack bars, 
            * bottom sheets.
    * 主なプロパティ
        * appBar, drawer, body
* AppBar
    * 主なプロパティ
        * title, bottom, leading
    * サイズの指定したい場合はPreferredSizeWidgetでラップする必要がある。
    * ただし、PreferredSizeWidget.preferredSizeは AppBar.bottomで指定したウィジェットの高さも含める必要がある
## ダイアログ
* showDialog
    * https://api.flutter.dev/flutter/material/showDialog.html
* Dialog
* AlertDialog
* SimpleDialog
## スナックバー
* Snackbar
    ```
    ScaffoldMessenger.of(context)
    ..hideCurrentSnackBar()
    ..showSnackBar(
        SnackBar(
            content: Text("message"),
            duration: const Duration(milliseconds: 4000),
        ),
    );
    ```



# Builder
* https://api.flutter.dev/flutter/widgets/Builder-class.html
* StatelessWidget サブクラスを定義するためのインラインの代替手段




# テキスト関連
* Text
    * https://api.flutter.dev/flutter/widgets/Text-class.html
    * レイアウト
        * 制約がない場合は1行で表示される。
            * 「折り返すポイントが無い」状態のため、softWrapやmaxLinesの設定有無にかかわらず1行となる。
        * 例
            * ListViewやRowやColumn直下のTextは1行となる。
            * FlexibleやExpanded、SizedBox等でラップすることで制約によって複数行で表示が可能となる。 
    * style
        * 省略すると、テキストは最も近いDefaultTextStyleのスタイルを使用
        * 指定されたスタイルの TextStyle.inheritプロパティが true (デフォルト) の場合、指定されたスタイルは最も近いDefaultTextStyleとマージされる
    * softWrap
        * デフォルトの動作では折り返す。
        * falseにすると折り返しをしなくなる。
    * maxLines
        * 指定した場合は複数行で表示される。
        * softWrapと併用できない
    * overflow
        * テキストが幅を超えた場合のふるまい
        * softWrap: falseや、maxLinesを指定した際に利用するパラメータとなる
* RichText
    * text属性にInlineSpan派生クラスを指定する
* InlineSpan派生クラス
    * TextSpan
    * WidgetSpan



# フォーム関連
* https://docs.flutter.dev/cookbook/forms/validation
* Form/FormState
    * 複数のフォーム フィールドをグループ化して検証するためのコンテナーとして機能
    * keyにGlobakKeyを指定することで_formKey.currentState で FormStateを取得できる。
    * FormStateのAPIで入力内容が検証などできる。
        * FormState.validate, save, 
* TextField
    * https://api.flutter.dev/flutter/material/TextField-class.html
    * マテリアル デザインのテキスト フィールド
    * onChanged, onSubmitted
    * decoration
    * テキストの制御にはTextEditingControllerを指定する
* TextFormField(FormField派生クラス)
    * Formと連携させる場合はこのクラスを利用する。
    * 内部でTextFieldを生成している


# radio, checkbox, switch
* Radio
* Switch
* Checkbox
## ListTileと組み合わせたウィジェット
* CheckboxListTile
* SwitchListTile
* RadioListTile


# カラー
* ColoredBox
* Colors.blue
* Color.fromRGBO(0, 255, 0, 1.0)
* Color(0xFF00FF00)

# アイコン
* Icon
* CircleAvatar
* FlutterLogo
* アイコンは下記から探すことができる。
* https://fonts.google.com/icons
* 例 `Icon(Icons.notifications)`

# ボタン関連
## ButtonStyleButton
* ButtonStyleButton
    * https://api.flutter.dev/flutter/material/ButtonStyleButton-class.html
    * ButtonStyleオブジェクトによってスタイルが定義されるボタンの基本StatefulWidget
    * TextButton, ElevatedButton, OutlinedButtonなどが継承
        * 以下は廃止となっている為注意
        * https://docs.flutter.dev/release/breaking-changes/buttons
        * FlatButton, RaisedButton, OutlineButton(ButtonTheme)
    * ButtonStyleButton は InkWellを生成している
    * InkWellの内部ではInkResponse, InkResponseの内部ではGestureDetectorを生成している。
* TextButton
    * テキストボタン
* ElevatedButton
    * https://api.flutter.dev/flutter/material/ElevatedButton-class.html
    * フラットなレイアウトに立体感を加えるために使用。既にelevatedなコンテンツ上では使用しないほうが良い。
    > Avoid using elevated buttons on already-elevated content such as dialogs or cards.
* OutlinedButton
    * アウトラインボタン
* スタイル
    * styleで指定する
    ```
    ElevatedButton(
        // サイズ指定: minimumSize, fixedSize, maximumSize,
        style: ElevatedButton.styleFrom(
            fixedSize: Size(getMediaQueryWidth(), double.infinity),
            //...
        ),
        // 円形
        // style: ElevatedButton.styleFrom(shape: const CircleBorder()),
    )
    ```
## その他
* IconButton
    * 内部ではInkResponseを利用
    * 一般的にAppBar.actionsなどで利用されるが、他の多くの箇所にも利用できる。


# 画像関連
* Image
* Image.asset
    * アプリ内(アセット)の画像を表示
* Image.network
    * ネットから画像を表示
    * 内部ではNetworkImage(ImageProviderの派生クラス)を利用している。
* Image.file
    * ローカルファイル形式で画像を表示
    * pubspec.yamlに設定が必要
* Image.memory
    * Uint8List形式で画像を表示
* ImageProvider
    * https://api.flutter.dev/flutter/painting/ImageProvider-class.html
    * Flutterの画像関連のウィジェットでは内部でImageProviderを利用して画像を読み込みしている。
    * 派生する具象クラスとして、FileImage, ResizeImage, AssetImageやMemoryImage等がある。
    ```
    const assetImage = AssetImage('assets/Dash.png');
    final 1pixeldot = MemoryImage(base64Decode("R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw=="));

    testWidgets(i.toString(), (widgetTester) async {
        await widgetTester.pumpWidget(MaterialApp(
            home: Scaffold(
            body: Center(
            child: CircleAvatar(
                radius: 100,
                //backgroundImage: 1pixeldot,
                backgroundImage: assetImage,
            ),
            ),
        )));
    });
    ```

# 非同期関連
* FutureBuilder
* StreamBuilder

# リスナー、ジェスチャー関連
* ListenableBuilder
* NotificationListener
* GestureDetector

# スクロール関連
* https://api.flutter.dev/flutter/widgets/ListView-class.html
* ListView, ListView.Builder
* SingleChildScrollView
* GridView
* ListTile
    * ListViewやCardでよく使われる。
    > Organizes up to 3 lines of text, and optional leading and trailing icons, into a row.
    > Less configurable than Row, but easier to use
* SwitchListTile
* RefreshIndicator
## PageView
* https://api.flutter.dev/flutter/widgets/PageView-class.html
* 各ページを移動すると移動の度に子は廃棄・再生成される
* PageViewのchildを廃棄せずに維持する方法
    * PageStorageKey
        * 筆者はPageStorageKeyを試したが、動作は変わらなかった。
    * AutomaticKeepAliveClientMixinを利用する
        * 下記のサンプルコードで、AutomaticKeepAliveClientMixinを利用している箇所があったため、こちらを参考にした。
            * https://api.flutter.dev/flutter/widgets/PageView/PageView.custom.html
    * サンプルコード
        ```
        PageView(
                children: [
                    KeepAliveItem(child: ChildWidget()),
                    
                    const RoomRequestScreen(),
                ],
            ),
        ```
        ```
        class KeepAliveItem extends StatefulWidget {
            const KeepAliveItem({super.key, required this.child});

            final Widget child;

            @override
            State<KeepAliveItem> createState() => _KeepAliveItemState();
        }

        class _KeepAliveItemState extends State<KeepAliveItem>
                with AutomaticKeepAliveClientMixin {
            @override
            bool get wantKeepAlive => true;

            @override
            Widget build(BuildContext context) {
                super.build(context);
                return widget.child;
            }
        }
        ```



# ライセンス
* LicensePage


# その他
* flutter/lib/src/material/constants.dart
    * AppBarのデフォルトの高さ等のconst値が定義されている
    ```
    //...

    /// The height of the toolbar component of the [AppBar].
    const double kToolbarHeight = 56.0;

    /// The height of the bottom navigation bar.
    const double kBottomNavigationBarHeight = 56.0;

    /// The height of a tab bar containing text.
    const double kTextTabBarHeight = kMinInteractiveDimension;

    //...
    ```