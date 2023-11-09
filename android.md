# adb

*  (Android Debug Bridge](https://developer.android.com/studio/command-line/adb?hl=ja)

## パッケージをダウンロード

> adb shell pm list packages -f

で一覧

> adb shell pm list packages -f | findstr com.DefaultCompany.android

で探したいcom.DefaultCompany.androidの場所がわかる。
```
package:/data/app/~~ylvW_-RnDo7gfU2OSV734g==/com.DefaultCompany.android-QsZdwmCEOEC46M7yH9jcNg==/base.apk=com.DefaultCompany.android
```
と出力されるので、package:から=com.DefaultCompany.androidまでをファイルのパスとしてダンプする。

> adb pull "/data/app/~~ylvW_-RnDo7gfU2OSV734g==/com.DefaultCompany.android-QsZdwmCEOEC46M7yH9jcNg==/base.apk" x.apk
