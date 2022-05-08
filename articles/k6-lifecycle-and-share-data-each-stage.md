---
title: "k6 VU 間で 共通の data を共有する。❌グローバル変数"
emoji: "🐊"
type: "tech"
topics: ["k6", "loadtest", "node"]
published: true
---

### はじめに

k6 で負荷テストを書いている時、VU 間で共有できる、グローバルな変数をスクリプトファイルには記述できません。
通常の node の実行ではスクリプトファイルでグローバル変数と解釈される部分も、k6 で実行される際には、VU (Virtual User) 回数実行されます。 例えば、ログインが必要な API の負荷テストをするとき、VU 回 ログインして認証情報を取得するのは、正確な対象 API の負荷テストにはなりません。ログイン処理は、最初の１回で認証情報を取得し、VU 間で共有して使用する方法を見つけたので共有します。

:::details lifecycle について

![k6-lifecycle-summary.png](/images/k6-lifecycle-summary.png)

```js
// 1. init code
console.log(`VU: ${__VU}`);

export function setup() {
  // 2. setup code
  console.log("setup code: Set up data for processing, share data among VUs");
}

export default () {
  // 3. VU code
  console.log(`SCENARIO: ${exec.scenario.name} :VU: ${__VU}  -  ITER: ${__ITER}`);
}

export function teardown() {
  // 4. teardown code
  console.log("teardown code: Process result of setup code, stop test environment");
}
```

:::

### 結論

![k6-date-lifecycle.png](/images/k6-date-lifecycle.png =300x)

lifecycle の ドキュメントにそのまま書いてありました。  
setup() の戻り値を default, teardown() の引数として使用できます。
<https://k6.io/docs/using-k6/test-life-cycle/#setup-and-teardown-stages>

```js
export function setup() {
  return { v: 1 };
}

export default function (data) {
  console.log(JSON.stringify(data));
}

export function teardown(data) {
  if (data.v != 1) {
    throw new Error("incorrect data: " + JSON.stringify(data));
  }
}
```

### 検討した他の方法

結論の情報をなかなか見つけられなかったので、他の方法も考えていました。

##### ❌ 環境変数を使用する方法

環境変数は、VU 間で共有できますが、k6 の スクリプトの実行の外から渡す必要があります。
--env (-e) オプションを使用できます。<https://k6.io/docs/using-k6/environment-variables/>

```
k6 run -e SESSION_ID="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

```ts
export type Envs = {
  SESSION_ID: string;
};

const envs: envs = {
  SESSION_ID: __ENV.SESSION,
};
```

##### ❌ ローカルに別のサーバーを立てて、VU 毎に参照する

ローカルに別のサーバーを立てて、setup()で動的な情報を POST、VU 毎に GET で値を参照する、 この方法であれあば、手順をコード化でき、負荷テスト対象のサーバーに余計な負荷をかけることもないですが。ローカルで大量の I/O が発生するため、何か影響がないか不安です。また、この方法では、最初はサーバーではなく、適当な /tmp/ファイルに保存すればいいと思いましたが、k6 は、node の native module をサポートしてないので、"path", "fs", "os" とう使うことができません。

```ts
export function setup() {
  // POST session id
}

// 1. init code
// GET session id
```

### 終わりに

今回紹介した、lifecycle や data 共有を使用した example を以下に実装しています。

- [lifecycle_test.ts](https://github.com/kis9a/k6-template-typescript/blob/master/src/scenarios/examples/lifecycle_test.ts)
- [random_sleep.ts](https://github.com/kis9a/k6-template-typescript/blob/master/src/scenarios/examples/random_sleep.ts)

k6 のシナリオは通常 Javasript で記述しますが、Typescript で記述し esbuild で Javascript に bundle する template についても実装していますので、よかったら試してみてください。

https://github.com/kis9a/k6-template-typescript
