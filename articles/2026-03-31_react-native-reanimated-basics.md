---
title: "React Native Reanimated 入門 — 全体像をざっくり把握する"
emoji: "🎬"
type: "tech"
topics: ["reactnative", "reanimated", "expo", "animation", "typescript"]
published: true
published_at: 2026-03-31 08:00
---

## はじめに

Expo / React Native でアプリを作っていて、「ボタンを押したときにちょっとフィードバックを付けたい」「モーダルをスッと出したい」みたいな場面が出てきました。ネイティブアプリだと当たり前にある動きを、自分のアプリにも入れたい。でも React Native でアニメーションってどうやるんだろう？　というのがこの記事の出発点です。

調べてみた結果を自分の理解としてまとめたものなので、間違いや「こっちのほうがいいよ」があればぜひコメントで教えてください。

## アニメーション周りにはどんな選択肢があるのか

まず「React Native アニメーション」で調べると、いくつかのライブラリ名がよく出てきます。

- **React Native Reanimated** — UI のスタイル（透明度、位置、スケールなど）を時間変化させる。Expo にも公式で組み込まれている。
- **react-native-gesture-handler** — タッチやスワイプといったジェスチャを認識するためのレイヤー。Reanimated と組み合わせて「指の動きに追従する UI」を作るケースが多い。
- **React Native Skia** — 2D 描画エンジン。キャンバスに絵を描くような表現力が必要なとき向けで、汎用 UI のちょっとした動きとは役割が違いそう。

自分のやりたいことは「画面のコンポーネントをフワッと動かす」くらいのところからなので、まず Reanimated を押さえるのがよさそうだ、というのがわかりました。Gesture Handler は Reanimated と一緒に使う場面が出てきたら改めて見ればいいし、Skia は「絵を描きたい」ときの選択肢、という整理です。

この記事では Reanimated に絞って、調べたこと・試したことをまとめていきます。

:::message
この記事は **Reanimated v4**（react-native-reanimated ~4.x）、**Expo SDK 54** 環境を前提としています。
また、Reanimated v4 は **React Native New Architecture 前提**です。
:::

## 参考文献

https://docs.swmansion.com/react-native-reanimated/

https://docs.expo.dev/versions/latest/sdk/reanimated/

## 本文

### そもそも Reanimated は何を解決するのか

React Native には JS スレッドと UI スレッドがあり、JS 側の処理負荷が高いとアニメーションやジェスチャ連動がカクつくことがあります。

Reanimated は SharedValue とワークレットを使って、アニメーション計算を UI スレッド側で実行できるのが強みです。JS が忙しい場面でも滑らかさを保ちやすく、この「2つのスレッド」の考え方はこのあとの API 理解でも繰り返し出てきます。

### 公式ドキュメントの地図（Core / Animations / Layout / CSS）

Reanimated v4 のドキュメントは、大きく次のカテゴリに分かれています。

- **Core**: `useSharedValue` / `useAnimatedStyle` / `useDerivedValue` など、日常的に使う基本フック
- **Animations**: `withTiming` / `withSpring` / `withDecay` と、`withSequence` などの修飾子
- **Layout Animations**: 要素の追加・削除・レイアウト変更に対するアニメーション（`entering` / `exiting` など）
- **CSS Transitions / CSS Animations**: CSS 的な `transition*` や `animation*` の書き方（iOS/Android/Web 対応）

最初は **Core + Animations** から入り、必要になったタイミングで Layout / CSS 系に広げるのが理解しやすいです。

### まず押さえたい 5 つの基本

ドキュメントを読んでいくと、まずは次の 5 つを押さえると理解しやすいことがわかりました。

| 概念 | 役割 |
|------|------|
| `useSharedValue` | JS スレッドと UI スレッド間で共有される値 |
| `useAnimatedStyle` | SharedValue を参照してスタイルを計算するワークレット |
| アニメーション関数 | `withSpring` / `withTiming` / `withDecay` など、値の時間変化を定義する関数 |
| アニメーション修飾子 | `withSequence` / `withDelay` / `withRepeat` など、アニメーション関数を組み合わせるための関数 |
| `useDerivedValue` | 他の SharedValue や state / props から宣言的に派生する読み取り専用 SharedValue |

入門段階では、この 5 つをセットで理解すると全体像を掴みやすいです。

### useSharedValue — 2つのスレッドをつなぐ値

```typescript
import { useSharedValue } from "react-native-reanimated";

const opacity = useSharedValue(0); // 初期値 0

// JS スレッドから更新できる
opacity.value = 1;
```

`useSharedValue` で作った値は、JS スレッドからも UI スレッドからも読み書きできます。
通常の `useState` と違い、値を変えてもコンポーネントの再レンダリングは発生しません。これがパフォーマンスの鍵です。

### useAnimatedStyle — スタイルを計算するワークレット

```typescript
import Animated, { useAnimatedStyle, useSharedValue } from "react-native-reanimated";

function FadeBox() {
  const opacity = useSharedValue(0);

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value, // SharedValue を参照
  }));

  return <Animated.View style={animatedStyle} />;
}
```

`useAnimatedStyle` に渡す関数は **ワークレット** として UI スレッドで実行されます。
`opacity.value` が変化するたびに自動的に再計算されてスタイルが更新されます。

`Animated.View`（または `Animated.Text` など）に `style={animatedStyle}` を渡すことで、このアニメーションが有効になります。

:::message
**ワークレットとは**
Reanimated では、UI スレッドで直接実行できる関数を**ワークレット**と呼びます。`useAnimatedStyle` や `useDerivedValue` に渡すコールバックは自動的にワークレットとして扱われます。通常の関数をワークレットにするには先頭に `'worklet'` ディレクティブを書きます。

```typescript
function clamp(value: number, min: number, max: number) {
  'worklet';
  return Math.min(Math.max(value, min), max);
}
```

「`useAnimatedStyle` の中身が UI スレッドで動いている」という感覚を持つと、なぜ `useSharedValue` が必要なのかが腑に落ちやすくなります。
:::


---

### アニメーション関数 / 修飾子 — 値をどう変化させ、どう組み合わせるか

値を更新するとき、ただ代入するのではなくアニメーション関数でラップすると動きが生まれます。

`withTiming` / `withSpring` / `withDecay` は **アニメーション関数**（値の変化そのものを定義）です。  
`withSequence` / `withDelay` / `withRepeat` などは **アニメーション修飾子**（関数を組み合わせて動きを構成）です。

#### withTiming — 一定時間で変化

```typescript
import { withTiming } from "react-native-reanimated";

opacity.value = withTiming(1, { duration: 300 }); // 300ms かけて 0 → 1
```

持続時間とイージングを指定できます。
完了タイミングを把握しやすいため、シーケンスや条件分岐に組みやすいのが特徴です。

#### withSpring — バネ物理で変化

```typescript
import { withSpring } from "react-native-reanimated";

// シンプルに使う
scale.value = withSpring(1.2);

// パラメータで挙動を調整
scale.value = withSpring(1.2, {
  damping: 12,    // 減衰（大きいほど揺れが少ない）
  stiffness: 300, // 硬さ（大きいほど速い）
  overshootClamping: true, // 目標値を超えないようにする
});
```

**damping の目安（`stiffness: 300` 前後で試した感覚）：**
- `damping: 6〜10` → バウンシー（弾む感じ）
- `damping: 15〜20` → 自然な動き
- `damping: 30以上` → ほぼ揺れなし

`stiffness` や `mass` の値によって体感が変わるので、あくまで参考程度に。

`overshootClamping: true` を付けると目標値を超えないので、プログレスバーや幅のアニメーションに適しています。

#### withSequence（修飾子）— 複数のアニメーションを順番に

```typescript
import { withSequence, withTiming, withSpring } from "react-native-reanimated";

// 1.35倍に拡大してから元に戻す（Twitter いいねのような動き）
scale.value = withSequence(
  withTiming(1.35, { duration: 100 }),  // ① 拡大（withTiming で確実に完了させる）
  withSpring(1, { damping: 12, stiffness: 300, overshootClamping: true }), // ② 戻る
);
```

### useDerivedValue — 他の値から導出される派生値

`useSharedValue` との違いを先に整理します。

| | `useSharedValue` | `useDerivedValue` |
|---|---|---|
| 値の管理 | 自分で読み書きする | 計算式で決まる（読み取り専用） |
| 更新タイミング | 自分で代入する | 依存する値が変わると自動 |

`useDerivedValue` に渡す updater 関数は、内部で参照している SharedValue やプロップが変わるたびに UI スレッド上で再実行されます。

```typescript
import { useDerivedValue, useSharedValue, withSpring } from "react-native-reanimated";

const progress = useDerivedValue(() =>
  withSpring(completed / total),
);
```

`withSpring` は**「新しい目標値へどう移動するか」を指定するもの**で、アニメーションを発火させるのは `useDerivedValue` の再計算です。`completed` や `total` が変わる → updater が再実行される → `withSpring` が新しい目標値に向けてアニメーションを走らせる、という流れです。

プロップに連動するアニメーション値にはこのパターンを使うとうまくいきました。

---

### 実践

#### チェックボックスのポップアニメーション

Twitter のいいねボタンのような「押したらポンと膨らんで戻る」動きです。

以下は説明用の抜粋なので、`Pressable` や `CheckIcon` など必要な import / 実装は適宜補ってください。

```typescript
import Animated, {
  useAnimatedStyle,
  useSharedValue,
  withSequence,
  withSpring,
  withTiming,
} from "react-native-reanimated";

function CheckButton({ onPress }: { onPress: () => void }) {
  const scale = useSharedValue(1);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  const handlePress = () => {
    scale.value = withSequence(
      withTiming(1.35, { duration: 100 }),
      withSpring(1, { damping: 12, stiffness: 300, overshootClamping: true }),
    );
    onPress();
  };

  return (
    <Animated.View style={animatedStyle}>
      <Pressable onPress={handlePress}>
        <CheckIcon />
      </Pressable>
    </Animated.View>
  );
}
```

---

#### プログレスバーのスプリングアニメーション

値が変わるたびにバーが滑らかに伸び縮みするプログレスバーです。

```typescript
import { View } from "react-native";
import Animated, {
  useAnimatedStyle,
  useDerivedValue,
  useSharedValue,
  withSpring,
} from "react-native-reanimated";

type Props = {
  completed: number;
  total: number;
};

export function ProgressBar({ completed, total }: Props) {
  const containerWidth = useSharedValue(0); // バーの実際の横幅
  const progress = useDerivedValue(() =>
    withSpring(total > 0 ? completed / total : 0, {
      damping: 30,
      stiffness: 200,
      overshootClamping: true, // 0〜1 の範囲を超えないよう固定
    }),
  );

  // progress=0 → translateX=-containerWidth（見えない）、progress=1 → translateX=0（全幅）
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: (progress.value - 1) * containerWidth.value }],
  }));

  return (
    <View
      style={{ height: 24, borderRadius: 12, overflow: "hidden", backgroundColor: "#e5e7eb" }}
      onLayout={(e) => {
        containerWidth.value = e.nativeEvent.layout.width;
      }}
    >
      <Animated.View
        style={[
          { width: "100%", height: "100%", borderRadius: 12, backgroundColor: "#3b82f6" },
          animatedStyle,
        ]}
      />
    </View>
  );
}
```

**ポイント：** `width` プロパティをアニメーションすると子要素のレイアウト変化が親に伝播し、`onLayout` が再発火するループが起きることがあります。代わりに `translateX` + `overflow: hidden` の組み合わせでレイアウトを変化させずにアニメーションします。

## まとめ

調べてみて一番「なるほど」と思ったのは `useDerivedValue` の存在でした。「アニメーション値は SharedValue や state / props から導出される値だ」という整理ができると、どこに何を書けばいいかが自然に決まってくる気がします。

Gesture Handler と組み合わせた「指の動きに追従する UI」やスクロール連動アニメーションはまだ手を付けていないので、そのあたりは試せたらまた書くかもしれません。
