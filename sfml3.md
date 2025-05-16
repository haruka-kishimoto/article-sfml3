
# SFML 3の新機能/SFML 2系からの変更点

この記事ではSFMLの新しいメジャーバージョンであるSFML 3について、その新機能とバージョン2系からの変更点を取り上げる。SFML 3はGitHubにて開発中であり、この記事の内容はGitHubプロジェクトのIssue/PRに基づいている。

## SFML 3

SFMLの新しいメジャーバージョン。[GitHub](https://github.com/SFML/SFML)にて開発中。

C++17を採用している（[\#1855](https://github.com/SFML/SFML/pull/1855)）。

[バージョン3.0.0](https://github.com/SFML/SFML/releases/tag/3.0.0)がリリース済み。

### 新しいイベント処理機構

`sf::Event` APIが刷新された（[\#2992](https://github.com/SFML/SFML/pull/2992)/[\#2766](https://github.com/SFML/SFML/pull/2766)）。`std::variant`と`std::optional`を使って設計され、アクティブなイベントに型安全な方法でアクセスできるようになった。

また`sf::Event::operator bool()`が追加され、ポーリングが簡素化された（[\#2968](https://github.com/SFML/SFML/pull/2968)）。

具体的な使用方法としては、`sf::Event::Closed`のようなプロパティを持たないイベントについて単にそのイベント種類であることを特定する場合は、イベント種類をテンプレート引数に`sf::Event::if()`を呼び出す。

ウィンドウのリサイズやマウスホイールの回転等のプロパティを持つイベントについては、イベント種類をテンプレート引数に`sf::Event::getIf()`を呼び出すと返るポインタ経由でプロパティにアクセスできる。

なおこの設計は依然SFML 3における最終的なものではないがマスターにマージすることで使用者からのフィードバックを集めるものとしている。

さらに[\#3139](https://github.com/SFML/SFML/pull/3139)にて`sf::Event::visit()`と`sf::WindowBase::handleEvents()`が追加された。

```cpp
// 旧来のイベント処理:
for (sf::Event event{}; window.pollEvent(event);) {
    if (event.type == sf::Event::Closed) {
        window.close();
    } else if (event.type == sf::Event::Resized) {
        ...
    }
}

// 改善されたイベント処理:
while (const auto event = window.pollEvent()) {
    if (event->is<sf::Event::Closed>()){
        window.close();
    } else if (const auto* resized = event->getIf<sf::Event::Resized>()) {
        // resized->sizeでリサイズ後のサイズにアクセスできる;
    }
}
```


### 低コストでコピー可能な引数/戻り値をconst参照渡しから値渡しに変更

[\#3047](https://github.com/SFML/SFML/issues/3047)

関数の引数および戻り値の`sf::Vector2<T>`/`sf::Color`/`sf::IpAddress`をconst参照渡しから値渡しに変更する。

個別のPR:

[\#3140](https://github.com/SFML/SFML/pull/3140)
`sf::Vector2<T>`

[\#3170](https://github.com/SFML/SFML/pull/3170)
`sf::Color`

[\#3179](https://github.com/SFML/SFML/pull/3179)
`sf::IpAddress`

PRでは変更の根拠の一つとしてC++コアガイドラインの項目[F.16](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rf-in)を取り上げている。

関数への入力専用引数について、ガイドラインでは（マシンのアーキテクチャ依存ではあるものの）2ワードまたは3ワードを低コストでコピー可能としており、そのようなオブジェクトは値渡しすべきだとしている。

PRでは念のため2ワードの型を値渡しで扱う。

以下は低コストでコピー可能な型かを判定する変数テンプレート:
```cpp
template<typename T>
constexpr bool is_cheaply_copied_type = std::is_trivially_copyable_v<T> and (sizeof(T) <= (2 * sizeof(void*)));
```

### `sf::Vector`の機能追加
`sf::Vector`に数学的機能が追加される。`sf::Vector3<T>`にはドット積やクロス積、長さ、正規化、コンポーネント単位での乗除算が、`sf::Vector2<T>`にはそれらに加えて**角度**を扱う機能が複数サポートされる。

### `sf::Angle`の追加
角度を扱う新しいクラスとして`sf::Angle`が追加される。デフォルト構築および度/ラジアンからの構築とそれぞれの値への変換、算術演算、ユーザ定義リテラル（`_deg`/`_rad`）をサポートする。

```cpp
using namespace sf::Literals;

// デフォルト構築。
sf::Angle angle{};

// ユーザ定義リテラル/ヘルパ関数による構築。
angle = 20._deg;
angle = sf::degrees(20.f);

// ラジアンへの変換。
float radians = angle.asRadians();

// ラジアンから構築。
angle = sf::radians(radians);
```

他に以下のメンバ関数を持つ。

- [-180°, 180°)に制限した値を返す`wrapSigned()`
    - 例えば`sf::degrees(360.f).wrapSigned()`の戻り値は`sf::degrees(0)`と等しい。
- [0°, 360°)に制限した値を返す`wrapUnsigned()`
    - 例えば`sf::degrees(-90).wrapUnsigned()`の戻り値は`sf::degrees(270)`と等しい。

#### `sf::Vector2`との併用

`sf::Vector2<T>`での`sf::Angle`を使った機能として、`sf::Vector2<T>`を極座標（長さと角度で表す座標）から構築するコンストラクタが追加された。

```cpp
sf::Vector2f v{ length, angle };
```

- `T`は浮動小数点数である必要がある。
- このコンストラクタで構築した`sf::Vector2<T>`の`length()`および`angle()`の戻り値はコンストラクタに与えた値とは必ずしも一致しない。


### `sf::Rect`再設計

#### 位置とサイズ

`sf::Rect<T>`のメンバ変数について、`left`/`top`/`width`/`height`の4つの`T`が削除され、代わりに`sf::Vector2<T>`で`position`と`size`を持つように変更された。またメンバ関数`getPosition()`/`getSize()`が削除された。

#### 交差判定
`intersects()`は`findIntersection()`となる。以前は交差部分を扱うかどうかでオーバーロードを提供していたが、1つのメンバ関数に集約される。戻り値が`bool`から`std::optional<sf::Rect<T>>`に変更され、交差部分があればそれを有効値として返すようになった。
```cpp
if (auto intersection = rect.findIntersection(other_rect); intersection) {
    const auto& interssection_rect = *intersection;
    ...
}
```


### 正規化テクスチャ座標

`sf::RenderStates`でテクスチャ座標を正規化された値として扱うように指定できるようになる。

`sf::CoordinateType`という`enum class`が追加された。値は正規化テクスチャ座標を表す`sf::CoordinateType::Normalized`とピクセル座標を表す`sf::CoordinateType::Pixels`の2つで、`sf::RenderStates`でのデフォルト値はピクセル座標となっている。

初心者にとってはピクセル座標のほうがテクスチャ画像のどの部分が表示されるのかの対応を理解しやすいものの、一般的には正規化されたテクスチャ座標を扱うことが多い。この変更によりそのようなデータをピクセル座標へ変換する手間なく直接扱えるようになる。

これは`sf::VertexArray`と`sf::VertexBuffer`の描画に限られ、その他のDrawableは引き続きピクセル座標をデフォルトとして使う（ピクセル座標をデフォルトとする議論は[\#1773](https://github.com/SFML/SFML/issues/1773)を参照）。

`target.draw(shape, sf::CoordinateType::Normalized)`のように、Drawableを正規化された値で描画するようオーバーロードを呼び出した場合の挙動については、SFMLユーザに警告なしにテクスチャ座標の扱いを（正規化された値ではなく）ピクセル座標としている。この点についてアサーションか`sf::err()`で対応する方法もあるが、既にそれらによる警告なしに`sf::Shape`/`sf::Sprite`/`sf::Text`においてテクスチャが上書きされる処理が存在するため、暗黙的に内部で描画内容が上書きされることが容認されているとしてそれに倣っている。


### ステンシルテスト

`sf::StencilMode`クラスによるステンシルテストのサポートが追加された。`glStencilFunc()`による比較に直接対応するように設計されている。

`sf::CoordinateType`と同じく`sf::RenderStates`で扱う。


## Unicodeサポート

### [\#3406](https://github.com/SFML/SFML/issues/3406)

Unicodeサポートに関するIssue/PRの一覧。

## Milestone 3.1での変更

### [\#3380](https://github.com/SFML/SFML/pull/3380)
Opusサポートを追加。

### [\#3367](https://github.com/SFML/SFML/pull/3367)
非const版の`sf::Event::getIf`を追加。

### [\#3366](https://github.com/SFML/SFML/pull/3366)
`sf::VertexArray`に`begin()`と`end()`を追加。`sf::VertexArray`を範囲for文で扱えるようになる。

### [\#3238](https://github.com/SFML/SFML/pull/3238)
イベント処理にCapsLock/NumLock/ScrollLockのサポートを追加。

### [\#2265](https://github.com/SFML/SFML/pull/2265)
ドラッグアンドドロップをサポート。ドラフト。


## SFML 3の新機能/SFML 2系からの変更点のリスト

### [\#3231](https://github.com/SFML/SFML/pull/3231)
ウィンドウスタイルを指定しない`sf::WindowBase::create()`を追加。

### [\#3214](https://github.com/SFML/SFML/pull/3214)
「アンチエイリアシング」の綴りを変更。

綴りが`Antialiasing`から`AntiAliasing`に変更された（「エイリアシング」の頭文字のAが大文字に変更された）。

具体的には`sf::RenderTexture`のメンバ関数`getMaximumAntialiasingLevel()`が`getMaximumAntiAliasingLevel()`に、`sf::ContextSettings`のメンバ変数`antialiasingLevel`が`antiAliasingLevel`に変更された。

### [\#3212](https://github.com/SFML/SFML/pull/3212)
`sf::Vector2<T>`/`sf::Vector3<T>`のメンバ関数の名前を変更。

[\#1979](https://github.com/SFML/SFML/pull/1979)と[\#2086](https://github.com/SFML/SFML/pull/2086) で追加されたメンバ関数の名前が変更された。

`lengthSq()`が`lengthSquared()`に、`cwiseMul()`と`cwiseDiv()`がそれぞれ`componentWiseMul()`と`componentWiseDiv()`に変更された。

### [\#3144](https://github.com/SFML/SFML/pull/3144)
`nullptr`による`sf::String`の構築を禁止。

### [\#2973](https://github.com/SFML/SFML/pull/2973)
ユーザ定義サウンドエフェクトのサポートを追加。

サウンドエフェクト(DSP)のサポートはもともと[\#1708](https://github.com/SFML/SFML/pull/1708)にて検討されていた。
しかしマージまで至らず、前提とするオーディオバックエンドがOpenALからminiaudioに置き換えられこともありPRがcloseされた。
その趣旨を受け継いだものが今回のPRであり、APIが拡張可能なものに再設計された上でのサポート追加となる。

### [\#2968](https://github.com/SFML/SFML/pull/2968)
`sf::Event`に`operator bool()`を追加。

### [\#2962](https://github.com/SFML/SFML/pull/2962)
オーディオモジュールで`enum class`を使用。`sf::SoundSource::Status`を`enum`から`enum class`に変更。

### [\#2891](https://github.com/SFML/SFML/pull/2891)
グラフィックスモジュールで`enum class`を使用。`sf::BlendMode::Factor`/`sf::BlendMode::Equation`、`sf::Shader::Type`、`sf::VertexBuffer::Usage`を`enum`から`enum class`に変更。

### [\#2850](https://github.com/SFML/SFML/pull/2850)
`sf::Keyboard::key`を`enum`から`enum class`に変更。

### [\#2838](https://github.com/SFML/SFML/pull/2838)
`sf::Mouse::XButton1`/`sf::Button::XButton2`を`sf::Mouse::Button::Extra1`/`sf::Mouse::Button::Extra2`に変更。

[\#2822](https://github.com/SFML/SFML/pull/2822)によって"Button"が繰り返されることになり冗長であることと、より意味のある名前にするために変更される。

### [\#2822](https://github.com/SFML/SFML/pull/2822)
`sf::Mouse::Button`/`sf::Mouse::Wheel`/`sf::Joystick::Axis`/`sf::Seonsor::Type`/`sf::Cursor::Type`を`enum`から`enum class`に変更。

以下については同様の変更の対象外:
- `sf::Keyboard`については[\#2850](https://github.com/SFML/SFML/pull/2850)に分割されている。
- `sf::Style`と`sf::ContextSettings::Attribute`についてはフラグを表すものとして含まない。
- `sf::Event::EventType`については[\#2766](https://github.com/SFML/SFML/pull/2766)でイベントAPIの改善が予定されているため含まない。

### [\#2818](https://github.com/SFML/SFML/pull/2818)
`sf::State`を追加。フルスクリーン/ウィンドウの指定を`sf::Style`から分離。

### [\#2766](https://github.com/SFML/SFML/pull/2766)
`sf::Event`のAPIを改善。[\#2668](https://github.com/SFML/SFML/issues/2668)での議論を経て`std::variant`を使用して実装。

### [\#2756](https://github.com/SFML/SFML/pull/2756)
これまで`sf::Image::saveToMemory()`はバッファを引数に取り、そのバッファに内容を格納できたかを`bool`で返していた。この変更ではバッファを受け取る引数を削除し、結果を`std::optional`で返すようにする。

### [\#2749](https://github.com/SFML/SFML/pull/2749)
オーディオバックエンドをOpenALからminiaudioに置き換え。

### [\#2417](https://github.com/SFML/SFML/pull/2417)
ウィンドウアイコンを`sf::Image`をもとに設定できるようにする。

### [\#2329](https://github.com/SFML/SFML/pull/2329)
`sf::InputSoundFile`/`sf::OutputSoundFile`がムーブに対応。

### [\#2286](https://github.com/SFML/SFML/pull/2286)
`sf::PrimitiveType`を`enum`から`enum class`に変更。

### [\#2137](https://github.com/SFML/SFML/pull/2137)
`sf::Image::copy()`の戻り値を`void`から`bool`に変更。成否を表し、`[[nodiscard]]`属性付き。

### [\#2131](https://github.com/SFML/SFML/pull/2131)
ネットワーク周りの`enum`を`enum class`に変更。

### [\#2086](https://github.com/SFML/SFML/pull/2086) 
`sf::Vector3<T>`にメンバ関数を追加。

`length()`, `lengthSq()`, `normalized()`, `dot()`, `corss()`, `cwiseMul()`, `cwiseDiv()`。詳細は後述の#1979を参照のこと。

### [\#2085](https://github.com/SFML/SFML/pull/2085)
`sf::Vector2<T>`を極座標から構築するコンストラクタを追加。ベクトルの長さとX軸からの角度を引数に取る。

### [\#2055](https://github.com/SFML/SFML/pull/2055)
各機能の数値ペアの引数を、2つの`T`から`sf::Vector2<T>`に変更（オーバーロードは提供しない）。

### [\#2016](https://github.com/SFML/SFML/pull/2016)
`sf::Font`, `sf::Text`, `sf::Image`, `sf::String`がムーブに対応。

### [\#2014](https://github.com/SFML/SFML/pull/2014)
`sf::Packet`がムーブに対応。

### [\#1979](https://github.com/SFML/SFML/pull/1979)
`sf::Vector2f<T>`のメンバ関数を拡充。

- 長さを返す`length()`
- 自身のドット積を返す`lengthSq()`
- 自身の正規化（長さで除算）した値を返す`normalized()`
    - 事前条件として自身がゼロベクトルではないこと。
- 対象への角度を返す`angleTo()`
- 自身の角度を返す`angle()`
- 回転させた値を返す`rotatedBy()`
- 指定した軸上への射影を返す`projectedOnto()`
    - 軸を正規化する必要はない。
    - 軸の長さはゼロであってはならない。
- 時計回りに90度回転させたベクトルを返す`perpendicular()`
- ドット積/クロス積を返す`dot()`/`cross()`
- コンポーネント単位の乗除算を行う`cwiseMul()`/`cwiseDiv()`。
    - "cwise"は"component-wise"の略。

### [\#1969](https://github.com/SFML/SFML/pull/1969)
`sf::Angle`を追加。

### [\#1952](https://github.com/SFML/SFML/pull/1952)
`sf::Rect<T>::intersects()`を`sf::Rect<T>::findIntersection()`に変更。

### [\#1948](https://github.com/SFML/SFML/pull/1948)
`sf::Rect<T>`の4つの`T`を引数に取るコンストラクタを削除。

### [\#1942](https://github.com/SFML/SFML/pull/1942)
`sf::Rect<T>::contains()`の引数を2つの`T`から`sf::Vector2<T>`に変更（オーバーロードは提供しない）。

### [\#1934](https://github.com/SFML/SFML/pull/1934)
`sf::Time`がconstexpr classになった。

### [\#1932](https://github.com/SFML/SFML/pull/1932)
`sf::FileInputStream`がムーブに対応。

### [\#1910](https://github.com/SFML/SFML/pull/1910)
`sf::Vertex`がconstexpr classになった。

### [\#1909](https://github.com/SFML/SFML/pull/1909)
`sf::Rect<T>`がconstexpr classになった。

### [\#1904](https://github.com/SFML/SFML/pull/1904)
`sf::Color`がconstexpr classになった。

### [\#1903](https://github.com/SFML/SFML/pull/1903)
`sf::Vector2<T>`と`sf::Vector3<T>`がconstexpr classになった。

### [\#1807](https://github.com/SFML/SFML/pull/1807)
`sf::RenderStates`に`sf::CoordinateType`を追加。

### [\#1453](https://github.com/SFML/SFML/pull/1453)
ステンシルテストのサポートを追加。

## 内部的な変更のリスト（一部）

以下の変更以外にも、C++17を用いたプロジェクトにおける当然のモダナイズ（range-based forやstructured bindings等々）が数多くあり、そのいずれも着実に取り込まれている。

### [\#2386](https://github.com/SFML/SFML/pull/2386)
引数の型を`const char*`から`std::string_view`に変更。

### [\#2332](https://github.com/SFML/SFML/pull/2332)
一時オブジェクトな`sf::Font`での`sf::Text`構築を防止する。

右辺値参照（`sf::Font&&`）を引数に取るオーバーロードを`= delete`指定する。

### [\#2315](https://github.com/SFML/SFML/pull/2315)
クランプ（値を範囲内に収める）処理を`std::clamp()`に置き換える。

値のクランプは`std::min(std::max(v, lo), hi)`というイディオムがあるが、それを`std::clamp()`に置き換える。

### [\#2195](https://github.com/SFML/SFML/pull/2195)
`sf::Vector2<T>`および`sf::Vector3<T>`の`length()`の実装を`std::hypot()`から`std::sqrt()`へ変更。

`std::hypot()`は計算途中のオーバーフロー/アンダーフローへの対処のために`std::sqrt()`よりも遅い。

### [\#2192](https://github.com/SFML/SFML/pull/2192), [\#2196](https://github.com/SFML/SFML/pull/2196), [\#2199](https://github.com/SFML/SFML/pull/2199), [\#2200](https://github.com/SFML/SFML/pull/2200)
SFML独自の古い固定長整数型のエイリアスをC++標準に置き換える（`sf::Int8`等を`std::int8_t`へ等）。

[\#2021](https://github.com/SFML/SFML/pull/2021)もVulkan.hppで同様の置き換えを含んでる。

### [\#2156](https://github.com/SFML/SFML/pull/2156)
uniform initializationと`= default`指定。

### [\#2090](https://github.com/SFML/SFML/pull/2090)
`Font::loadPage()`で`try_emplace()`を使うように変更。

### [\#2044](https://github.com/SFML/SFML/pull/2044)
メッセージ出力で`std::quoted()`を使うように変更。

以前は引用符をエスケープしたり閉じるのに一文字くっつけたりと涙ぐましい。

### [\#2043](https://github.com/SFML/SFML/pull/2043)
ファイルが見つからなかった場合にその絶対パスを出力する（以前はファイル名のみ）。

### [\#2019](https://github.com/SFML/SFML/pull/2019)
インクリメントを必要がなければ後置から前置に変更。

### [\#2015](https://github.com/SFML/SFML/pull/2015)
`std::endl`の使い過ぎを修正。

### [\#1964](https://github.com/SFML/SFML/pull/1964)
あちこちの引数を`const std::string&`から`std::filesystem::path`に変更。

### [\#1901](https://github.com/SFML/SFML/pull/1901)
`sf::NonCopyable`を削除して`= delete`で置き換える。

### [\#1880](https://github.com/SFML/SFML/pull/1880)
clang-tidyのmodernize-use-autoで洗い出した情報欠落が起きない部分で`auto`を使うように変更。
