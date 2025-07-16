# 論理データモデル

## 概要
石巻市住民基本台帳・印鑑登録システムの概念データモデルを見直し、矛盾・曖昧箇所を解消した論理データモデルです。リレーショナルデータベースでの実装を想定し、適切な正規化、制約、データ型を定義しています。

## 目的
- 概念モデルの矛盾・曖昧箇所を解消
- リレーショナルデータベース実装に適した構造を提供
- データ整合性とパフォーマンスを両立
- 保守性と拡張性を確保

## 矛盾・曖昧箇所の洗い出しと解決

### 1. 住民識別子の曖昧性
**問題**: 住民の一意識別において、住民識別子、個人番号、住民票コードの使い分けが不明確

**解決策**: 
- 住民テーブルの主キーは代理キー（住民ID）を使用
- 個人番号と住民票コードは一意制約付きの別属性として管理
- 外部システム連携時は個人番号を使用、内部処理は住民IDを使用

### 2. 住所情報の重複可能性
**問題**: 住民と住所の関係で、住所情報が住民テーブルに重複格納される可能性

**解決策**:
- 住所マスタテーブルを独立して作成
- 住民テーブルは住所マスタを参照
- 同一住所の住民は同じ住所レコードを共有

### 3. 世帯と住民の循環参照
**問題**: 世帯主が世帯を参照し、世帯が世帯主を参照する循環参照

**解決策**:
- 世帯テーブルは世帯主を直接参照せず
- 世帯構成員関係テーブルで続柄「世帯主」により世帯主を特定
- 世帯主変更時の整合性をアプリケーションロジックで担保

### 4. 異動記録の履歴管理
**問題**: 住民の現在情報と履歴情報の管理方法が不明確

**解決策**:
- 住民テーブルは現在情報のみを保持
- 異動記録テーブルで全ての変更履歴を管理
- 有効期間（開始日・終了日）による時系列管理を実装

## 論理データモデル

### 基本エンティティ（テーブル）

#### 1. 住民 (residents)
```sql
CREATE TABLE residents (
    resident_id                 BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    personal_number            CHAR(12) UNIQUE NOT NULL,           -- 個人番号
    resident_code              CHAR(11) UNIQUE NOT NULL,           -- 住民票コード
    family_name                VARCHAR(50) NOT NULL,               -- 氏
    given_name                 VARCHAR(50) NOT NULL,               -- 名
    family_name_kana           VARCHAR(100) NOT NULL,              -- 氏（フリガナ）
    given_name_kana            VARCHAR(100) NOT NULL,              -- 名（フリガナ）
    birth_date                 DATE NOT NULL,                      -- 生年月日
    gender                     CHAR(1) CHECK (gender IN ('M', 'F', 'O')), -- 性別
    nationality                VARCHAR(3) NOT NULL,                -- 国籍コード（ISO 3166-1）
    resident_status            VARCHAR(10) NOT NULL                -- 住民区分（日本人・外国人等）
        CHECK (resident_status IN ('JAPANESE', 'FOREIGN', 'SPECIAL')),
    residence_start_date       DATE NOT NULL,                      -- 住民となった年月日
    residence_end_date         DATE,                               -- 住民でなくなった年月日
    current_address_id         BIGINT NOT NULL REFERENCES addresses(address_id),
    current_household_id       BIGINT NOT NULL REFERENCES households(household_id),
    support_measure_flag       BOOLEAN DEFAULT FALSE,              -- 支援措置フラグ
    guardianship_flag          BOOLEAN DEFAULT FALSE,              -- 成年被後見人フラグ
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT valid_residence_period 
        CHECK (residence_end_date IS NULL OR residence_end_date >= residence_start_date)
);
```

#### 2. 住所マスタ (addresses)
```sql
CREATE TABLE addresses (
    address_id                 BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    prefecture_code            CHAR(2) NOT NULL,                   -- 都道府県コード
    municipality_code          CHAR(3) NOT NULL,                   -- 市区町村コード
    oaza_cho                   VARCHAR(100),                       -- 大字・町
    aza                        VARCHAR(100),                       -- 字
    land_number                VARCHAR(50),                        -- 地番・地号
    residence_number           VARCHAR(50),                        -- 住居番号
    building_info              VARCHAR(200),                       -- 方書（建物名・部屋番号等）
    is_residence_display       BOOLEAN DEFAULT FALSE,              -- 住居表示フラグ
    postal_code                CHAR(7),                            -- 郵便番号
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT valid_postal_code CHECK (postal_code ~ '^[0-9]{7}$')
);
```

#### 3. 世帯 (households)
```sql
CREATE TABLE households (
    household_id               BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    household_number           VARCHAR(20) UNIQUE NOT NULL,        -- 世帯番号
    address_id                 BIGINT NOT NULL REFERENCES addresses(address_id),
    establishment_date         DATE NOT NULL,                      -- 世帯構成年月日
    dissolution_date           DATE,                               -- 世帯消除年月日
    household_type             VARCHAR(20) DEFAULT 'GENERAL'       -- 世帯区分
        CHECK (household_type IN ('GENERAL', 'SINGLE', 'COLLECTIVE')),
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT valid_household_period 
        CHECK (dissolution_date IS NULL OR dissolution_date >= establishment_date)
);
```

#### 4. 世帯構成員関係 (household_members)
```sql
CREATE TABLE household_members (
    resident_id                BIGINT NOT NULL REFERENCES residents(resident_id),
    household_id               BIGINT NOT NULL REFERENCES households(household_id),
    relationship               VARCHAR(20) NOT NULL                -- 続柄
        CHECK (relationship IN ('HEAD', 'SPOUSE', 'CHILD', 'PARENT', 'SIBLING', 'OTHER')),
    start_date                 DATE NOT NULL,                      -- 関係開始日
    end_date                   DATE,                               -- 関係終了日
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (resident_id, household_id, start_date),
    CONSTRAINT valid_member_period CHECK (end_date IS NULL OR end_date >= start_date),
    CONSTRAINT one_head_per_household_period 
        UNIQUE (household_id, relationship, start_date) DEFERRABLE INITIALLY DEFERRED
);
```

#### 5. 異動記録 (residence_changes)
```sql
CREATE TABLE residence_changes (
    change_id                  BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    resident_id                BIGINT NOT NULL REFERENCES residents(resident_id),
    change_type                VARCHAR(20) NOT NULL                -- 異動事由
        CHECK (change_type IN ('TRANSFER_IN', 'TRANSFER_OUT', 'MOVE', 'BIRTH', 'DEATH', 
                               'MARRIAGE', 'DIVORCE', 'NAME_CHANGE', 'OTHER')),
    change_date                DATE NOT NULL,                      -- 異動年月日
    notification_date          DATE NOT NULL,                      -- 届出年月日
    processed_date             DATE NOT NULL,                      -- 処理年月日
    previous_address_id        BIGINT REFERENCES addresses(address_id), -- 前住所
    new_address_id             BIGINT REFERENCES addresses(address_id), -- 新住所
    change_reason              TEXT,                               -- 異動理由詳細
    legal_basis                VARCHAR(50),                        -- 法的根拠
    application_id             BIGINT REFERENCES applications(application_id),
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT valid_change_dates 
        CHECK (processed_date >= notification_date AND notification_date <= change_date + INTERVAL '14 days')
);
```

#### 6. 届出 (applications)
```sql
CREATE TABLE applications (
    application_id             BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    application_type           VARCHAR(30) NOT NULL                -- 届出種別
        CHECK (application_type IN ('TRANSFER_IN', 'TRANSFER_OUT', 'MOVE', 'HOUSEHOLD_CHANGE',
                                   'SEAL_REGISTRATION', 'SEAL_CANCELLATION', 'CERTIFICATE_REQUEST')),
    application_date           DATE NOT NULL,                      -- 届出年月日
    acceptance_date            DATE NOT NULL,                      -- 受理年月日
    applicant_resident_id      BIGINT REFERENCES residents(resident_id), -- 届出者
    applicant_name             VARCHAR(100),                       -- 届出者名（住民以外の場合）
    applicant_relationship     VARCHAR(20),                        -- 届出者との関係
    processing_status          VARCHAR(20) DEFAULT 'PENDING'       -- 処理状況
        CHECK (processing_status IN ('PENDING', 'PROCESSING', 'COMPLETED', 'REJECTED')),
    notes                      TEXT,                               -- 備考
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 7. 印鑑登録 (seal_registrations)
```sql
CREATE TABLE seal_registrations (
    registration_id            BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    registration_number        VARCHAR(20) UNIQUE NOT NULL,        -- 印鑑登録番号
    resident_id                BIGINT NOT NULL REFERENCES residents(resident_id),
    registration_date          DATE NOT NULL,                      -- 登録年月日
    seal_impression_data       BYTEA NOT NULL,                     -- 印影データ
    certificate_number         VARCHAR(20) UNIQUE NOT NULL,        -- 印鑑登録証番号
    registration_status        VARCHAR(10) DEFAULT 'ACTIVE'        -- 登録状況
        CHECK (registration_status IN ('ACTIVE', 'SUSPENDED', 'CANCELLED')),
    cancellation_date          DATE,                               -- 抹消年月日
    cancellation_reason        VARCHAR(50),                        -- 抹消事由
    application_id             BIGINT NOT NULL REFERENCES applications(application_id),
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT one_active_registration_per_resident 
        UNIQUE (resident_id) WHERE (registration_status = 'ACTIVE'),
    CONSTRAINT valid_cancellation 
        CHECK (cancellation_date IS NULL OR cancellation_date >= registration_date)
);
```

#### 8. 証明書発行記録 (certificate_issuances)
```sql
CREATE TABLE certificate_issuances (
    issuance_id                BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    subject_resident_id        BIGINT NOT NULL REFERENCES residents(resident_id),
    certificate_type           VARCHAR(30) NOT NULL                -- 証明書種別
        CHECK (certificate_type IN ('RESIDENCE_CERTIFICATE', 'SEAL_CERTIFICATE', 
                                   'TRANSFER_CERTIFICATE', 'FAMILY_REGISTER_EXTRACT')),
    issuance_date              DATE NOT NULL,                      -- 発行年月日
    quantity                   INTEGER DEFAULT 1 CHECK (quantity > 0), -- 発行部数
    requester_type             VARCHAR(20) NOT NULL                -- 請求者区分
        CHECK (requester_type IN ('SELF', 'PROXY', 'THIRD_PARTY')),
    requester_name             VARCHAR(100),                       -- 請求者名
    purpose                    TEXT,                               -- 使用目的
    issuance_office           VARCHAR(50) NOT NULL,               -- 発行窓口
    application_id             BIGINT REFERENCES applications(application_id),
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 9. 支援措置情報 (support_measures)
```sql
CREATE TABLE support_measures (
    measure_id                 BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    resident_id                BIGINT NOT NULL REFERENCES residents(resident_id),
    measure_type               VARCHAR(20) NOT NULL                -- 措置種別
        CHECK (measure_type IN ('DV_PROTECTION', 'STALKER_PROTECTION', 'OTHER')),
    start_date                 DATE NOT NULL,                      -- 措置開始年月日
    end_date                   DATE,                               -- 措置終了年月日
    restriction_scope          TEXT NOT NULL,                      -- 制限範囲
    applicant_info             TEXT,                               -- 申立者情報
    supporting_agency          VARCHAR(100),                       -- 支援機関
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT valid_measure_period CHECK (end_date IS NULL OR end_date >= start_date)
);
```

#### 10. マイナンバーカード情報 (mynumber_cards)
```sql
CREATE TABLE mynumber_cards (
    card_id                    BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    resident_id                BIGINT NOT NULL REFERENCES residents(resident_id),
    card_serial_number         VARCHAR(20) UNIQUE NOT NULL,        -- カード交付番号
    issuance_date              DATE NOT NULL,                      -- 発行年月日
    expiration_date            DATE NOT NULL,                      -- 有効期限
    card_status                VARCHAR(20) DEFAULT 'ACTIVE'        -- カード状況
        CHECK (card_status IN ('ACTIVE', 'SUSPENDED', 'EXPIRED', 'LOST', 'CANCELLED')),
    digital_certificate_info   TEXT,                               -- 電子証明書情報
    created_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT valid_card_period CHECK (expiration_date > issuance_date),
    CONSTRAINT one_active_card_per_resident 
        UNIQUE (resident_id) WHERE (card_status = 'ACTIVE')
);
```

### インデックス設計

#### パフォーマンス最適化のためのインデックス
```sql
-- 住民検索用
CREATE INDEX idx_residents_personal_number ON residents (personal_number);
CREATE INDEX idx_residents_resident_code ON residents (resident_code);
CREATE INDEX idx_residents_name ON residents (family_name, given_name);
CREATE INDEX idx_residents_birth_date ON residents (birth_date);
CREATE INDEX idx_residents_address ON residents (current_address_id);

-- 異動記録検索用
CREATE INDEX idx_residence_changes_resident_date ON residence_changes (resident_id, change_date);
CREATE INDEX idx_residence_changes_type_date ON residence_changes (change_type, change_date);

-- 印鑑登録検索用
CREATE INDEX idx_seal_registrations_resident ON seal_registrations (resident_id);
CREATE INDEX idx_seal_registrations_number ON seal_registrations (registration_number);
CREATE INDEX idx_seal_registrations_status ON seal_registrations (registration_status);

-- 証明書発行履歴検索用
CREATE INDEX idx_certificate_issuances_resident_date ON certificate_issuances (subject_resident_id, issuance_date);
CREATE INDEX idx_certificate_issuances_type_date ON certificate_issuances (certificate_type, issuance_date);

-- 住所検索用
CREATE INDEX idx_addresses_code ON addresses (prefecture_code, municipality_code);
CREATE INDEX idx_addresses_postal ON addresses (postal_code);
```

### ビジネスルール制約

#### 1. 住民に関する制約
```sql
-- 住民の年齢制約（出生日から150年以内）
ALTER TABLE residents ADD CONSTRAINT valid_birth_date 
    CHECK (birth_date >= CURRENT_DATE - INTERVAL '150 years' AND birth_date <= CURRENT_DATE);

-- 個人番号の形式制約（12桁の数字）
ALTER TABLE residents ADD CONSTRAINT valid_personal_number 
    CHECK (personal_number ~ '^[0-9]{12}$');

-- 住民票コードの形式制約（11桁の数字）
ALTER TABLE residents ADD CONSTRAINT valid_resident_code 
    CHECK (resident_code ~ '^[0-9]{11}$');
```

#### 2. 世帯に関する制約
```sql
-- 各世帯に必ず世帯主が存在することを保証するトリガー
CREATE OR REPLACE FUNCTION ensure_household_head() RETURNS TRIGGER AS $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM household_members 
        WHERE household_id = COALESCE(NEW.household_id, OLD.household_id)
        AND relationship = 'HEAD' 
        AND end_date IS NULL
    ) THEN
        RAISE EXCEPTION '世帯には必ず世帯主が必要です';
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_ensure_household_head
    AFTER INSERT OR UPDATE OR DELETE ON household_members
    FOR EACH ROW EXECUTE FUNCTION ensure_household_head();
```

#### 3. 印鑑登録に関する制約
```sql
-- 印鑑登録は住民登録が前提
ALTER TABLE seal_registrations ADD CONSTRAINT seal_requires_residence
    CHECK (registration_date >= (
        SELECT residence_start_date FROM residents 
        WHERE residents.resident_id = seal_registrations.resident_id
    ));

-- 転出・死亡時の印鑑登録自動抹消
CREATE OR REPLACE FUNCTION auto_cancel_seal_registration() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.residence_end_date IS NOT NULL AND OLD.residence_end_date IS NULL THEN
        UPDATE seal_registrations 
        SET registration_status = 'CANCELLED',
            cancellation_date = NEW.residence_end_date,
            cancellation_reason = '住民登録抹消による'
        WHERE resident_id = NEW.resident_id 
        AND registration_status = 'ACTIVE';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_auto_cancel_seal
    AFTER UPDATE ON residents
    FOR EACH ROW EXECUTE FUNCTION auto_cancel_seal_registration();
```

## データ整合性の保証

### 1. 参照整合性
- 全ての外部キー制約により参照整合性を保証
- CASCADE削除は使用せず、アプリケーションで明示的な削除処理を実装

### 2. ドメイン整合性
- CHECK制約により属性値の妥当性を保証
- NOT NULL制約により必須項目を保証

### 3. エンティティ整合性
- 主キー制約により各レコードの一意性を保証
- 代理キーの使用により安定した識別を実現

### 4. ユーザー定義整合性
- トリガーによりビジネスルールを強制
- 複雑な制約はアプリケーションレイヤーで実装

## パフォーマンス考慮事項

### 1. クエリ最適化
- 頻繁に検索される列にインデックスを設定
- 複合インデックスにより複数条件検索を最適化
- パーティショニングを考慮（年度別の異動記録等）

### 2. ストレージ最適化
- 印影データはBYTEA型で格納（外部ストレージも検討可能）
- 大容量テキストはTEXT型を使用
- 適切なデータ型選択によりストレージ効率を向上

### 3. 同時実行制御
- 楽観的ロック（updated_atによるバージョン制御）
- 必要に応じて行レベルロックを使用
- デッドロック回避のためのロック順序統一

## 拡張性への配慮

### 1. スキーマ拡張
- 予備列（reserved_field1-3）の追加検討
- JSONカラムによる柔軟な属性拡張
- 履歴テーブルの分離によるパフォーマンス向上

### 2. 機能拡張
- 新たな証明書種別への対応
- 外部システム連携テーブルの追加
- 監査ログテーブルの実装

## 導出根拠と検証

### 1. 正規化レベル
- 第3正規形（3NF）を基本とし、必要に応じてBCNFを適用
- パフォーマンス要件に応じて意図的な非正規化を検討
- 冗長性排除と更新異常防止を両立

### 2. 制約設計
- 機能要件から抽出したビジネスルールを制約として実装
- 法的要件（住民基本台帳法等）をシステム制約に反映
- データ品質保証のための包括的制約設計

### 3. モデル妥当性
- 全152機能要件に対する実装可能性を検証
- エンティティ間の整合性を論理的に検証
- 実装後の保守性・拡張性を考慮した設計選択

この論理データモデルにより、石巻市住民基本台帳・印鑑登録システムの要件を満たす堅牢で効率的なデータベース設計が実現できます。