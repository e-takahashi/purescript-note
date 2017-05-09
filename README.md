# purescript-note

* foreign moduleは、参照するpurescriptファイルと同名でないと駄目なようだ。    
例えば、Ex2.pursに対してEx2.jsがないと以下のエラーが出る。
````
The foreign module implementation for module Ex2 is missing.
````
* ContTの理解    
````
ContT r m a
````
