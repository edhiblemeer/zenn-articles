---
title: "第7章: 実践で遭遇した課題と解決策"
---

# 実践で遭遇した課題と解決策

## 課題1: ログイン処理のタイミング問題

### 症状
Playwrightでフォーム入力が早すぎてログイン失敗

### 解決策
```python
page.goto(login_url)
page.wait_for_load_state("networkidle")  # ページ完全読込待機
page.wait_for_selector('input[name="login_id"]', timeout=10000)
page.fill('input[name="login_id"]', self.login_id)
page.wait_for_timeout(500)  # 入力確定待機
```

### 教訓
- ブラウザ自動化では適切な待機処理が必須
- `wait_for_load_state()`や`wait_for_selector()`を活用

## 課題2: 日付フォーマットの不一致

### 症状
CSV日付が`YYYY/MM/DD`だが、コードは`YYYY-MM-DD`を期待

### 解決策
```python
def parse_date(date_str: str) -> datetime:
    """複数フォーマットに対応した日付パース"""
    if '/' in date_str:
        return datetime.strptime(date_str, '%Y/%m/%d')
    elif '-' in date_str:
        return datetime.strptime(date_str, '%Y-%m-%d')
```

### 教訓
- 外部データは複数フォーマットを想定
- 入力検証で早期エラー検出

## 課題3: 深夜帯時間の扱い

### 症状
20:00-1:30のような翌日にまたがる時間の扱いでエラー

### 解決策
```python
# 退勤時間が出勤時間より小さい場合は翌日扱い(+24時間)
if leave_hour < going_hour:
    leave_hour += 24
```

### 教訓
- ドメイン固有のビジネスルールを明確化
- 境界値に注意
