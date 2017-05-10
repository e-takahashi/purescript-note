# purescript-note

* foreign moduleは、参照するpurescriptファイルと同名でないと駄目なようだ。    
例えば、Ex2.pursに対してEx2.jsがないと以下のエラーが出る。
````
The foreign module implementation for module Ex2 is missing.
````

## readJSON

[PureScript by Example](https://leanpub.com/purescript/read)の10.16 Working With Untyped Dataで登場している __readJSON__ がライブラリに見当たらないので作ってみた。
ちなみに[Exercisesのソース]にもなかった。

[Wrapping JavaScript for PureScript](http://blog.ndk.io/purescript-ffi.html)の中ほどにあるコードを参考にした。

先ずは__readJSON__の仕様は次の通り。

````
readJSON :: forall a. String -> F a
````

まずはJavaScriptの実装コードを作成する。
ファイル名はMyJSON.js。

````
"use strict";

exports.readJSONImpl = function (json, onSuccess, onFailure) {
  try {
	  var obj = JSON.parse(json);
	  return onSuccess(obj);
  } catch (e) {
	  return onFailure(e.toString());
  }
};
````

成功時と失敗時それぞれのコールバック onSuccess、onFailure を引数に取る。
成功時、失敗時ともに コールバックの実行結果を return していることに注意。
Effを使わないため、function()にする必要はない。
また、コールバックを呼ぶだけでは駄目で、結果を return で返さなければならない。

次にPureScriptコードを実行する。

必然的にPureScriptのモジュール名はMyJSONでファイル名はMyJSON.pursとなる。

````
module MyJSON where

import Prelude

import Data.Foreign 
import Data.Function.Uncurried (Fn3, runFn3)
import Data.Either
import Control.Monad.Except
import Control.Plus
import Data.NonEmpty
import Data.List.Types 

type Error' = String

foreign import readJSONImpl :: forall a.
                               Fn3
                               String
                               (a -> Either Error' a)
                               (Error' -> Either Error' a)
                               (Either Error' a)

readJSON :: forall a. String -> F a
readJSON s =
  case f s of
    Right a -> pure a
    Left  e -> throwError (NonEmptyList ((JSONError e) :| empty))
  where
    f s = runFn3 readJSONImpl s Right Left
````

先ずはJavaScript側でエラー時にコールバックへ渡すデータの型Error'を決める。
````
type Error' = String
````
ここでは簡単にStringとしておいた。

次にJavaScriptの関数をインポートする。
JavaScriptの実装で複数の引数を受取るようにしたので、Data.Function.UncurriedのFn3を使う。
実行時はrunFn3を使う。

コールバックの扱いはEitherを使う。
成功時と失敗時それぞれのコールバック関数と全体の戻り値の型を `Either Error' a` とする。
ここになかなかたどり着けなかったが次のように考えた。

`readJSONImpl`の結果をEitherにしておいて、その結果に応じてFつまりExceptの振舞いをすればいい、と考える。
それでまずは`readJSONImpl`の結果の型が`Either Error' a`となる。
ちなみに`a`が成功時の値だ。

これより成功時と失敗時それぞれのコールバック関数の結果も同じく`Either Error' a`となる。
後は、JavaScript実装に従って、成功時はパース結果を受ける関数なので、`a -> Either Error' a`、
失敗時はエラー文字列を受ける関数なので、`Error' -> Either Error' a`となる。
なお、`Error`の実装はこの時点で決定している。

この後は、このreadJSONImplを呼び出して、結果に応じFの振舞いをする関数を書けばいい。

readJSONImplを呼び出すコードは以下の通り。

````
f s = runFn3 readJSONImpl s Right Left
````

これもなかなかたどり着けなった。
コールバック関数をどう渡すかで悩んだが、単純にRightとLeftを渡せばよかった。
どちらもEitherのコンストラクタで、そのまま当てはまる。

FはData.Foreignで定義されている型だが、実体は
````
type F = Except MultipleErrors
type MultipleErrors = NonEmptyList ForeignError
data ForeignError
  = ForeignError String
  | TypeMismatch String String
  | ErrorAtIndex Int ForeignError
  | ErrorAtProperty String ForeignError
  | JSONError String
````
である。
[Data.Foreign](https://pursuit.purescript.org/packages/purescript-foreign/4.0.0/docs/Data.Foreign#t:Foreign)にすべてある。

場合わけはcase of式でやって、成功時（Right a)は、単純にaをpureで返せばいい。
失敗時が少し手間取った。

計算自体はthrowErrorでいいが、問題はthrowErrorへ渡すデータである。
Fの定義より、MultipleErrorsつまりForeignErrorを要素とするNonEmptyListを作らなければならない。
ForeignErrorは単純なADTなのでいいとして、NonEmptyListをどうするか。

[NonEmptyList](https://pursuit.purescript.org/packages/purescript-lists/4.0.1/docs/Data.List.Types#t:NonEmptyList)に定義がある。
````
NonEmptyList (NonEmpty List a)
````
がコンストラクタだ。
[Data.NonEmpty](https://pursuit.purescript.org/packages/purescript-nonempty/4.0.0/docs/Data.NonEmpty#t:NonEmpty)より、
NonEmptyの作り方を見ると、例に目的にかなうコードがあったので使った。
````
Left  e -> throwError (NonEmptyList ((JSONError e) :| empty))
````
なお、次のコードでも動作する。
````
Left  e -> throwError (NonEmptyList (singleton (JSONError e)))
````

以上で実装は完了。
PSCiで動作を確認できる。
（プロジェクトの作成やパッケージのインストールは省略）

````
import Control.Monad.Except
import Data.Foreign
import MyJSON

runExcept (readJSON "\"hello\"" :: F String)
(Right "hello")

runExcept (readJSON "hello" :: F String)
(Left (NonEmptyList (NonEmpty (JSONError "SyntaxError: Unexpected token h") Nil)))

runExcept (readJSON "123" :: F Int)
(Right 123)
````


