# 設計書

## 概要

QuizBiteは、Next.js（フロントエンド）とGo（バックエンド）を使用したモダンなWebアプリケーションです。ユーザーは手軽に5問のクイズに挑戦し、進捗を追跡してランクアップを目指すことができます。管理者はAIを活用してコンテンツを効率的に作成・管理できます。

## アーキテクチャ

### システム構成

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   フロントエンド   │    │   バックエンド    │    │   外部サービス    │
│   (Next.js)     │◄──►│     (Go)        │◄──►│  AWS Bedrock    │
│                 │    │                 │    │   (AI生成)      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                              │
                              ▼
                       ┌─────────────────┐
                       │   データベース    │
                       │    (MySQL)      │
                       └─────────────────┘
```

### 技術スタック

- **フロントエンド**: Next.js 14 + TypeScript + TailwindCSS
- **バックエンド**: Go 1.21 + Gin Framework
- **データベース**: MySQL 8.0
- **AI生成**: AWS Bedrock (Claude)
- **開発環境**: Docker Compose
- **認証**: セッションベース認証（管理者用）

## コンポーネントと インターフェース

### フロントエンドコンポーネント

#### ページコンポーネント
- `HomePage`: メインページ（今日のお題、テーマ一覧）
- `QuizPage`: クイズ実行ページ
- `ResultPage`: 結果表示ページ
- `AdminPage`: 管理者ページ

#### 共通コンポーネント
- `QuizCard`: クイズテーマ表示カード
- `QuestionCard`: 問題表示カード
- `ProgressBar`: 進捗表示バー
- `RankBadge`: ランク表示バッジ
- `ShareButton`: シェアボタン

### バックエンドAPI

#### ユーザー向けAPI
```
GET /api/themes              # テーマ一覧取得
GET /api/themes/featured     # 今日のお題取得
GET /api/quiz/:theme_id      # クイズセット取得
POST /api/quiz/submit        # 回答送信
GET /api/user/stats          # ユーザー統計取得
```

#### 管理者向けAPI
```
POST /api/admin/login        # 管理者ログイン
GET /api/admin/themes        # テーマ管理一覧
POST /api/admin/themes       # テーマ作成
PUT /api/admin/themes/:id    # テーマ更新
DELETE /api/admin/themes/:id # テーマ削除
POST /api/admin/generate     # AI問題生成
POST /api/admin/featured     # 注目テーマ設定
```

## データモデル

### データベーススキーマ

#### themes テーブル
```sql
CREATE TABLE themes (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);
```

#### quiz_sets テーブル
```sql
CREATE TABLE quiz_sets (
    id INT PRIMARY KEY AUTO_INCREMENT,
    theme_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (theme_id) REFERENCES themes(id) ON DELETE CASCADE,
    INDEX idx_theme_id (theme_id)
);
```

#### questions テーブル
```sql
CREATE TABLE questions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    quiz_set_id INT NOT NULL,
    question_text TEXT NOT NULL,
    option_a VARCHAR(200) NOT NULL,
    option_b VARCHAR(200) NOT NULL,
    option_c VARCHAR(200) NOT NULL,
    option_d VARCHAR(200) NOT NULL,
    correct_answer ENUM('A', 'B', 'C', 'D') NOT NULL,
    explanation TEXT,
    question_order INT NOT NULL,
    FOREIGN KEY (quiz_set_id) REFERENCES quiz_sets(id) ON DELETE CASCADE,
    INDEX idx_quiz_set_id (quiz_set_id)
);
```

#### user_sessions テーブル
```sql
CREATE TABLE user_sessions (
    id VARCHAR(36) PRIMARY KEY,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_activity TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

#### attempts テーブル
```sql
CREATE TABLE attempts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    session_id VARCHAR(36) NOT NULL,
    question_id INT NOT NULL,
    selected_answer ENUM('A', 'B', 'C', 'D') NOT NULL,
    is_correct BOOLEAN NOT NULL,
    answered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES user_sessions(id) ON DELETE CASCADE,
    FOREIGN KEY (question_id) REFERENCES questions(id) ON DELETE CASCADE,
    UNIQUE KEY unique_session_question (session_id, question_id),
    INDEX idx_session_id (session_id),
    INDEX idx_question_id (question_id)
);
```

#### daily_featured テーブル
```sql
CREATE TABLE daily_featured (
    id INT PRIMARY KEY AUTO_INCREMENT,
    theme_id INT NOT NULL,
    featured_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (theme_id) REFERENCES themes(id) ON DELETE CASCADE,
    UNIQUE KEY unique_date (featured_date),
    INDEX idx_featured_date (featured_date)
);
```

#### admin_users テーブル
```sql
CREATE TABLE admin_users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### データ関係図

```mermaid
erDiagram
    themes ||--o{ quiz_sets : has
    quiz_sets ||--o{ questions : contains
    questions ||--o{ attempts : answered
    user_sessions ||--o{ attempts : makes
    themes ||--o{ daily_featured : featured_as
    
    themes {
        int id PK
        string title
        text description
        timestamp created_at
        timestamp updated_at
        boolean is_active
    }
    
    quiz_sets {
        int id PK
        int theme_id FK
        timestamp created_at
    }
    
    questions {
        int id PK
        int quiz_set_id FK
        text question_text
        string option_a
        string option_b
        string option_c
        string option_d
        enum correct_answer
        text explanation
        int question_order
    }
    
    user_sessions {
        string id PK
        timestamp created_at
        timestamp last_activity
    }
    
    attempts {
        int id PK
        string session_id FK
        int question_id FK
        enum selected_answer
        boolean is_correct
        timestamp answered_at
    }
    
    daily_featured {
        int id PK
        int theme_id FK
        date featured_date
        timestamp created_at
    }
```

## エラーハンドリング

### フロントエンドエラーハンドリング

#### エラー種別
- **ネットワークエラー**: 接続失敗、タイムアウト
- **APIエラー**: 4xx、5xx レスポンス
- **バリデーションエラー**: 入力値検証失敗

#### エラー表示戦略
```typescript
interface ErrorState {
  type: 'network' | 'api' | 'validation';
  message: string;
  retryable: boolean;
}

// エラーメッセージの日本語化
const ERROR_MESSAGES = {
  NETWORK_ERROR: 'インターネット接続を確認してください',
  SERVER_ERROR: 'サーバーエラーが発生しました。しばらく待ってから再試行してください',
  NOT_FOUND: 'お探しのコンテンツが見つかりません',
  VALIDATION_ERROR: '入力内容を確認してください'
};
```

### バックエンドエラーハンドリング

#### エラーレスポンス形式
```go
type ErrorResponse struct {
    Error   string `json:"error"`
    Code    string `json:"code"`
    Message string `json:"message"`
}
```

#### エラー種別とHTTPステータス
- **400 Bad Request**: バリデーションエラー
- **401 Unauthorized**: 認証エラー（管理者）
- **404 Not Found**: リソース未発見
- **429 Too Many Requests**: レート制限
- **500 Internal Server Error**: サーバー内部エラー

## テスト戦略

### フロントエンドテスト

#### 単体テスト (Jest + Testing Library)
- コンポーネントの描画テスト
- ユーザーインタラクションテスト
- カスタムフックのテスト

#### 統合テスト
- APIとの連携テスト
- ページ遷移テスト
- エラーハンドリングテスト

### バックエンドテスト

#### 単体テスト
```go
// テスト例
func TestQuizService_GetQuizSet(t *testing.T) {
    // モックデータベース設定
    // テストケース実行
    // アサーション
}
```

#### 統合テスト
- データベース連携テスト
- AI生成API連携テスト
- エンドツーエンドAPIテスト

### テストデータ管理

#### シードデータ
```sql
-- テスト用テーマデータ
INSERT INTO themes (title, description) VALUES 
('ラッコの生態', 'ラッコに関する豆知識'),
('日本の歴史', '日本史の興味深いエピソード');

-- テスト用問題セット
INSERT INTO quiz_sets (theme_id) VALUES (1), (2);
```

#### テスト環境設定
- Docker Compose でのテスト用DB構築
- テストデータの自動投入
- テスト実行後のクリーンアップ

### パフォーマンステスト

#### フロントエンド
- Lighthouse による性能測定
- Core Web Vitals の監視
- バンドルサイズの最適化

#### バックエンド
- API レスポンス時間測定
- データベースクエリ最適化
- 同時接続数テスト

## セキュリティ考慮事項

### 認証・認可
- 管理者のみセッションベース認証
- CSRF トークンによる保護
- パスワードのハッシュ化（bcrypt）

### データ保護
- SQL インジェクション対策（プリペアドステートメント）
- XSS 対策（入力値サニタイズ）
- レート制限による DoS 攻撃対策

### プライバシー
- ユーザー個人情報の最小化
- セッションIDのみでユーザー識別
- データ保持期間の制限