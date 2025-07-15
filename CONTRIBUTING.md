# 貢献ガイド

Multidimensional Visualizer プロジェクトへの貢献を歓迎します！このガイドでは、プロジェクトに効果的に貢献する方法について説明します。

## 貢献の種類

### 1. コード貢献
- 新機能の実装
- バグ修正
- パフォーマンス改善
- コードリファクタリング
- テストの追加・改善

### 2. ドキュメント貢献
- README やガイドの改善
- API ドキュメントの作成
- チュートリアルの作成
- 翻訳支援

### 3. 課題報告・提案
- バグ報告
- 機能要求
- ユーザビリティの改善提案
- デザイン提案

## 開発環境のセットアップ

### 前提条件
```bash
# Git の設定
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# 必要なツール（技術スタック決定後に更新）
# Node.js の場合:
node --version  # v18.0.0 以上
npm --version   # v8.0.0 以上

# Python の場合:
python --version  # 3.9 以上
pip --version
```

### リポジトリのフォークとクローン
```bash
# 1. GitHub でリポジトリをフォーク

# 2. フォークしたリポジトリをクローン
git clone https://github.com/YOUR_USERNAME/multidimensional-visualizer.git
cd multidimensional-visualizer

# 3. 上流リポジトリを追加
git remote add upstream https://github.com/muumuu8181/multidimensional-visualizer.git

# 4. リモートを確認
git remote -v
```

### 開発環境構築
```bash
# 依存関係のインストール（実装後に更新予定）
npm install  # または pip install -r requirements.txt

# 開発環境の起動
npm run dev  # または python -m uvicorn app:app --reload

# テストの実行
npm test    # または pytest
```

## 開発フロー

### 1. イシューの確認
- 既存のイシューを確認し、重複を避ける
- 大きな変更の場合は、事前にイシューで議論
- アサインされているイシューは他の人が作業中

### 2. ブランチの作成
```bash
# 最新の main ブランチに更新
git checkout main
git pull upstream main

# 機能ブランチを作成
git checkout -b feature/your-feature-name
# または
git checkout -b bugfix/issue-number-description
```

### ブランチ命名規則
- `feature/`: 新機能の追加
- `bugfix/`: バグ修正
- `docs/`: ドキュメント関連
- `refactor/`: リファクタリング
- `test/`: テスト関連
- `chore/`: その他のメンテナンス

### 3. コーディング
#### コーディング規約
```python
# Python の場合（PEP 8 準拠）
def calculate_pca(data: np.ndarray, 
                  n_components: int = 2,
                  random_state: Optional[int] = None) -> Tuple[np.ndarray, np.ndarray]:
    """
    主成分分析を実行する
    
    Args:
        data: 入力データ (n_samples, n_features)
        n_components: 出力する主成分数
        random_state: 乱数シード
        
    Returns:
        transformed_data: 変換後のデータ
        components: 主成分ベクトル
        
    Raises:
        ValueError: データの形状が無効な場合
    """
    if data.ndim != 2:
        raise ValueError("データは2次元配列である必要があります")
    
    # 実装...
    return transformed_data, components
```

```javascript
// JavaScript の場合
/**
 * データを正規化する
 * @param {number[][]} data - 入力データ
 * @param {string} method - 正規化手法 ('minmax', 'zscore')
 * @returns {number[][]} 正規化されたデータ
 */
function normalizeData(data, method = 'minmax') {
  if (!Array.isArray(data) || data.length === 0) {
    throw new Error('有効なデータ配列が必要です');
  }
  
  // 実装...
  return normalizedData;
}
```

#### テストの作成
```python
# test_pca.py
import pytest
import numpy as np
from multidim_viz.algorithms import calculate_pca

class TestPCA:
    def test_basic_pca(self):
        """基本的なPCAのテスト"""
        # テストデータの準備
        data = np.random.rand(100, 5)
        
        # PCAの実行
        transformed, components = calculate_pca(data, n_components=2)
        
        # 結果の検証
        assert transformed.shape == (100, 2)
        assert components.shape == (2, 5)
        
    def test_invalid_input(self):
        """無効な入力のテスト"""
        with pytest.raises(ValueError):
            calculate_pca(np.array([1, 2, 3]))  # 1次元配列
```

### 4. コミット
```bash
# ステージング
git add .

# コミットメッセージの規約
git commit -m "type(scope): description

詳細な説明（必要に応じて）

- 変更点1
- 変更点2

Fixes #123"
```

#### コミットメッセージの規約
- `feat(scope)`: 新機能の追加
- `fix(scope)`: バグ修正
- `docs(scope)`: ドキュメント関連
- `style(scope)`: コードスタイルの修正
- `refactor(scope)`: リファクタリング
- `test(scope)`: テスト関連
- `chore(scope)`: その他のメンテナンス

例:
```
feat(visualization): PCA可視化機能を追加

- 2D/3D散布図での表示に対応
- インタラクティブなズーム・回転機能
- カラーマッピングオプション追加

Closes #15
```

### 5. プルリクエスト
```bash
# ブランチをプッシュ
git push origin feature/your-feature-name

# GitHub でプルリクエストを作成
```

#### プルリクエストの内容
```markdown
## 概要
このPRでは、PCAによる次元削減機能を実装しました。

## 変更内容
- [ ] PCAアルゴリズムの実装
- [ ] 2D/3D可視化機能
- [ ] ユニットテストの追加
- [ ] ドキュメントの更新

## テスト方法
```bash
# テストの実行方法
npm test
# または
python -m pytest tests/test_pca.py
```

## スクリーンショット
（該当する場合）

## 関連イシュー
Closes #15

## チェックリスト
- [ ] コードは規約に従っている
- [ ] テストが追加されている
- [ ] ドキュメントが更新されている
- [ ] 既存のテストがパスしている
```

## コードレビュー

### レビュアーへの依頼
- 適切なレビュアーを指名
- 変更の背景と目的を明確に説明
- 特に注意してほしい点があれば記載

### レビューの観点
1. **機能性**: 要求通りに動作するか
2. **パフォーマンス**: 効率的な実装か
3. **保守性**: 理解しやすく修正しやすいか
4. **テスト**: 適切にテストされているか
5. **セキュリティ**: セキュリティ上の問題はないか

### レビューへの対応
```bash
# フィードバックに基づく修正
git add .
git commit -m "review: レビューフィードバックを反映

- 変数名を明確化
- エラーハンドリングを改善
- テストケースを追加"

git push origin feature/your-feature-name
```

## 継続的インテグレーション

### 自動チェック
- コードスタイルの検証
- ユニットテストの実行
- カバレッジの測定
- セキュリティスキャン

### ローカルでの事前チェック
```bash
# リンター実行
npm run lint     # または flake8, pylint
npm run format   # または black, prettier

# テスト実行
npm test         # または pytest

# 型チェック
npm run typecheck # または mypy
```

## リリースプロセス

### バージョニング
[Semantic Versioning](https://semver.org/) に従います:
- `MAJOR.MINOR.PATCH`
- `1.0.0`: 初回リリース
- `1.1.0`: 新機能追加
- `1.1.1`: バグ修正

### リリースノート
```markdown
# v1.1.0 (2025-08-01)

## 新機能
- PCA による次元削減機能 (#15)
- インタラクティブな3D可視化 (#23)

## 改善
- パフォーマンスを20%向上 (#28)
- UIの応答性改善 (#31)

## バグ修正
- データ読み込みエラーを修正 (#26)
- メモリリークを解消 (#29)

## 破壊的変更
- API の一部が変更されました（移行ガイド参照）
```

## コミュニティガイドライン

### 行動規範
- 敬意を持って接する
- 建設的なフィードバックを提供
- 初心者を歓迎し、サポートする
- 多様性を尊重する

### 議論の場
- **GitHub Issues**: バグ報告、機能要求
- **GitHub Discussions**: 一般的な質問、アイデア
- **Pull Requests**: コードレビュー、実装議論

### 質問・サポート
1. まず既存のドキュメントを確認
2. GitHub Discussions で検索
3. 新しい質問を投稿
4. 適切なラベルを使用

## よくある質問

### Q: 初めて貢献する場合、どこから始めれば良いですか？
A: `good first issue` ラベルの付いたイシューから始めることをお勧めします。

### Q: 機能要求はどこに投稿すれば良いですか？
A: GitHub Issues に `enhancement` ラベルを付けて投稿してください。

### Q: テストが失敗した場合はどうすれば良いですか？
A: ローカルで再現し、修正してから再度プッシュしてください。解決できない場合はコメントで質問してください。

### Q: コードレビューにはどの程度時間がかかりますか？
A: 通常1-3営業日以内にレビューを行います。緊急の場合はメンションしてください。

## リソース

### 参考資料
- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Python PEP 8](https://peps.python.org/pep-0008/)
- [JavaScript Standard Style](https://standardjs.com/)

### ツール
- **IDE**: VSCode, PyCharm, WebStorm
- **Git GUI**: GitHub Desktop, SourceTree
- **テスト**: Jest, pytest, Mocha
- **リンター**: ESLint, Pylint, Flake8

---

このガイドは継続的に更新されます。質問や改善提案があれば、お気軽にお知らせください。

最終更新日: 2025年7月15日