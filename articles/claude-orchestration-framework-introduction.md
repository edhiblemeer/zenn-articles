---
title: "個人開発165時間の記録：Claude Codeで大規模開発が破綻して、オーケストレーションフレームワークを作った話"
emoji: "🏗️"
type: "tech"
topics: ["claudecode", "python", "nodejs", "個人開発", "開発効率化"]
published: true
---

## 💥 開発が破綻した日

2025年9月、投資分析EA（Expert Advisor）の開発プロジェクトを始めた。

Claude Codeを使った大規模開発。タスクが20個を超えたあたりで、完全に破綻した。

```bash
TASK-BA-001: PTYラッパー実装
TASK-BA-002: セッション管理システム
TASK-BA-003: Webhook統合
TASK-BA-004: メッセンジャー統合
...
TASK-DO-003: CI/CD自動化
TASK-QA-003: E2Eテスト自動化
```

**発生した問題**:
- ❌ どのタスクが完了したか分からない
- ❌ 品質チェックを忘れてバグ混入
- ❌ 並列開発でGit競合地獄
- ❌ Claude Codeのセッションが切れてコンテキスト消失
- ❌ 進捗管理が完全にカオス

**結論**: 「このままじゃプロジェクトが死ぬ...」

## 💡 解決策を作ることにした

外部ツールを探したが、Claude Code専用のオーケストレーションツールは存在しない。

→ **自分で作るしかない**

こうして、**Claude Orchestration Framework v3.0** の開発が始まった。

### プロジェクト目標

「Claude Codeで複雑なソフトウェア開発プロジェクトを効率的に管理する」

**ターゲット**:
- 個人開発者（複数プロジェクト並行）
- スタートアップ（少人数で高速開発）
- AI活用開発（Claude Code/Cursor利用者）

---

## 🎬 実際の動作

### Dashboard（localhost:8765）

リアルタイム監視ダッシュボードで、全タスクの進捗を可視化：

![Dashboard全体画面](https://raw.githubusercontent.com/edhiblemeer/zenn-articles/main/images/dashboard-overview.png)
*Claude Orchestration Dashboard - PM Message対話とタスク管理*

**表示内容**:
- 📊 Current Wave: 1（実行中のWave番号）
- ✅ Completed Tasks: 8（完了タスク数）
- 🔄 Pending Tasks: 7（待機中タスク数）
- 👥 Active Sessions: 5（アクティブセッション数）
- 💬 PM Messages: AI対話による品質改善提案

**PM Message機能（Phase 2.6i新機能）**:

スクリーンショットの通り、品質スコア95.74%（Excellent）のタスクでも、
AIが**さらなる改善提案**を対話形式で提示：

```
質問: PTYラッパーのユニットテストが品質スコア95%以上で完了しています。
      完了として確定しますか？

Option A: 95.74%カバレッジで完了確定 ⭐ PM推奨
  メリット:
  - 高カバレッジ達成済み（95%以上目標達成）
  - 早期完了でプロジェクトタイムライン遵守

  デメリット:
  - レアケース・エッジケースのテストが不完全な可能性

Option B: プロパティベーステストで99%+カバレッジ達成
  メリット:
  - ランダム入力生成により予期せぬエッジケース発見

  デメリット:
  - QAタイムライン圧迫（プロパティテスト実装に追加時間）
```

この対話システムにより、**品質と納期のトレードオフを明示的に判断**できます。

---

### タスク管理ビュー

![タスク一覧画面](https://raw.githubusercontent.com/edhiblemeer/zenn-articles/main/images/tmux-parallel-execution.png)
*タスク一覧と進捗状況 - Backend/DevOps並列実行*

**タスクステータス**:
- 🟢 **completed**: PTYラッパー実装、セッション管理、スラッシュコマンド実装など（8件完了）
- 🟡 **pending**: LINE/Slack/Discord Webhook統合（3件待機中）
- 🔵 **in progress**: Cloudflareトンネル自動セットアップ（実行中）

**カテゴリ別管理**:
- `backend`: PTY、セッション管理、Webhook（TASK-BA-001〜008）
- `devops`: インフラ自動化、CI/CD（TASK-DO-001〜002）
- `qa`: テスト自動化（別Wave）

---

## 🏗️ 設計思想：3層セッション管理アーキテクチャ

### Layer 1: Git Worktree（物理分離）

**課題**: Epic並列開発でGit競合が頻発

**解決策**: Epic別にWorktreeを物理分離

```bash
# 従来の問題
main/
├── backend/   # 編集中にコンフリクト！
├── frontend/  # 編集中にコンフリクト！
└── devops/    # 編集中にコンフリクト！

# Worktree分離後
.worktrees/
├── epic-backend/   → Backend開発専用
├── epic-frontend/  → Frontend開発専用
└── epic-devops/    → DevOps開発専用

→ 競合ゼロ！
```

**メリット**:
- ✅ Epic間の完全分離
- ✅ 並列開発の安全性
- ✅ ブランチ切り替え不要

### Layer 2: Session Pool（論理管理）

**課題**: Claude Codeセッションが30分で切れる

**解決策**: UUID-based Session管理

```python
# セッション作成
session_ba_001 = SessionManager.create(
    task_id="TASK-BA-001",
    timeout=30,  # 30分で自動クリーンアップ
    metadata={
        "epic": "backend",
        "assignee": "claude-sonnet-4"
    }
)

# セッション再開
SessionManager.resume("session_ba_001")
```

**機能**:
- UUID識別（セキュア）
- 30分タイムアウト（メモリ効率化）
- tmux統合（セッション永続化）
- 自動クリーンアップ

スクリーンショットの**Active Sessions: 5**は、
現在5つのtmuxセッションが並列実行中であることを示しています。

### Layer 3: Quality Context（評価）

**課題**: 「品質が良い」の定義が曖昧

**解決策**: 8次元品質スコア（0.0-1.0）

| Step | 検証項目 | 合格基準 | ツール |
|------|---------|---------|--------|
| 1 | Syntax | エラー0 | Python/TypeScript parser |
| 2 | Type | 型エラー0 | mypy/tsc |
| 3 | Lint | Critical 0 | pylint/eslint |
| 4 | Security | 脆弱性0（High以上） | bandit/npm audit |
| 5 | Test | unit ≥80% | pytest/jest |
| 6 | Performance | API<200ms | pytest-benchmark |
| 7 | Documentation | 完全性≥80% | docstring coverage |
| 8 | Integration | 主要パス100%合格 | E2E tests |

**総合品質スコア**: ≥0.8（合格基準）

スクリーンショットのTASK-QA-001では**95.74% (Excellent)**を達成。

```python
# 実際の評価例
{
    "task_id": "TASK-QA-001",
    "quality_score": 0.9574,
    "status": "Excellent",
    "test_coverage": 95.74,
    "breakdown": {
        "correctness": 0.95,
        "maintainability": 0.90,
        "performance": 0.85,
        "security": 1.0,
        "test_quality": 0.95,
        "documentation": 0.85,
        "code_style": 0.95,
        "integration": 0.95
    }
}
```

---

## 🤖 v3.0の革命: AI-Driven Dynamic Analysis

### 従来の問題（v2.0以前）

**静的ルールベース**:
```python
# 固定されたEpic分解ルール
EPIC_RULES = {
    "backend": ["database", "api", "auth"],
    "frontend": ["ui", "components", "routing"],
    "devops": ["ci", "cd", "monitoring"]
}
```

**問題点**:
- ❌ プロジェクトごとに最適なEpic戦略が異なる
- ❌ ルールが硬直的
- ❌ 新しいプロジェクトタイプに対応できない

### v3.0の解決策: Claude Code AI First

**動的AI判定**:
```bash
# AIが最適なEpic戦略を自動選択
/pm:epic-breakdown your-project.md --strategy auto
```

**5つのEpic戦略**:

| 戦略 | 適用例 | Epic構成 |
|-----|-------|---------|
| `layer` | トレーディングボット | Layer 0 (LLM) / Layer 1 (Trading) / Layer 2 (GPU) |
| `phase` | MVP開発 | Phase 1 (Prototype) / Phase 2 (Beta) / Phase 3 (Production) |
| `function` | SaaS | Core / Learning / Distribution / Monitoring |
| `generic` | 汎用Web App | Frontend / Backend / DevOps / QA |
| `auto` | **AIが自動判定** | PRD内容から最適戦略を選択 |

### AI Engine実装

```python
# core/ai_engine.py
class AIEngine:
    def analyze_prd(self, prd_content: str) -> EpicStrategy:
        """PRD内容を分析して最適なEpic戦略を判定"""
        prompt = f"""
        Analyze this PRD and determine the best Epic breakdown strategy:

        {prd_content}

        Return JSON:
        {{
          "strategy": "layer|phase|function|generic",
          "reasoning": "...",
          "epics": [
            {{"name": "...", "description": "...", "tasks": [...]}}
          ]
        }}
        """

        response = claude_code_api.chat(prompt)
        return self.validate_schema(response)
```

---

## 😅 開発で苦戦したポイント

### 1. AI判定ログ200件事件（Phase 2.6i）

Phase 2.6i「エスカレーションライフサイクル」実装中、異常事態発生。

```bash
$ ls .pm/logs/ai_decisions/ | wc -l
200

$ ls .pm/logs/ai_decisions/ | head -5
failure_TASK-DO-001_1on1_permission_prompt_20251106_233908.json
failure_TASK-DO-001_1on1_permission_prompt_20251106_235553.json
failure_TASK-DO-001_1on1_permission_prompt_20251106_235633.json
...
```

**何が起きた？**

`TASK-DO-001`（DevOps CI/CD設定）で、
**permission_prompt失敗ログが200件連続生成**。

**原因**:
- エスカレーション判定ロジックのバグ
- AI判定が無限ループ
- 1分ごとにログ生成（200分 = 3.3時間稼働）

**解決策**:
```python
# core/escalation_manager.py（修正後）
def should_escalate(self, task_id: str) -> bool:
    """エスカレーション判定（無限ループ防止）"""
    history = self.get_escalation_history(task_id)

    # 同一タスクで5回以上失敗 → エスカレーション停止
    if len(history) >= 5:
        logger.warning(f"Escalation limit reached for {task_id}")
        return False

    return self.ai_engine.judge_escalation(task_id)
```

**学び**:
- ✅ AI判定には必ずリミット設定
- ✅ ログ監視の自動化
- ✅ 異常検知アラート

### 2. Task tool並列実行の罠

**従来の逐次実行**: **110分**

```bash
# 逐次実行（遅い）
for task in TASK-BA-001 TASK-BA-002 TASK-BA-003; do
  claude execute $task
done
```

**並列実行**: **37分**（66%削減）

```bash
# 並列実行（速い）
claude execute TASK-BA-001 &
claude execute TASK-BA-002 &
claude execute TASK-BA-003 &
wait
```

**でも、問題発生**:
- tmuxセッション管理が複雑
- ログファイルの競合
- リソース枯渇

**解決策**: `core/subagent_launcher.sh`

```bash
#!/bin/bash
# SubAgent自動起動スクリプト

function launch_subagent() {
    local task_id=$1
    local session_name="pm-${task_id}"

    # tmuxセッション作成
    tmux new-session -d -s "$session_name"

    # Claude Code実行
    tmux send-keys -t "$session_name" \
        "claude -p --dangerously-skip-permissions @.pm/execution/task-${task_id}-instruction.md" \
        C-m

    # ログ出力
    tmux pipe-pane -t "$session_name" \
        "cat > .pm/logs/${task_id}_execution.log"
}

# 並列起動（最大3並列）
for task in TASK-BA-001 TASK-BA-002 TASK-BA-003; do
    launch_subagent "$task"
done
```

スクリーンショットの**Active Sessions: 5**は、
この仕組みで5つのタスクが並列実行中であることを示しています。

**最適化結果**:
- ✅ tmux自動管理
- ✅ ログ分離
- ✅ リソース制御（最大3並列）

---

## 📈 開発タイムライン（透明性重視）

個人開発165時間の記録を公開します。

### Phase 1-12（基盤実装）: **165時間**

| Phase | 内容 | 時間 | 完了日 |
|-------|-----|------|--------|
| Phase 1 | 要件定義・設計 | 12時間 | 2025-09-15 |
| Phase 2 | Git Worktree統合 | 16時間 | 2025-09-22 |
| Phase 3 | Session Pool実装 | 20時間 | 2025-09-29 |
| Phase 4 | Quality評価システム | 24時間 | 2025-10-06 |
| Phase 5-12 | Dashboard/統合テスト | 93時間 | 2025-10-20 |

**合計**: **165時間**（約1ヶ月、1日平均5.5時間）

### v3.0 Week 1-10（AI統合）: 完了

| Week | 機能 | 成果 |
|------|-----|------|
| Week 1-3 | Epic/Task自動分解 | AI判定システム |
| Week 4 | Agent Composition | 動的エージェント生成 |
| Week 5 | PRD Questions | 対話型PRD作成 |
| Week 6 | Dependency分析 | タスク依存関係自動検出 |
| Week 7 | Quality Criteria | カスタム品質基準 |
| Week 8-9 | Integration Testing | auth-systemサンプル完走 |
| Week 10 | Documentation | 主要ドキュメント3件更新 |

### Phase 2（品質改善）: **完了**

**Phase 2 Improvements**（2025-10-28）:
- ✅ タスク指示書自動生成（11タスク、100%成功率）
- ✅ Task tool並列実行（従来110分 → 37分、**66%削減**）
- ✅ 平均品質スコア**0.86**（目標≥0.8達成）
- ✅ 平均文字数19,536文字（詳細な指示書）

**Phase 2.6i Escalation**（2025-11-11最新）:
- ✅ エスカレーションライフサイクル管理
- ✅ PM Message対話システム（スクリーンショット参照）
- 🔥 AI判定システム（200件ログ事件で苦戦）

---

## 🚀 開発成果

### Before（通常のClaude Code開発）

- ❌ タスク管理: 手動 → 混乱
- ❌ 品質評価: 不明 → 不安
- ❌ 並列実行: なし → 遅い
- ❌ セッション: 30分で消失 → 再開困難

### After（Claude Orchestration Framework使用）

- ✅ タスク分解: 自動生成 → **96%時間削減**（2時間→5分）
- ✅ 品質評価: 8次元スコア → **安心**（0.86/1.0）
- ✅ 並列実行: 自動管理 → **66%効率化**（110分→37分）
- ✅ セッション: 永続化 → **いつでも再開可能**

### 実績数値まとめ

| 項目 | Before | After | 改善率 |
|-----|--------|-------|--------|
| Epic/Task分解 | 手動2時間 | 自動5分 | **96%削減** |
| 並列実行時間 | 110分 | 37分 | **66%削減** |
| 品質可視化 | なし | 0.86/1.0 | **100%可視化** |
| セッション管理 | 手動 | 自動 | **100%自動化** |
| Active Sessions | 1個 | 5個並列 | **5倍並列化** |

スクリーンショットの通り、**8件完了 + 7件待機 + 5セッション並列実行**という
大規模プロジェクト管理が実現しています。

---

## 🛠️ 技術スタック

### Backend（セッション管理・品質評価）

- **Python 3.11+**
  - `core/ai_engine.py` - AI判定エンジン
  - `core/session_manager.py` - セッション管理
  - `core/quality_evaluator.py` - 品質評価
  - `core/escalation_manager.py` - エスカレーション管理

### Frontend（監視Dashboard）

- **Node.js 18+ / Express**
  - `monitoring/dashboard_server.js` - Webサーバー
  - WebSocket（リアルタイム通信）
  - localhost:8765でアクセス

### Infrastructure

- **tmux** - セッション永続化（Active Sessions: 5）
- **Git Worktree** - Epic物理分離
- **JSON-RPC 2.0** - AI通信プロトコル

### 統合フレームワーク

以下の3つのOSSフレームワークを統合：

1. **CCPM** (automazeio/ccpm) - Git Worktree並列開発
2. **ccswarm** (nwiizo/ccswarm) - JSON-RPC通信、品質評価
3. **claude-mpm** (bobmatnyc/claude-mpm) - Session Pool管理

---

## 🗺️ 次の展開

### Phase 2.7（検討中）

**Remote CLI Controller統合**:
- このフレームワークを実戦投入するための実験プロジェクト
- LINE/Slack/Discordからの遠隔操作
- 外出先でのタスク確認

現在、このフレームワーク自体を使って開発中です（メタ開発）。

### v4.0（構想）

- チーム開発対応（複数開発者の協調）
- クラウド統合（AWS/GCP）
- AI自動応答（SlackBot化）

---

## 📚 ドキュメント

**完全なドキュメント**: 2,434行 + 3,217行

- `docs/USER_GUIDE.md` - ユーザーガイド（2,434行）
- `docs/DEVELOPER_GUIDE.md` - 開発者ガイド（3,217行）
- `docs/pm_automation/` - 詳細ドキュメント28ファイル

---

## 🙏 オープンソース化について

### 現在の状況

- ✅ v3.0開発完了（2025-10-30）
- ✅ Phase 2.6i統合完了（2025-11-11）
- 🚧 GitHub公開準備中

### 公開予定内容

- フレームワーク本体（MIT License予定）
- 完全なドキュメント
- サンプルプロジェクト
- セットアップガイド

### 応援してください！

個人開発は孤独です。以下の形で応援いただけると励みになります：

- 💬 「こんな機能欲しい！」のコメント
- 🐛 バグ報告・改善提案
- ⭐ GitHub Star（公開時）
- 📣 SNSでのシェア

**連絡先**:
- Zennコメント欄
- Twitter: [@ediblemeer](https://x.com/ediblemeer)
- GitHub: 公開時に追記

---

## 🎓 まとめ

### 個人開発165時間で得たもの

**技術的成果**:
- ✅ 3層セッション管理アーキテクチャ
- ✅ AI-Driven Dynamic Analysis
- ✅ 8次元品質評価システム
- ✅ 66%の開発効率化
- ✅ 5セッション並列実行

**個人的成長**:
- Python/Node.js/tmux/Git Worktreeの深い理解
- AI活用開発のノウハウ
- 大規模個人開発のプロジェクト管理能力

**コミュニティへの貢献**:
- Claude Code開発者向けツール
- オープンソース化で知見共有（予定）

### 次のステップ

1. **今週**: Phase 2.6i完全統合
2. **来週**: Remote CLI Controller実験
3. **1ヶ月後**: GitHub公開

Claude Codeで大規模開発に挑戦する全ての開発者へ。

**Let's orchestrate together! 🚀**

---

**2025年11月17日 公開**
