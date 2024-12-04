行程成本更新的 implementation：
 
行程 A 由體驗 a & b 組成，
在某一時間點成立的訂單 001（訂單內容為行程 A），訂單成立時行程 A 的總價是 $150（體驗 a 單價 $100 + 體驗 b 單價 $50）
在訂單 001 成立後，體驗 b 漲價至 $60
在某一時間點成立的訂單 002（訂單內容為行程 A），訂單成立時行程 A 的總價是 $160（體驗 a 單價 $100 + 體驗 b 單價 $60）
如何在資料處理上讓訂單 001 的價格保持在 $150，但訂單 002 的價格是 $160


# 資料表設計

使用 postgres 資料庫，以下是資料表設計：

```
// 體驗表：記錄每個體驗的基本信息和價格歷史
Table experiences {
  experience_id VARCHAR(50) [primary key]
  name VARCHAR(100)
  price_history JSONB
}

// price_history examples: 
// [
//   {
//     "price": 100.00, 
//     "timestamp": "2023-01-01T10:00:00",
//     "reason": "初始定價"
//   },
//   {
//     "price": 120.00, 
//     "timestamp": "2023-06-15T14:30:00", 
//     "reason": "季節性調整"
//   }
// ]

// 行程表：定義行程的基本信息
Table trips {
  trip_id VARCHAR(50) [primary key]
  name VARCHAR(100)
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
}

// 行程-體驗關聯表：記錄行程中包含的體驗
Table trip_experiences {
  trip_id VARCHAR(50)
  experience_id VARCHAR(50)
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  
  // 複合主鍵
  Indexes {
    (trip_id, experience_id) [pk]
  }
}

// 訂單表：記錄每筆訂單的詳細信息
Table orders {
  order_id VARCHAR(50) [primary key]
  trip_id VARCHAR(50)
  order_time TIMESTAMP
  total_price DECIMAL(10,2)
}

// 定義外鍵關係
Ref: trip_experiences.trip_id > trips.trip_id
Ref: trip_experiences.experience_id > experiences.experience_id
Ref: orders.trip_id > trips.trip_id
```

# 價格計算函數

```sql
-- 創建函數來獲取指定時間點的有效價格
CREATE OR REPLACE FUNCTION get_current_price(experience_id_param VARCHAR)
RETURNS DECIMAL AS $$
BEGIN
    RETURN (
        SELECT (value->>'price')::decimal
        FROM experiences,
             jsonb_array_elements(price_history) AS ph(value)
        WHERE experience_id = experience_id_param
        AND (value->>'timestamp')::timestamp <= CURRENT_TIMESTAMP
        ORDER BY (value->>'timestamp')::timestamp DESC
        LIMIT 1
    );
END;
$$ LANGUAGE plpgsql;

-- 創建視圖來查詢行程當前價格
CREATE OR REPLACE VIEW trip_current_prices AS
SELECT 
    t.trip_id,
    t.name,
    SUM(get_current_price(te.experience_id)) as current_total_price
FROM trips t
JOIN trip_experiences te ON t.trip_id = te.trip_id
GROUP BY t.trip_id, t.name;
```

# 初始化體驗表

```sql
INSERT INTO experiences (experience_id, name, price_history)
VALUES 
(
  'exp_a',
  '體驗A',
  jsonb_build_array(
    jsonb_build_object(
      'price', 100.00,
      'timestamp', to_char(CURRENT_TIMESTAMP, 'YYYY-MM-DD"T"HH24:MI:SS'),
      'reason', '初始定價'
    )
  )
),
(
  'exp_b',
  '體驗B',
  jsonb_build_array(
    jsonb_build_object(
      'price', 50.00,
      'timestamp', to_char(CURRENT_TIMESTAMP, 'YYYY-MM-DD"T"HH24:MI:SS'),
      'reason', '初始定價'
    )
  )
);
```

# 初始化行程表

```sql
INSERT INTO trips (trip_id, name)
VALUES ('trip_a', '行程A');

-- 建立行程 A 與體驗的關聯
INSERT INTO trip_experiences (trip_id, experience_id)
VALUES 
('trip_a', 'exp_a'),
('trip_a', 'exp_b');
```

# 新增訂單

```sql
-- 新增訂單 001
INSERT INTO orders (order_id, trip_id, order_time, total_price)
SELECT 
  'order_001',
  'trip_a',
  CURRENT_TIMESTAMP,
  (
    SELECT SUM(get_current_price(te.experience_id))
    FROM trip_experiences te
    WHERE te.trip_id = 'trip_a'
  );
```

# 更新體驗價格

```sql
-- 更新體驗 B 的價格從 $50 漲到 $60
UPDATE experiences 
SET price_history = price_history || jsonb_build_array(
    jsonb_build_object(
      'price', 60.00,
      'timestamp', CURRENT_TIMESTAMP,
      'reason', '價格調整'
    )
  )
WHERE experience_id = 'exp_b';
```

# 設定未來價格

```sql
-- 為體驗 B 新增包含未來價格的記錄
UPDATE experiences 
SET price_history = price_history || jsonb_build_array(
    jsonb_build_object(
      'price', 85.00,
      'timestamp', CURRENT_TIMESTAMP + INTERVAL '7 days',
      'reason', '春節期間價格'
    ),
    jsonb_build_object(
      'price', 90.00,
      'timestamp', CURRENT_TIMESTAMP + INTERVAL '14 days',
      'reason', '春節旺季價格'
    ),
    jsonb_build_object(
      'price', 80.00,
      'timestamp', CURRENT_TIMESTAMP + INTERVAL '30 days',
      'reason', '春節後恢復原價'
    )
  )
WHERE experience_id = 'exp_b';
```

# 查詢特定時間的價格

```sql
-- 查詢特定時間點的價格
WITH price_at_time AS (
  SELECT 
    experience_id,
    (
      SELECT (value->>'price')::decimal
      FROM jsonb_array_elements(price_history) AS ph(value)
      WHERE (value->>'timestamp')::timestamp <= CURRENT_TIMESTAMP  -- 改變這個時間點來查詢不同時間的價格
      ORDER BY (value->>'timestamp')::timestamp DESC
      LIMIT 1
    ) as effective_price
  FROM experiences
  WHERE experience_id = 'exp_b'
)
SELECT * FROM price_at_time;
```

# 指定下單時間的訂單

```sql
-- 在 2024-12-18 新增訂單 003
INSERT INTO orders (order_id, trip_id, order_time, total_price)
VALUES (
  'order_003',
  'trip_a',
  '2024-12-18 00:00:00'::timestamp,  -- 指定下單時間
  (
    SELECT SUM(
      (
        SELECT (value->>'price')::decimal
        FROM jsonb_array_elements(e.price_history) AS ph(value)
        WHERE (value->>'timestamp')::timestamp <= '2024-12-18 00:00:00'::timestamp
        ORDER BY (value->>'timestamp')::timestamp DESC
        LIMIT 1
      )
    )
    FROM trip_experiences te
    JOIN experiences e ON e.experience_id = te.experience_id
    WHERE te.trip_id = 'trip_a'
  )
);
```
