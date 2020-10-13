---
title: "Firestore RulesのテストをJestで書くテスト"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "firestore", "javascript", "jest"]
published: true
---
# 結論
Jest使うのはやめた方が良さそう(拘りが無い限り)

という残念な結果に至ったが、Firestore Rulesにおけるテストの始め方を記していく。恐らく他のJSテストフレームワークでも大差は無い。
初学者向けの説明となるので興味無い方は「本題」の章に飛んで頂きたい。

# 目標
最初のテストを書く。
細かい設定等は端折っているので、そちらは適時公式ドキュメント等を参照して頂きたい。

# 前提
当記事はFirestoreを用いたJSプロジェクト作成済みの状態を前提とする。

また、Firebase JavaScript SDK は以下の環境を前提としている。
- Node.js: `10.15.0` or greater
- yarn: `1.0.0` or greater
- java: `1.8.0` or greater

Java8+ が必要なことに注意。未導入の場合は下記からインストール(要サインアップ)
[Java SE Development Kit 8 Downloads](https://www.oracle.com/java/technologies/javase/javase-jdk8-downloads.html)

# 事前準備

## 導入するライブラリ
1. [@firebase/rules-unit-testing](https://firebase.google.com/docs/firestore/security/test-rules-emulator) (ver 1.0.4)
FirebaseSDK に含まれるテスト用SDK。
2020/08 からパッケージ名称が`@firebase/testing`から`@firebase/rules-unit-testing`に変更されている。

2. [Jest](https://jestjs.io/) (ver 26.4.2)
[世界で最も使われているJSテストフレームワーク](https://ashleynolan.co.uk/blog/frontend-tooling-survey-2019-results#js-testing)の1つ。Facebook製。
Node.js上で[JSDOM](https://github.com/jsdom/jsdom)というブラウザのエミュレーション環境を用いてテストが実行される。
Jestのお作法は下記の記事が参考になる。
  [Facebook製のJavaScriptテストツール「Jest」の逆引き使用例](https://qiita.com/chimame/items/e97883fd46b67529d59)

ということで、プロジェクト直下で下記を実行。
`yarn add --dev @firebase/rules-unit-testing jest`

## エミュレーターの準備
次に `firebase init` を実行。
> ? Which Firebase CLI features do you want to set up for this folder? Press Space to select features, then Enter to confirm your choices.

と聞かれるので `Emulators` を選択。
この時、まだfirestoreをinitしていないなら一緒にセットアップする。
（プロジェクト直下に `firestore.rules` が存在しない場合）
既にブラウザコンソールでRulesを開発済みの場合でも、ローカルの `firestore.rules` と同期を取ってくれる。

その後の設問はデフォルト値でOK。
> ? Which port do you want to use for the functions emulator? 5001
? Which port do you want to use for the firestore emulator? 8080
? Would you like to enable the Emulator UI? Yes
? Which port do you want to use for the Emulator UI (leave empty to use any available port)? NaN

`firebase emulators:start` を実行するとエミュレート環境が立ち上がる。ローカル上に擬似的なFirebase環境を構築するようなイメージ。Firestoreのみテストしたい場合は `firebase emulators:start --only firestore`を実行する。

[http://localhost:4000/firestore](http://localhost:4000/firestore) にアクセスして、Firebase Emulator Suiteが表示されれば成功。
![Firebase Emulator Suite](https://storage.googleapis.com/zenn-user-upload/ch4esd0zii31x24n4cltpzwtpun5)
画面上からモックデータを登録することも出来るが、どういうシチュエーションで使うのかよく分かってない・・・

## Jestの設定
- **テストファイルの作成**
Jestのデフォルトでは `*.test.js`、 `*.spec.js`、もしくは`__tests__` 配下にある全てのファイルをテストファイルとして検出する。拡張子は`.js` `.ts` `.jsx` `.tsx`に対応している。
ここでは `__tests__/firestore.test.js` を作成する事とする。
- **npm-scriptの設定**
`package.json` にテスト実行用のスクリプトを定義する。
```json
"scripts": {
  "test": "jest"
},
```
### Jestの基本的なAPI
最初のテストを書くうえで必要なところだけ抑えておく。
（分かりやすい様にTypeScriptの記法でシグネチャを記載するが、後述のサンプルコードはJSで実装する）

- **describe(name:string, fn:Function)**
テストの見出しのようなもの。機能単位やテストの性質単位、Firestoreだとコレクション単位等で分類分けする際に用いる。
基本的には`fn`の中にテストを記載していくこととなる。後述するテストサイクルのスコープも担う。

- **test(name:string, fn:Function, timeout:number?)**
テストを実行する際に用いる関数。`fn`の返却値からテストの成否を判定する。
`it`という、`test`と全く同じ機能の関数も存在する（他テストフレームワークでは`it`という命名が多用されていることへの配慮）

- **beforeAll(fn:string, timeout:number?)**
テスト開始直後に1回だけ呼ばれる。`describe`を用いてスコープを切ることが可能。

- **beforeEach(fn:string, timeout:number?)**
`test`開始前に毎回呼ばれる。

- **afterAll(fn:string, timeout:number?)**
テスト終了前に1回だけ呼ばれる。`describe`を用いてスコープを切ることが可能。

- **afterEach(fn:string, timeout:number?)**
`test`終了後に毎回呼ばれる。

スコープについては[公式チュートリアル](https://jestjs.io/docs/ja/26.4/setup-teardown#%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%97)の例が分かりやかったので下記に引用する。
>
>```js
>beforeAll(() => console.log('1 - beforeAll'));
>afterAll(() => console.log('1 - afterAll'));
>beforeEach(() => console.log('1 - beforeEach'));
>afterEach(() => console.log('1 - afterEach'));
>test('', () => console.log('1 - test'));
>describe('Scoped / Nested block', () => {
>  beforeAll(() => console.log('2 - beforeAll'));
>  afterAll(() => console.log('2 - afterAll'));
>  beforeEach(() => console.log('2 - beforeEach'));
>  afterEach(() => console.log('2 - afterEach'));
>  test('', () => console.log('2 - test'));
>});
>
>// 1 - beforeAll
>// 1 - beforeEach
>// 1 - test
>// 1 - afterEach
>// 2 - beforeAll
>// 1 - beforeEach
>// 2 - beforeEach
>// 2 - test
>// 2 - afterEach
>// 1 - afterEach
>// 2 - afterAll
>// 1 - afterAll
>```

:::message
通常、Jestではテストの評価をする際 `expect`というメソッドを用いるが、今回はその機能をFirebase SDKで代用する形となる。
:::

# テストの作成

## テスト対象のFirestore Rules
下記の簡単なRulesを例としてテストを書いていく。
```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null;
      allow update: if request.auth.uid == userId;
    }
  }
}
```
- 認証済みユーザーなら読み取りと作成可
- 自身のUIDとドキュメントIDが同値なら更新可
- 削除不可

## ローカルのRulesファイル読み込み
まずはテスト対象となるRulesファイルの読み込みを行う。
読み込みはNode標準モジュールである`fs`を用いて`loadFirestoreRules`に渡す。
これはテスト開始時に1回だけ行えばいいので`beforeAll`で実行する。
```js
// ローカルファイル操作用Nodeモジュール
const fs = require('fs')
// Firebase SDK テストユーティリティ
const firebase = require('@firebase/rules-unit-testing')

// FirebaseのProject ID
const PROJECT_ID = 'my-project'

// テスト開始時に1回だけ実行される
beforeAll(async () => {
  // ローカルにあるRulesファイルを読み込む
  const rules = fs.readFileSync('firestore.rules', 'utf8')
  await firebase.loadFirestoreRules({ projectId: PROJECT_ID, rules })
})
```

## 環境変数の設定(任意)
環境変数 `FIRESTORE_EMULATOR_HOST`に、エミュレーターのホスト名:ポート番号を指定する。今回の様にデフォルトポートで立ち上げた場合は不要だが、「デフォルトポートを使うよ」という旨のワーニングが毎回出る。
> Warning: FIRESTORE_EMULATOR_HOST not set, using default value localhost:8080

煩わしいので、コードで環境変数の設定をしておく。
```js
// エミュレーターホスト指定
process.env.FIRESTORE_EMULATOR_HOST = 'localhost:8080'
```

## カバレッジレポートの出力(任意)
Firebaseのテストユーティリティにはカバレッジレポートの出力機能が存在する。
テスト実行後にエミュレータをブラウザで確認すれば表示されるが、実行の都度ローカルへ保存するようにする。`afterAll`にて行う。
```js
// HTTP通信用Nodeモジュール
const http = require('http')

// カバレッジレポート出力URL
const COVERAGE_URL = `http://${process.env.FIRESTORE_EMULATOR_HOST}/emulator/v1/projects/${PROJECT_ID}:ruleCoverage.html`

// テスト終了前に1回だけ実行される
afterAll(async () => {
  // カバレッジレポートの出力
  const coverageFile = 'firestore-coverage.html'
  const fstream = fs.createWriteStream(coverageFile)
  await new Promise((resolve, reject) => {
    http.get(COVERAGE_URL, (res) => {
      res.pipe(fstream, { end: true })
      res.on('end', resolve)
      res.on('error', reject)
    })
  })
})
```

## Firestoreの初期化/削除
Firestoreの初期化は`initializeTestApp()`を使う。オプションとして認証情報を渡せば、認証済みユーザーとして初期化されたアプリ情報が返却される。
これはテストの都度用いる事になるのでグローバル関数として登録しておくと便利。
```js
// Firestoreの初期化
function getAuthedFirestore(auth) {
  return firebase.initializeTestApp({ projectId: PROJECT_ID, auth }).firestore()
}
```
`getAuthedFirestore({uid: 'alice'})` の様に呼び出せば、UIDが`alice`の認証済みユーザーとしてfirestoreオブジェクトを使うことができる。
認証されてないユーザーとしてテストを行いたい場合は引数に`null`を渡す。

この情報はエミュレーターのメモリ上に残り続ける為、テストの再実行性を考慮して最後には必ずアプリ情報を削除するようにする。
```js
// テスト終了前に1回だけ実行される
afterAll(async () => {
  // エミュレーター上に作られたアプリ情報を全て消去する
  await Promise.all(firebase.apps().map((app) => app.delete()))

  // カバレッジレポートの出力
  const coverageFile = 'firestore-coverage.html'
  //~~(省略)~~
```

ここまでがテンプレートのようなもので、ソースはこんな感じになる。
:::details __tests__/firestore.test.js
```js
// ローカルファイル操作用Nodeモジュール
const fs = require('fs')
// HTTP通信用Nodeモジュール
const http = require('http')
// Firebase SDK テストユーティリティ
const firebase = require('@firebase/rules-unit-testing')

// FirebaseのProject ID
const PROJECT_ID = 'my-project'

// エミュレーターホスト指定
process.env.FIRESTORE_EMULATOR_HOST = 'localhost:8080'

// カバレッジレポート出力URL
const COVERAGE_URL = `http://${process.env.FIRESTORE_EMULATOR_HOST}/emulator/v1/projects/${PROJECT_ID}:ruleCoverage.html`

// Firestoreの初期化
function getAuthedFirestore(auth) {
  return firebase.initializeTestApp({ projectId: PROJECT_ID, auth }).firestore()
}

// テスト開始時に1回だけ実行される
beforeAll(async () => {
  // ローカルにあるRulesファイルを読み込む
  const rules = fs.readFileSync('firestore.rules', 'utf8')
  await firebase.loadFirestoreRules({ projectId: PROJECT_ID, rules })
})

// テスト終了前に1回だけ実行される
afterAll(async () => {
  // エミュレーター上に作られたアプリ情報を全て消去する
  await Promise.all(firebase.apps().map((app) => app.delete()))

  // カバレッジレポートの出力
  const coverageFile = 'firestore-coverage.html'
  const fstream = fs.createWriteStream(coverageFile)
  await new Promise((resolve, reject) => {
    http.get(COVERAGE_URL, (res) => {
      res.pipe(fstream, { end: true })
      res.on('end', resolve)
      res.on('error', reject)
    })
  })
})

```
:::

## テストを書く
まずは`describe`でスコープを切る。分類名は仮で「最初のテスト」としておく。
この際、テストの都度Firestoreのデータを全消去するようにする。
```js
describe('最初のテスト', () => {
  // テストが完了する度データをクリア
  afterEach(async () => {
    await firebase.clearFirestoreData({ projectId: PROJECT_ID })
  })
})
```
毎回ゼロスタートにしておかないと、他テストで投入された予期せぬデータが邪魔をすることがある。グローバル登録しても良いが、テストケースによってはデータを引き継ぎたい場合も出てくるのでスコープ内で定義した方が無難な処理。

続いて「自身のUIDとドキュメントIDが一致する場合、登録できること」のテストを書く。`test`関数の中でFirebase SDKのテストユーティリティを叩く実装となる。
`firebase.assertSucceeds()`で成功、`firebase.assertFails()`で失敗した場合にテスト合格としてPromiseが返却される。
```js
describe('最初のテスト', () => {
  // テストが完了する度データをクリア
  afterEach(async () => {
    await firebase.clearFirestoreData({ projectId: PROJECT_ID })
  })
  test('SET - Authed', async () => {
    const db = getAuthedFirestore({ uid: 'alice' })
    const usersRef = db.collection('users').doc('alice')
    // セット出来ること
    await firebase.assertSucceeds(
      usersRef.set({
        owner: 'alice',
        createAt: firebase.firestore.FieldValue.serverTimestamp(),
      })
    )
  })
})
```

## テストを実行してみる
ここまで出来たらエミュレーターを起動したまま、別窓で`yarn test`を実行する。
テストに成功すると下記が出力される。
![Jest成功ターミナル画面](https://storage.googleapis.com/zenn-user-upload/jqsy59hisf2ldtiei12l8vthw860 =450x)

ローカルの `firestore-coverage.html` もしくは [http://localhost:8080/emulator/v1/projects/my-project:ruleCoverage.html](http://localhost:8080/emulator/v1/projects/my-project:ruleCoverage.html) にアクセスするとカバレッジレポートを確認できる。
![カバレッジレポート](https://storage.googleapis.com/zenn-user-upload/lybtgemmrlhshyz64vxdbk1j4gje =450x)
確認したいルールにhoverすると、通過した頻度とルールから返却された値が表示される。

## テストパターンを網羅する
今回のRulesだと、usersコレクションに対して
- `get` `list` `create` `delete` の4パターン × 認証済みユーザー、認証されてないユーザー の2パターン
- `update` で ユーザーID一致、不一致の2パターン

で、計10パターンのテストを用意する。
読み取り許可してるのは `read`(getとlistの糖衣構文)なので、どちらか片方で良い気もするが、後々ルールを変更した時を考えると出来るだけ細分化して書いた方がベター。

以下、テストコード全文を記載する。
:::details __tests__/firestore.test.js
```js
// ローカルファイル操作用Nodeモジュール
const fs = require('fs')
// HTTP通信用Nodeモジュール
const http = require('http')
// Firebase SDK テストユーティリティ
const firebase = require('@firebase/rules-unit-testing')

// FirebaseのProject ID
const PROJECT_ID = 'my-project'

// カバレッジレポート出力URL
const FIRESTORE_EMULATOR_HOST = 'localhost:8080'
const COVERAGE_URL = `http://${FIRESTORE_EMULATOR_HOST}/emulator/v1/projects/${PROJECT_ID}:ruleCoverage.html`

process.env.FIRESTORE_EMULATOR_HOST = FIRESTORE_EMULATOR_HOST

// Firestoreの初期化
function getAuthedFirestore(auth) {
  return firebase.initializeTestApp({ projectId: PROJECT_ID, auth }).firestore()
}

// テスト開始時に1回だけ実行される
beforeAll(async () => {
  // ローカルにあるRulesファイルを読み込む
  const rules = fs.readFileSync('firestore.rules', 'utf8')
  await firebase.loadFirestoreRules({ projectId: PROJECT_ID, rules })
})

// テスト終了前に1回だけ実行される
afterAll(async () => {
  // エミュレーター上に作られたアプリ情報を全て消去する
  await Promise.all(firebase.apps().map((app) => app.delete()))

  // カバレッジレポートの出力
  const coverageFile = 'firestore-coverage.html'
  const fstream = fs.createWriteStream(coverageFile)
  await new Promise((resolve, reject) => {
    http.get(COVERAGE_URL, (res) => {
      res.pipe(fstream, { end: true })
      res.on('end', resolve)
      res.on('error', reject)
    })
  })
})

describe('最初のテスト', () => {
  // テストが完了する度データをクリア
  afterEach(async () => {
    await firebase.clearFirestoreData({ projectId: PROJECT_ID })
  })
  test('SET - Authed', async () => {
    const db = getAuthedFirestore({ uid: 'alice' })
    const usersRef = db.collection('users').doc('alice')
    // セット出来ること
    await firebase.assertSucceeds(
      usersRef.set({
        owner: 'alice',
        createAt: firebase.firestore.FieldValue.serverTimestamp(),
      })
    )
  })

  test('SET - Not Authed', async () => {
    const db = getAuthedFirestore(null)
    const usersRef = db.collection('users').doc('alice')
    // セット出来ないこと
    await firebase.assertFails(
      usersRef.set({
        owner: 'alice',
        createAt: firebase.firestore.FieldValue.serverTimestamp(),
      })
    )
  })

  test('UPDATE - Matched UserID', async () => {
    const db = getAuthedFirestore({ uid: 'alice' })
    const usersRef = db.collection('users').doc('alice')
    await usersRef.set({
      owner: 'alice',
      createAt: firebase.firestore.FieldValue.serverTimestamp(),
    })
    // 更新出来ること
    await firebase.assertSucceeds(
      usersRef.update({
        owner: 'alice',
        updateAt: firebase.firestore.FieldValue.serverTimestamp(),
      })
    )
  })

  test('UPDATE - Unmatched UserID', async () => {
    // 他者のデータを登録
    const _db = getAuthedFirestore({ uid: 'bob' })
    const _usersRef = _db.collection('users').doc('bob')
    await _usersRef.set({
      owner: 'bob',
      createAt: firebase.firestore.FieldValue.serverTimestamp(),
    })
    const db = getAuthedFirestore({ uid: 'alice' })
    const usersRef = db.collection('users').doc('bob')
    // 更新出来ないこと
    await firebase.assertFails(
      usersRef.update({
        owner: 'bob',
        updateAt: firebase.firestore.FieldValue.serverTimestamp(),
      })
    )
  })

  test('DELETE - Authed', async () => {
    const db = getAuthedFirestore({ uid: 'alice' })
    const usersRef = db.collection('users').doc('alice')
    await usersRef.set({
      owner: 'alice',
      createAt: firebase.firestore.FieldValue.serverTimestamp(),
    })
    // 削除出来ないこと
    await firebase.assertFails(usersRef.delete())
  })

  test('DELETE - Not Authed', async () => {
    const _db = getAuthedFirestore({ uid: 'alice' })
    const _usersRef = _db.collection('users').doc('alice')
    await _usersRef.set({
      owner: 'alice',
      createAt: firebase.firestore.FieldValue.serverTimestamp(),
    })
    const db = getAuthedFirestore(null)
    const usersRef = db.collection('users').doc('alice')
    // 削除出来ないこと
    await firebase.assertFails(usersRef.delete())
  })

  test('GET - Authed', async () => {
    const db = getAuthedFirestore({ uid: 'alice' })
    const usersRef = db.collection('users').doc('alice')
    // 1件取得出来ること
    await firebase.assertSucceeds(usersRef.get())
  })

  test('GET - Not Authed', async () => {
    const db = getAuthedFirestore(null)
    const usersRef = db.collection('users').doc('alice')
    // 1件取得出来ること
    await firebase.assertFails(usersRef.get())
  })

  test('LIST - Authed', async () => {
    const db = getAuthedFirestore({ uid: 'alice' })
    const usersRef = db.collection('users')
    // 全件取得出来ること
    await firebase.assertSucceeds(usersRef.get())
  })

  test('LIST - Not Authed', async () => {
    const db = getAuthedFirestore(null)
    const usersRef = db.collection('users')
    // 全件取得出来ること
    await firebase.assertFails(usersRef.get())
  })
})
```
:::

一部( `UPDATE - Unmatch UserID`等 )、違うユーザーのアプリ情報を用意してデータ登録 → テスト用のアプリ情報を用意してテスト という回りくどい書き方をしているが、ここはFirebase Admin SDKを用いればもっと簡潔に書ける(今回は触れない)

# 本題
前置きが長くなってしまったが、ここからが本題。
上記テストコードを実行すると以下のエラーが発生する。
> @firebase/firestore: Firestore (7.21.1): FIRESTORE (7.21.1) INTERNAL ASSERTION FAILED: Unexpected state

これはissueが立てられており、既にClose済み。
原因はFirebase SDKではなくJestのバグであると結論づいてる。
[FIRESTORE (7.14.3) INTERNAL ASSERTION FAILED: value must be undefined or Uint8Array](https://github.com/firebase/firebase-js-sdk/issues/3096#issuecomment-637176584)

該当のバグはこのissueとなるが、2019/02にOpenして2020/10現在も閉じられていない。
[ArrayBuffer regression in node env](https://github.com/facebook/jest/issues/7780)

## 対策
上記Firebaseのissueにて中の人が教示してくれているが、下記の対応をすれば回避できる。
- `yarn add --dev jest-environment-node` を実行
- `__test-utils__/custom-jest-environment.js` を作成し、下記を実装する
:::details __test-utils__/custom-jest-environment.js
```js
'use strict'

const NodeEnvironment = require('jest-environment-node')

class MyEnvironment extends NodeEnvironment {
  constructor(config) {
    super(
      Object.assign({}, config, {
        globals: Object.assign({}, config.globals, {
          Uint32Array,
          Uint8Array,
          ArrayBuffer,
        }),
      })
    )
  }

  async setup() {}

  async teardown() {}
}

module.exports = MyEnvironment
```
:::
- `package.json` に下記を追記する
```json
  "jest": {
    "testEnvironment": "./__test-utils__/custom-jest-environment.js"
  },
```
これで正常にテストがパスされるはず。
![テストオールグリーン](https://storage.googleapis.com/zenn-user-upload/6c9i9rxl79qzdue6fb0jxnpuywqj =450x)

とはいえ、バグ回避の為だけのモジュールを置いておくのは気持ちが悪い。
[公式のクイックスタート](https://github.com/firebase/quickstart-testing/tree/master/unit-test-security-rules)では[Mocha](https://mochajs.org/)を採用している。大人しくこっち使った方が良さそう。

# おわり
業務で扱う場合はもちろん、「個人開発だからテストは書かない」という人もRulesのテストだけはちゃんと書いた方がいいです。隙を見せるとすぐゴミデータを投入されますし、何よりRulesの開発はテスト環境が無いと辛いです。[debug関数](https://firebase.google.com/docs/reference/rules/rules.debug)使えないですし。むしろTDDでやった方が開発効率が上がります。
あと、Jestはやめた方が良いと結論づけましたが、既に導入済みであれば無理して別フレームワークを導入する必要もないと思います。
Firestoreはいいぞ。