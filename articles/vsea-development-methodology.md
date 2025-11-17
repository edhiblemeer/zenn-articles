---
title: "「完璧な設計」と「とりあえず動けば良い」の間を取る開発手法を考えた【VSEA】皆さんの意見ください"
emoji: "🏗️"
type: "idea"
topics: ["アーキテクチャ", "設計", "開発手法", "個人開発", "アジャイル"]
published: true
---

## 🤔 こんな経験ありませんか？

個人開発やスタートアップでよくある問題：

**パターンA: 完璧主義の罠**
```
Week 1-2: 「完璧な設計をしよう！」
Week 3-4: 「まだ設計中...」
Week 5:   「仕様変わった、設計やり直し...」
Week 6:   「いつ動くの？」😭
```

**パターンB: MVP地獄**
```
Week 1: 「とりあえず動けば良い！」→ 爆速実装
Week 2: 「機能追加しよ」→ コピペ量産
Week 3: 「バグ修正どこいじれば...」→ 迷子
Week 4: 「リファクタしたいけど怖い...」→ 技術的負債
```

どちらも経験ある方、多いのでは？

## 💡 「中間」を取る手法を考えました

**VSEA (Vertical Slice Evolutionary Architecture)** という開発手法を考えてみました。

ウォーターフォールとMVP型アジャイルの**良いとこ取り**を目指しています。

### コンセプト

> 「最小限の共通基盤（スケルトン）を先に作り、ユーザーストーリー単位の縦スライスで実装し、動かしながらアーキテクチャを進化させる」

図にするとこんな感じ：

```
ウォーターフォール          VSEA              MVP型アジャイル
                            ↓
完璧な設計 ────────→  仮設 + 進化  ←──────── とりあえず動く
(遅い・硬い)         (速い・柔軟)         (速い・崩壊)
```

## 🏗️ 4つのフェーズ

### Phase 1: UX整合設計（1〜2日）

**目的**: ストーリー間のUX矛盾を先に潰す

```markdown
やること:
- 各ユーザーストーリーを俯瞰
- 状態遷移・画面遷移を確認
- 「この画面、前提データないじゃん」を排除

やらないこと:
- 詳細設計
- DB設計
- 実装
```

**成果物**: UXマップ、システム境界定義

### Phase 2: 共通スケルトン（2〜3日）

**目的**: 最小限の共通基盤を「仮設」で作る

**重要**: これは**仮設構造**です。後で変える前提。

```typescript
// 例: AuthService（interfaceのみ）
interface AuthService {
  login(email: string, password: string): Promise<User>;
  logout(): Promise<void>;
  getUser(): Promise<User | null>;
}

// 実装はダミーでOK
class DummyAuthService implements AuthService {
  async login() { return { id: 1, name: "dummy" }; }
  async logout() { /* 何もしない */ }
  async getUser() { return null; }
}
```

**ポイント**:
- インターフェースだけ定義
- ロジックは書かない
- 「後で変える」前提で保守性より可変性優先

### Phase 3: 縦スライス実装（1ストーリー = 1〜3日）

**目的**: UI → DB まで1本通す

```
User Story: 「ユーザーは商品を検索できる」

実装順序:
1. UI 検索フォーム（ダミー入力でOK）
2. UseCase (SearchProductUseCase)
3. Repository (interface)
4. ダミーデータ返却
5. UI 結果表示

→ 動作確認 ✅

バリデーション？例外処理？最適化？
→ Phase 4でやる（今は動作優先）
```

**重要原則**: 「常に動く状態」を維持

### Phase 4: 肉付け（ストーリーごと）

**目的**: 動作状態を保ちながら品質を上げる

```markdown
優先順位:
1. 正常系の厳密化
2. バリデーション追加
3. 例外処理
4. パフォーマンス改善

重複コードの共通化:
- 2回目: まだ様子見
- 3回目: 共通化する ← 「3回ルール」
```

**「3回ルール」が重要**:
- 1回目: 偶然かもしれない
- 2回目: まだ分からない
- 3回目: パターン確定 → 共通化

## 🎯 実例で説明

### Before（MVP型）

```typescript
// UserController.ts
async login(email, password) {
  const user = await db.query("SELECT * FROM users WHERE email = ?", [email]);
  if (!user) throw new Error("Not found");
  // パスワードチェック
  // セッション作成
  // ...
}

// AdminController.ts（コピペ）
async login(email, password) {
  const user = await db.query("SELECT * FROM users WHERE email = ?", [email]);
  // ほぼ同じコード😭
}

// GuestController.ts（またコピペ）
async login(email, password) {
  // さらに同じコード😭😭😭
}
```

### After（VSEA）

```typescript
// Phase 2: スケルトン
interface AuthService {
  login(email: string, password: string): Promise<User>;
}

// Phase 3: 縦スライス（1回目）
class UserController {
  constructor(private auth: AuthService) {}
  async login(email, password) {
    return this.auth.login(email, password); // interfaceに依存
  }
}

// Phase 3: 縦スライス（2回目）
class AdminController {
  constructor(private auth: AuthService) {}
  async login(email, password) {
    return this.auth.login(email, password); // 同じ
  }
}

// Phase 4: 3回目で共通化（3回ルール発動）
class AuthServiceImpl implements AuthService {
  async login(email, password) {
    // ここに1回だけ実装
    const user = await db.query("SELECT * FROM users WHERE email = ?", [email]);
    // ...
  }
}
```

**結果**:
- ✅ 重複なし
- ✅ テスト容易
- ✅ 変更が楽

## 🤖 Claude Code等のAIペアプロと相性抜群

個人開発でVSEA使うと、**AIが「Platformオーナー」になる**。

### AIとの役割分担

```yaml
Phase 1: UX整合設計
  人間: ストーリーリスト作成（30分）
  AI: UX矛盾検出（30分）
  人間: 最終判断（30分）
  → 1.5時間で完了

Phase 2: スケルトン
  人間: 必要な共通レイヤリスト（15分）
  AI: interface定義生成（40分）
  人間: レビュー（15分）
  → 1時間で完了

Phase 3: 縦スライス
  人間: 「User Story 1: 商品検索」（5分）
  AI: UI→DB縦スライス生成（45分）
  人間: 動作確認（10分）
  → 1時間で1スライス完了

Phase 4: 肉付け
  人間: 「3スライス完了、リファクタして」
  AI: 重複検出→共通化提案（30分）
  人間: 承認→実行（15分）
  → 45分で完了
```

**従来**: 1〜2ヶ月かかってたプロジェクトが **10日で完成**。

## 📊 メリット・デメリット

### ✅ メリット

1. **速い**: スケルトンから動作まで短い
2. **安定**: 常時動作で構造崩壊リスク低い
3. **柔軟**: 仕様変更がストーリー単位で容易
4. **拡張性**: 実働から共通化→不要な抽象がない
5. **AI相性**: フェーズ明確→AIへの指示が簡単

### ⚠️ デメリット・課題

1. **学習コスト**: 「仮設」概念の理解が必要
2. **規律必要**: 「3回ルール」を守る習慣化
3. **縦スライス粒度**: どこまで分割すべきか判断が難しい
4. **Phase 1コスト**: UX整合に時間かけすぎると本末転倒

## 🙋 皆さんの意見を聞きたいです

### 質問1: この手法、使えそう？

- ✅ 使えそう / 試してみたい
- ⚠️ 一部は良いけど懸念あり
- ❌ 実用的じゃない

### 質問2: どこが懸念？

特に以下の点、ご意見ください：

1. **「仮設」概念**: 共通基盤を「後で変える前提」で作るのは不安？
2. **「3回ルール」**: 2回目で共通化したくなりませんか？
3. **Phase 1のコスト**: UX整合設計に何日かけるべき？
4. **縦スライス粒度**: 「1ストーリー = 1スライス」は自明？

### 質問3: 他の手法との比較

以下の手法を使ったことある方、比較意見お願いします：

- **Scrum / Agile**: VSEAとの違いは？
- **Vertical Slice Architecture**: VSEAとどう違う？
- **Shape Up**: 似てる？違う？
- **ドメイン駆動設計（DDD）**: 相性は？

### 質問4: 実際に試してみた方

もし試してみた方がいたら、以下教えてください：

- どんなプロジェクトで使った？
- どのPhaseが一番難しかった？
- 工数削減できた？
- 失敗した点は？

## 💬 コメント歓迎！

以下のような意見、大歓迎です：

- 🐛 「ここが間違ってる」
- 💡 「こうした方が良い」
- 🤔 「こんな場合どうする？」
- 📝 「似た手法でこれがある」
- 🚀 「試してみた結果」

特に知りたいこと：

1. **個人開発での適用例**
2. **チーム開発での課題**
3. **AI時代の開発手法として妥当か**
4. **既存手法との組み合わせ方**

## 📚 参考資料

VSEAは以下の手法から影響を受けています：

- **Vertical Slice Architecture**: Jimmy Bogard氏
- **Evolutionary Architecture**: Neal Ford氏
- **Shape Up**: Basecamp社
- **Scrum / Agile**: 一般的なアジャイル手法

## 🎓 まとめ

**VSEA（Vertical Slice Evolutionary Architecture）** のポイント：

1. ✅ **Phase 1**: UX整合を先に確認（1〜2日）
2. ✅ **Phase 2**: 共通スケルトンを「仮設」で作る（2〜3日）
3. ✅ **Phase 3**: 縦スライスで動作優先実装（1ストーリー = 1〜3日）
4. ✅ **Phase 4**: 動作状態を保ちながら肉付け（継続的）

**キー原則**:
- 未来予測より実働検証
- 常に縦スライスで動かす
- 共通レイヤは仮設
- 抽象化は3回使われてから
- 常時統合

**AI時代の開発手法として**:
- Claude Code等のAIペアプロと相性抜群
- 個人開発でも高品質開発が可能
- 工数80〜95%削減の可能性

---

**皆さんの意見、お待ちしています！**

特に以下の観点でコメントいただけると嬉しいです：

- 💭 この手法、実務で使えそう？
- 🤔 どこに懸念を感じる？
- 📊 他の手法と比較してどう？
- 🚀 試してみた感想は？

**連絡先**:
- Zennコメント欄
- Twitter: [@ediblemeer](https://x.com/ediblemeer)

**2025年11月17日 公開**
