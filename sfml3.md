
# SFML 3の新機能/API変更点

最終更新日時: 2023年02月28日 23時17分

この記事ではSFMLの次期メジャーバージョンであるSFML 3について、その新機能とバージョン2.5系/2.6系からのAPI変更点を取り上げる。SFML 3はGitHubにて開発中であり、この記事の内容はGitHubプロジェクトのIssue/PRのうち、ラベルがlabel:featureかつlabel:s:acceptedであるものに基づいている。

## SFML 3

SFMLの次期メジャーバージョン。[GitHub](https://github.com/SFML/SFML)にて開発中。

C++17を採用している（[\#1855](https://github.com/SFML/SFML/pull/1855)）。

### sf::Vectorの機能追加
`sf::Vector`に数学的機能が追加される。`sf::Vector3<T>`にはドット積やクロス積、長さ、正規化、コンポーネント単位での乗除算が、`sf::Vector2<T>`にはそれらに加えて**角度**を扱う機能が複数サポートされる。

### sf::Angleの追加およびsf::Vector2fとの併用
角度を扱う新しいクラスとして`sf::Angle`が追加される。デフォルト構築および度/ラジアンからの構築とそれぞれの値への変換、算術演算、ユーザ定義リテラル（_deg/_rad）をサポートする。

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
    - 例えばsf::degrees(360.f).wrapSigned()の戻り値はsf::degrees(0)と等しい。
- [0°, 360°)に制限した値を返す`wrapUnsigned()`
    - 例えばsf::degrees(-90).wrapUnsigned()の戻り値はsf::degrees(270)と等しい。

`sf::Vector2f`との併用。

例えばゲーム開発において極座標を用いるとキャラクタ等の任意の方向への動き等を簡単に表現できる。

```cpp
using SpeedPerSecond = float;
SpeedPerSecond speed = 100.f;

// 極座標での構築。長さ（この例では速度）とX軸からの角度を引数に取る。
sf::Vector2f offset{ speed, sf::degrees(-20.f) };

// キャラクタ等の現在位置を更新する。
// deltaはゲームループの経過時間（単位はSpeedPerSecondと合わせて秒とする）。
auto potision += offset * delta;
```

### sf::Rectの交差判定刷新
`intersects()`は`findIntersection()`となる。以前は交差部分を扱うかどうかでオーバーロードを提供していたが、1つのメンバ関数に集約される。戻り値が`bool`から`std::optional<sf::Rect<T>>`に変更され、交差部分があればそれを有効値として返すようになった。
```cpp
// 戻り値を使う例（当然discardableなので使わなくてもよい）。
if (auto intersection = rect.findIntersection(other_rect); intersection) {
    const auto& interssection_rect = intersection.value();
    ...
 }
```


## 新機能/API変更点のリスト

### [\#2417](https://github.com/SFML/SFML/pull/2417)
ウィンドウアイコンを`sf::Image`をもとに設定できるようにする。

### [\#2137](https://github.com/SFML/SFML/pull/2137)
`sf::Image::copy()`の戻り値を`void`から`bool`に変更。成否を表し、`[[nodiscard]]`属性付き。

### [\#2131](https://github.com/SFML/SFML/pull/2131)
ネットワーク周りの`enum`を`enum class`に変更。

具体的には`sf::Ftp::TransferMode`, `sf::Http::Responce::Status`, `sf::Http::Request::Method`, `sf::Sockte::Status`, `sf::Socket::Type`。

### [\#2086](https://github.com/SFML/SFML/pull/2086) 
`sf::Vector3<T>`にメンバ関数を追加。

`length()`, `lengthSq()`, `normalized()`, `dot()`, `corss()`, `cwiseMul()`, `cwiseDiv()`。詳細は後述の#1979を参照のこと。

### [\#2085](https://github.com/SFML/SFML/pull/2085)
`sf::Vector2<T>`を極座標から構築するコンストラクタを追加。ベクトルの長さとX軸からの角度を引数に取る。

### [\#2055](https://github.com/SFML/SFML/pull/2055)
各機能の数値ペアの引数を、2つの`T`から`sf::Vector2<T>`に変更（オーバーロードは提供しない）。

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

### [\#1910](https://github.com/SFML/SFML/pull/1910)
`sf::Vertex`がconstexpr classになった。

### [\#1909](https://github.com/SFML/SFML/pull/1909)
`sf::Rect<T>`がconstexpr classになった。

### [\#1904](https://github.com/SFML/SFML/pull/1904)
`sf::Color`がconstexpr classになった。

### [\#1903](https://github.com/SFML/SFML/pull/1903)
`sf::Vector2<T>`と`sf::Vector3<T>`がconstexpr classになった。

### [\#1932](https://github.com/SFML/SFML/pull/1932)
`sf::FileInputStream`がムーブに対応。

### [\#2014](https://github.com/SFML/SFML/pull/2014)
`sf::Packet`がムーブに対応。

### [\#2016](https://github.com/SFML/SFML/pull/2016)
`sf::Font`, `sf::Text`, `sf::Image`, `sf::String`がムーブに対応。

### [\#2329](https://github.com/SFML/SFML/pull/2329)
`sf::InputSoundFile/sf::OutputSoundFile`がムーブに対応。


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

### [\#2192](https://github.com/SFML/SFML/pull/2192), [\#2196](https://github.com/SFML/SFML/pull/2196), [\#2199](https://github.com/SFML/SFML/pull/2199), [\#2200](https://github.com/SFML/SFML/pull/2200)
SFML独自の古い固定長整数型のエイリアスをC++標準に置き換える（`sf::Int8`等を`std::int8_t`へ等）。

[\#2021](https://github.com/SFML/SFML/pull/2021)もVulkan.hppで同様の置き換えを含んでる。

### [\#2195](https://github.com/SFML/SFML/pull/2195)
`sf::Vector2<T>`および`sf::Vector3<T>`の`length()`の実装を`std::hypot()`から`std::sqrt()`へ変更。

`std::hypot()`は計算途中のオーバーフロー/アンダーフローへの対処のために`std::sqrt()`よりも遅い。

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
