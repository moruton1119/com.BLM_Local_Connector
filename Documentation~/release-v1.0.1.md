---
title: "BLM Connector v1.0.1 リリース 〜バグ修正と安全性向上〜"
emoji: "🐛"
type: "tech"
topics: ["Unity", "VRChat", "BOOTH", "CSharp", "リファクタリング"]
published: false
---

# v1.0.1 リリースのお知らせ

Morulab Unity Tools (BLM Connector) v1.0.1 をリリースしました！

今回はユーザーから報告いただいた2つの重要なバグ修正と、安全性向上のための新機能、そしてコードのリファクタリングを行いました。

## 対象読者

- BLM Connectorをお使いの方
- Unity Editor拡張開発に興味がある方
- アセットインポートの自動化に関心がある方

---

# バグ修正

## 1. モーダルのImportボタンでタグが付かない問題

### 問題の概要

商品詳細モーダルから「Import」ボタンを押してUnityPackageをインポートした際、`BLM_PID_xxx` タグが正しく付与されない問題がありました。

### 原因

```csharp
// 修正前 (BLMConnectorWindow.cs)
AssetImportQueue.StartManualImport(product.id);  // IsImporting = true
try {
    BLMAssetImporter.ImportAsset(asset, product.name);  // 非同期メソッド
    ...
}
finally {
    AssetImportQueue.EndManualImport();  // 即座に IsImporting = false
}
```

`AssetDatabase.ImportPackage()` は**非同期メソッド**です。インポートが完了する前に `EndManualImport()` が実行されてしまい、`BLMProductTagger` がタグ付けを行うタイミングで `IsImporting` が既に `false` になっていました。

### 修正内容

モーダルからのインポートもキュー経由で行うように変更しました：

```csharp
// 修正後
if (asset.assetType == AssetType.UnityPackage)
{
    importedProductIds.Add(product.id);
    AssetImportQueue.Enqueue(asset.fullPath, product.id);
    AssetImportQueue.StartImport();  // キュー経由で処理
    // ...
}
```

これにより、ダブルクリックでの一括インポートと同じ処理フローになり、タグ付けが確実に行われるようになりました。

---

## 2. テクスチャが変な場所にインポートされる問題

### 問題の概要

テクスチャをインポートすると、プロジェクト内の意図しない場所にコピーされたり、インポート自体が失敗する現象が報告されました。

### 原因

```csharp
// 修正前 (BLMAssetImporter.cs)
string destinationFolder = Path.Combine(DefaultImportFolder, sanitizedProductName);
// DefaultImportFolder = "Assets/BLM_Imports" (相対パス)

Directory.CreateDirectory(destinationFolder);  // カレントディレクトリ基準
File.Copy(asset.fullPath, destPath, true);    // 同上
```

`Directory.CreateDirectory()` と `File.Copy()` は**カレントディレクトリ**を基準にパスを解釈します。Unityエディタのカレントディレクトリは必ずしもプロジェクトルートではないため、意図しない場所にフォルダが作成されていました。

### 修正内容

パスを絶対パスに統一しました：

```csharp
// 修正後
string projectPath = Directory.GetParent(Application.dataPath)?.FullName;
string absoluteDestFolder = Path.Combine(projectPath, DefaultImportFolder, sanitizedProductName);

Directory.CreateDirectory(absoluteDestFolder);  // 絶対パスで作成
File.Copy(asset.fullPath, absoluteDestPath, true);  // 絶対パスでコピー
```

---

# 新機能

## 削除時の安全警告

### 背景

同じショップの複数商品をインポートすると、フォルダ構造が以下のようになることがあります：

```
Assets/BLM_Imports/
  └── ShopName/           ← BLM_PID_123, BLM_PID_456 両方のタグ付き
      ├── ProductA/
      └── ProductB/
```

この状態で商品Aを削除すると、親フォルダ `ShopName` ごと消えてしまい、商品Bも一緒に消えてしまいます。

### 実装内容

削除前にフォルダのラベルを確認し、複数の `BLM_PID_xxx` タグが含まれている場合は警告を表示します：

```
┌─────────────────────────────────────────────┐
│  Warning: Shared Folder                     │
├─────────────────────────────────────────────┤
│  This folder contains other products:       │
│  • Product ID: 456                          │
│  • Product ID: 789                          │
│                                             │
│  Deleting will remove ALL contents          │
│  including other products.                  │
│                                             │
│  Target: Assets/BLM_Imports/ShopName        │
├─────────────────────────────────────────────┤
│              [Delete Anyway]  [Cancel]      │
└─────────────────────────────────────────────┘
```

---

# リファクタリング

## 定数の集約 (BLMConstants.cs)

ハードコードされていた文字列・数値を1箇所に集約しました：

```csharp
public static class BLMConstants
{
    public const string DefaultImportFolder = "Assets/BLM_Imports";
    public const string ThumbnailCacheDir = "Library/Moruton.BLMConnector/Thumbnails";
    public const string LabelPrefix_PID = "BLM_PID_";
    public const string Label_Managed = "BLM_Managed";
    public const int MaxFolderNameLength = 50;
    // ...
}
```

## 拡張子判定の共通化 (AssetTypeUtils.cs)

`BLMDatabaseService` と `LocalAssetService` に重複していた拡張子判定ロジックを共通化しました：

```csharp
public static class AssetTypeUtils
{
    public static readonly string[] TextureExtensions = { ".png", ".jpg", ".jpeg", ".tga", ".psd" };
    public static readonly string[] ModelExtensions = { ".fbx", ".obj" };
    public static readonly string[] AudioExtensions = { ".wav", ".mp3", ".ogg" };

    public static AssetType GetAssetType(string extension) { ... }
    public static bool IsTexture(string extension) { ... }
}
```

---

# まとめ

v1.0.1では以下の改善を行いました：

| カテゴリ | 内容 |
|---------|------|
| バグ修正 | モーダルImport時のタグ付け問題を解決 |
| バグ修正 | テクスチャのインポート先を修正 |
| 新機能 | 削除時の安全警告を追加 |
| リファクタリング | 定数を `BLMConstants.cs` に集約 |
| リファクタリング | 拡張子判定を `AssetTypeUtils.cs` に共通化 |

---

# アップデート方法

VCC（VRChat Creator Companion）経由でインストール済みの場合、次回プロジェクトを開いた際に自動的にアップデートの通知が表示されます。

または、VCCの「Installed Packages」から手動でアップデートしてください。

---

# フィードバックお待ちしています

不具合報告や機能要望は [GitHub Issues](https://github.com/moruton1119/com.morulab.unity-tools/issues) までお願いします！

あなたのフィードバックが、このツールをより良くします。
