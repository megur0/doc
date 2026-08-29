---
title: "Google Maps API - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > Google Maps API

* 以下はネイティブアプリ(Android / iOS)からGoogle Mapsを利用する場合を想定した、APIの有効化とAPIキー作成の手順。
* 下記の例では`alpha`コンポーネントを使っているが、gcloudのバージョンによっては`alpha`なしで実行できる(APIキー関連のコマンドはGAになっている)。手元の環境で`gcloud services api-keys --help`を確認するとよい。

## APIの有効化
```sh
gcloud --project [PROJECT_ID] services enable maps-android-backend.googleapis.com maps-ios-backend.googleapis.com
```


## APIキーの作成
前提: Google MapsのAPIが有効になっていること。

```sh
gcloud --project [PROJECT_ID] alpha services api-keys create --display-name="google map key for flutter"
```


## APIキーが呼び出せるAPIの制限
```sh
gcloud --project [PROJECT_ID] services api-keys list

gcloud --project [PROJECT_ID] alpha services api-keys update [KEY_ID] \
  --api-target=service=maps-android-backend.googleapis.com \
  --api-target=service=maps-ios-backend.googleapis.com
```


## アプリケーションによる制限(未検証)
必要に応じて、以下のようにバンドルID(iOS)やパッケージ名+SHA-1フィンガープリント(Android)による制限をかける。

```sh
gcloud --project [PROJECT_ID] services api-keys list

# iOS
gcloud --project [PROJECT_ID] alpha services api-keys update [KEY_ID] \
  --allowed-bundle-ids=[ALLOWED_BUNDLE_ID_1],[ALLOWED_BUNDLE_ID_2]

# Android
gcloud --project [PROJECT_ID] alpha services api-keys update [KEY_ID] \
  --allowed-application=sha1_fingerprint=[SHA1_FINGERPRINT_1],package_name=[PACKAGE_NAME_1] \
  --allowed-application=sha1_fingerprint=[SHA1_FINGERPRINT_2],package_name=[PACKAGE_NAME_2]
```

* APIキーの制限全般については https://cloud.google.com/docs/authentication/api-keys#api_key_restrictions
