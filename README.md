# 学習ログアプリ

学習時間の記録・生活ログ・カレンダー集計ができるPWA（Progressive Web App）です。

## 主な機能

- 学習タイマー（複数同時計測対応）
- 科目別の自由設定・集計
- 生活ログ（起床・就寝・スクリーンタイム）
- カレンダーでの過去の記録閲覧
- 勉強時間グラフ・睡眠リズムグラフなどのインサイト
- ダークモード（システム連動 / 手動切り替え）
- Firebase Authenticationによるメール認証ログイン
- ご意見箱（管理者への問い合わせ・返信機能）

---

## 公開・利用パターンについて

このリポジトリの使い方は大きく2通りあります。どちらに該当するか確認してから進めてください。

### パターン① このまま自分の学習ログサイトとして運用する

`firebaseConfig` は変更不要です。今動いているFirebaseプロジェクトにそのまま接続され続けます。

```
GitHub（このコード） → 既存のFirebaseプロジェクト → 本番データ
```

この場合、必須対応は **「Firestoreセキュリティルールの設定」のみ** です（後述）。
`firebaseConfig` の値自体はクライアントに必ず露出する設計のため、Firestoreルールさえ正しく設定されていれば、GitHubで公開しても実害はありません。

### パターン② 誰でも使えるテンプレートとして配布する

他の人が自分のFirebaseプロジェクトに繋いで使えるようにしたい場合は、`firebaseConfig` をプレースホルダーに置き換えてください。

```js
const firebaseConfig = {
  apiKey:            "YOUR_API_KEY",
  authDomain:        "YOUR_PROJECT.firebaseapp.com",
  projectId:         "YOUR_PROJECT_ID",
  storageBucket:     "YOUR_PROJECT.firebasestorage.app",
  messagingSenderId: "YOUR_SENDER_ID",
  appId:             "YOUR_APP_ID",
};
```

利用者は自分のFirebaseプロジェクトを作成し、この値を書き換えて使う形になります（`index.html` と `login_index.html` の両方）。

---

## セットアップ

### 1. Firebaseプロジェクトを用意

1. [Firebase Console](https://console.firebase.google.com/) でプロジェクトを作成（パターン①の場合は既存のものを利用）
2. **Authentication** → Sign-in method で「メール/パスワード」を有効化
3. **Firestore Database** を作成（本番モードでOK。ルールは手順2で設定）

### 2. Firestoreセキュリティルール（必須）

Firebase Console → Firestore Database → ルール で、以下を設定してください。
（このリポジトリの `firestore.rules` ファイルと同じ内容です）

```
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn() {
      return request.auth != null;
    }

    function isAdminUser() {
      return signedIn()
        && get(
          /databases/$(database)/documents/users/$(request.auth.uid)
        ).data.isAdmin == true;
    }

    match /users/{userId}/{document=**} {

      allow read, delete: if signedIn()
                           && request.auth.uid == userId;

      // 新規作成時、isAdmin を true にして自分で管理者になりすますことを防ぐ
      allow create: if signedIn()
                    && request.auth.uid == userId
                    && (
                      !('isAdmin' in request.resource.data)
                      || request.resource.data.isAdmin == false
                    );
    }

    match /users/{userId} {

      // isAdmin フィールドは自分自身では変更できない
      // （Firebase Console から手動で設定する運用にするため）
      allow update: if signedIn()
                    && request.auth.uid == userId
                    && request.resource.data.isAdmin
                       == resource.data.isAdmin;
    }

    match /users/{userId}/{subcollection}/{document=**} {

      // days/tasks などサブコレクションは自由に書き込み可能
      allow write: if signedIn()
                   && request.auth.uid == userId;
    }

    match /feedback/{id} {

      allow create: if signedIn();

      allow read: if isAdminUser()
                  || resource.data.uid == request.auth.uid;

      allow update: if isAdminUser()
                    || (
                      resource.data.uid == request.auth.uid
                      && request.resource.data
                        .diff(resource.data)
                        .affectedKeys()
                        .hasOnly(['replyRead'])
                    );

      allow delete: if isAdminUser();
    }
  }
}
```

このルールのポイント:

- `users/{uid}` ドキュメントの `isAdmin` フィールドは、**本人による作成時に `true` を仕込むことも、後から自分で書き換えることもできません**。Firebase Console から手動で設定する以外に管理者になる方法がない設計です。
- `feedback` コレクションは、本人または管理者のみ閲覧でき、本人は既読フラグ（`replyRead`）以外を書き換えられません。

### 3. 管理者ユーザーを設定する

このアプリの管理者判定は **コードにメールアドレスを書く方式ではなく、Firestore側のフラグで管理**します。

1. 管理者にしたいアカウントで一度アプリにログインする（`users/{uid}` ドキュメントが自動作成されます）
2. Firebase Console → Firestore Database → `users` コレクション → 該当ユーザーの `uid` のドキュメントを開く
3. フィールドを追加: `isAdmin`（boolean） = `true`

これだけで、そのアカウントは「ご意見箱」に届いた意見の閲覧・返信・削除ができるようになります。
コード上にメールアドレスや個人情報を一切残さずに管理者を設定できます。

### 4. ファイル構成・デプロイ

```
/                       ← login_index.html を index.html としてここに配置
/dash/index.html        ← ダッシュボード本体（このリポジトリの index.html）
/service-worker.js      ← （任意）PWAキャッシュ用
/manifest.json          ← PWAマニフェスト（任意で作成）
/image.png              ← アプリアイコン（任意で用意）
```

Cloudflare Pages / Netlify / Vercel などの静的ホスティングにそのままデプロイできます。

---

## 公開前チェックリスト

| 項目 | 状態 |
|---|---|
| `manifest.json` | 公開OK |
| `firebaseConfig`（apiKey等） | 公開OK（Firestoreルール必須） |
| Google Site Verification タグ | 削除済み |
| 管理者メールアドレスのコード直書き | 削除済み（Firestoreの`isAdmin`フラグ方式に変更） |
| `.env` | リポジトリに含めない（`.gitignore`で除外設定済み） |
| `serviceAccountKey.json` 等の秘密鍵 | リポジトリに含めない（`.gitignore`で除外設定済み） |
| Firestoreセキュリティルール | **必須・公開前に必ず設定** |

**最も重要なのはGitHubに上げること自体ではなく、Firestoreセキュリティルールが正しく設定されていることです。** 上記の手順2のルールを必ず適用してから運用してください。
