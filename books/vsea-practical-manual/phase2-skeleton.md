---
title: "第3章: フェーズ② 共通スケルトン作成"
---

# フェーズ②: 共通スケルトン作成

## 目的

各モジュールのインターフェースを定義し、全体の骨格を作る

## 成果物

1. 各モジュールのクラス定義
2. インターフェースメソッド(実装は空)
3. モジュール間の依存関係図

## モジュール分割例

```
Presentation Layer:
  - GUI (View)

Application Layer:
  - AppController (Controller)

Business Logic Layer:
  - DataReader
  - DataProcessor
  - DataUpdater

Infrastructure Layer:
  - FileSystem
  - Network
```

## インターフェース定義例

```python
# data_reader.py
from typing import Optional
import pandas as pd

class DataReader:
    """CSVファイルの読込と検証を担当"""

    def __init__(self):
        """初期化"""
        pass

    def read(self, filepath: str) -> Optional[pd.DataFrame]:
        """
        CSVファイル読込

        Args:
            filepath: 読込むCSVファイルのパス

        Returns:
            pandas DataFrame or None(エラー時)
        """
        pass

    def validate(self, df: pd.DataFrame) -> bool:
        """
        データ検証

        Args:
            df: 検証対象のDataFrame

        Returns:
            True: 検証成功, False: 検証失敗
        """
        pass
```

## チェックリスト

- [ ] 全モジュールのクラス定義完了
- [ ] 各メソッドにDocstring記載
- [ ] 型ヒント付与
- [ ] 依存関係図作成
- [ ] インターフェースレビュー完了

## 所要時間

**30分-1時間**
