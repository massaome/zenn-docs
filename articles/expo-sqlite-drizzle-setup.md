---
title: "Expo SQLiteとDrizzle ORMで始めるReact Nativeのローカルデータベース開発"
emoji: "🗄️"
type: "tech"
topics: ["react-native", "expo", "sqlite", "drizzle", "typescript"]
published: true
published_at: 2025-11-17 08:00
---

## 🎯 はじめに

React Nativeアプリで本格的なローカルデータベースを構築していきたい！
ということで、[Drizzle公式ドキュメント](https://orm.drizzle.team/docs/get-started/expo-new)に基づいて、Expo SQLiteとDrizzle ORMを組み合わせてデータベースを導入してみました。

## 📖 本文

### Step 1: Expoプロジェクトの用意

Expoのプロジェクトを用意、もしくは新しく作成しておきます。
Expoのセットアップについては別記事で書きました。
https://zenn.dev/massaome/articles/expo-setup-2025

### Step 2: Expo SQLiteパッケージのインストール

```bash
# 必要なパッケージのインストール
pnpm expo install expo-sqlite
```

### Step 3: 必要なパッケージのインストール

```bash
# drizzle-ormとdrizzle-kitをインストール
pnpm add drizzle-orm
pnpm add -D drizzle-kit babel-plugin-inline-import
```

### Step 4: データベース接続の設定

`App.tsx`や`_layout.tsx`などのアプリのエントリーポイントでデータベース接続を初期化します。

```typescript:App.tsx
import * as SQLite from 'expo-sqlite';
import { drizzle } from 'drizzle-orm/expo-sqlite';

const expo = SQLite.openDatabaseSync('db.db');
const db = drizzle(expo);
```

### Step 5: テーブルスキーマの作成

```typescript:src/db/schema.ts
import { int, sqliteTable, text } from "drizzle-orm/sqlite-core";

export const usersTable = sqliteTable("users_table", {
  id: int().primaryKey({ autoIncrement: true }),
  name: text().notNull(),
  age: int().notNull(),
  email: text().notNull().unique(),
});
```

### Step 6: Drizzle設定ファイルの作成

Drizzle configは、Drizzle Kitで使用される設定ファイルで、データベース接続、マイグレーションフォルダ、スキーマファイルに関するすべての情報が含まれています。

`./src/db/schema.ts`: スキーマ定義はソースコードなので`src/`に配置
`./drizzle`: 自動生成されるマイグレーションファイルなので、ルートに配置

```typescript:drizzle.config.ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
	dialect: "sqlite",
	driver: "expo",
	schema: "./src/db/schema.ts",
	out: "./drizzle",
});

```

### Step 7: Metro設定の更新

SQLファイルの読み込みを可能にするため、Metro設定を更新します。

```javascript:metro.config.js
const { getDefaultConfig } = require('expo/metro-config');

/** @type {import('expo/metro-config').MetroConfig} */
const config = getDefaultConfig(__dirname);

config.resolver.sourceExts.push('sql');

module.exports = config;
```

### Step 8: Babel設定の更新

マイグレーションファイル（.sqlファイル）をインポートするために、Babel設定の更新を行います。


```javascript:babel.config.js
module.exports = function(api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [["inline-import", { "extensions": [".sql"] }]] // <-- add this
  };
};
```

### Step 9: マイグレーションの生成

スキーマからマイグレーションファイルを生成します。

```bash
npx drizzle-kit generate
```

これによって、Step 6で設定していた`./drizzle`フォルダにマイグレーションファイルが生成されます。

### Step10: アプリの起動と動作確認

データベース操作用のコンポーネント例:

```typescript:MainApp.tsx
import { Text, View } from 'react-native';
import { useEffect, useState } from 'react';
import { usersTable } from './db/schema';
import { db } from './src/provider/DrizzleProvider';

export default function MainApp() {
  const [items, setItems] = useState<typeof usersTable.$inferSelect[]>([]);

  useEffect(() => {
    (async () => {
      // サンプルデータの挿入
      await db.delete(usersTable);
      await db.insert(usersTable).values([
        {
          name: 'John',
          age: 30,
          email: 'john@example.com',
        },
      ]);

      // データの取得
      const users = await db.select().from(usersTable);
      setItems(users);
    })();
  }, []);

  return (
    <View
      style={{
        display: 'flex',
        flexDirection: 'column',
        alignItems: 'center',
        width: '100%',
        height: '100%',
        justifyContent: 'center',
      }}
    >
      {items.map((item) => (
        <Text key={item.id}>{item.email}</Text>
      ))}
    </View>
  );
}
```

### （お好みで）プロバイダーパターンでの実装

これは好みですが、筆者は`App.tsx`や`_layout.tsx`などのエントリーポイントに色々と書くと、関連しているものが分かりにくくなって辛いため、関心の分離として、データベース関連の処理をプロバイダーに用意しています。

まず、データベースプロバイダーを作成します：

```typescript:src/provider/DrizzleProvider.tsx
import { drizzle } from "drizzle-orm/expo-sqlite";
import { useMigrations } from "drizzle-orm/expo-sqlite/migrator";
import { openDatabaseSync } from "expo-sqlite";
import type { ReactNode } from "react";

import migrations from "../../drizzle/migrations";

const expoDb = openDatabaseSync("db.db");
export const db = drizzle(expoDb);

interface DrizzleProviderProps {
  children: ReactNode;
}

export function DrizzleProvider({ children }: DrizzleProviderProps) {
  const { success, error: migrateError } = useMigrations(db, migrations);

  // マイグレーションエラーの処理
  if (migrateError) {
    console.error("Migration Error:", migrateError);
    throw migrateError;
  }

  // マイグレーション完了まで待機
  if (!success) {
    return null; // ローディング表示を検討
  }

  return <>{children}</>;
}
```

次に、`App.tsx`でプロバイダーを使用します：

```typescript:App.tsx
import { DrizzleProvider } from "./src/provider/DrizzleProvider";
import MainApp from "./MainApp";

export default function App() {
  return (
    <DrizzleProvider>
      <MainApp />
    </DrizzleProvider>
  );
}
```

### Drizzle Studioでのデバッグ環境構築

Drizzle Studioを利用することでブラウザ上からデータベースの内容を確認することができます。
GUIを利用してデータベースを確認できる非常に便利なツールです。

Drizzle StudioをReact Nativeで利用するため追加のライブラリをインストールします：

```bash
pnpm add expo-drizzle-studio-plugin
```

`DrizzleProvider.tsx`にDrizzle Studioの設定を追加します：

```diff:src/provider/DrizzleProvider.tsx
import { drizzle } from "drizzle-orm/expo-sqlite";
import { useMigrations } from "drizzle-orm/expo-sqlite/migrator";
+ import { useDrizzleStudio } from "expo-drizzle-studio-plugin";
import { openDatabaseSync } from "expo-sqlite";
import type { ReactNode } from "react";

import migrations from "@/drizzle/migrations";

const expoDb = openDatabaseSync("db.db");
export const db = drizzle(expoDb);

interface DrizzleProviderProps {
  children: ReactNode;
}

export function DrizzleProvider({ children }: DrizzleProviderProps) {
  const { success, error: migrateError } = useMigrations(db, migrations);

+  // Drizzle Studio の設定（開発環境でのみ有効）
+  useDrizzleStudio(expoDb);

  // マイグレーションエラーの処理
  if (migrateError) {
    console.error("Migration Error:", migrateError);
    throw migrateError;
  }

  // マイグレーション完了まで待機
  if (!success) {
    return null; // ローディング表示を検討
  }

  return <>{children}</>;
}
```

**Drizzle Studioの起動手順**：

1. `npx expo start`コマンドを実行してExpo Goを起動

2. ターミナル画面で`Shift + m`を押してメニューを表示

3. "Open expo-drizzle-studio-plugin"を選択

```bash
? Dev tools (native only) › - Use arrow-keys. Return to submit.
    Inspect elements
    Toggle performance monitor
    Toggle developer menu
    Reload app
    Open React devtools
❯   Open expo-drizzle-studio-plugin
```

4. ブラウザが自動で起動し、データベースの中身がGUIで表示されます

**主な機能**：
- **テーブル構造の可視化**: スキーマとリレーションを図で確認
- **データの参照・編集**: GUIでのCRUD操作
- **リアルタイム更新**: アプリでの変更がリアルタイムで反映

:::message alert
Drizzle Studioは開発時のみの使用を想定しており、本番環境では無効化されます。
:::


## 📋 まとめ

Expo SQLiteとDrizzle ORMの組み合わせにより、React Nativeアプリにローカルデータベース環境を構築できました。

## 🔗 参考文献

- [Drizzle ORM - Get Started with Expo](https://orm.drizzle.team/docs/get-started/expo-new)
- [Expo SQLite 公式ドキュメント](https://docs.expo.dev/versions/latest/sdk/sqlite/)
- [Drizzle Kit CLI Reference](https://orm.drizzle.team/kit-docs/overview)
