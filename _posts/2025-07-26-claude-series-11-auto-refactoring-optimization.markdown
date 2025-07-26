---
layout: post
title: "[Claude 입문 #11] 자동 리팩토링과 최적화"
date: 2025-07-26 10:00:00 +0900
categories: [AI, Claude]
tags: [claude, ai, tutorial, beginner, series, refactoring, optimization, claude-code, automation]
mermaid: true
---

## 코드를 더 나은 코드로

Claude Code의 자동 리팩토링 기능은 단순한 코드 정리를 넘어 성능 최적화, 가독성 향상, 유지보수성 개선을 동시에 달성합니다. AI가 코드의 의도를 이해하고 더 나은 구현을 제안합니다.

### Claude Code 리팩토링 기능 개요

| 기능 | 설명 | 효과 | 난이도 |
|------|------|------|---------|
| **코드 스멜 감지** | 문제가 있는 코드 패턴 자동 발견 | 코드 품질 향상 | 🟢 쉬움 |
| **디자인 패턴 적용** | 적절한 패턴으로 자동 변환 | 구조 개선 | 🔴 어려움 |
| **성능 최적화** | 알고리즘 및 메모리 최적화 | 실행 속도 향상 | 🟡 보통 |
| **코드 현대화** | 레거시 코드를 최신 문법으로 | 유지보수성 향상 | 🟢 쉬움 |
| **일괄 리팩토링** | 프로젝트 전체 일관성 적용 | 코드 통일성 | 🟡 보통 |

## 리팩토링 기본 기능

### 1. 코드 스멜 자동 감지

```mermaid
graph TD
    A[코드 스캔] --> B[스멜 감지]
    B --> C[긴 메서드]
    B --> D[중복 코드]
    B --> E[복잡한 조건문]
    B --> F[과도한 주석]
    
    C --> G[메서드 추출]
    D --> H[공통 함수화]
    E --> I[조건 단순화]
    F --> J[자체 설명적 코드]
```

#### 실시간 감지 예시
```javascript
// 원본 코드 - Claude가 자동으로 문제점 표시
function processOrder(order) {
  // 🔴 Claude: 함수가 너무 깁니다 (150줄)
  let total = 0;
  let discount = 0;
  
  // 재고 확인
  for(let item of order.items) {
    if(inventory[item.id] < item.quantity) {
      return {error: "재고 부족"};
    }
  }
  
  // 가격 계산
  for(let item of order.items) {
    total += item.price * item.quantity;
  }
  
  // 할인 적용 - 🟡 Claude: 복잡한 조건문
  if(order.customer.type === 'vip') {
    if(total > 100000) {
      discount = 0.2;
    } else if(total > 50000) {
      discount = 0.1;
    }
  } else if(order.customer.type === 'regular') {
    if(total > 200000) {
      discount = 0.1;
    }
  }
  
  // ... 더 많은 코드
}
```

### 2. 원클릭 리팩토링

```javascript
// Claude 제안: 메서드 추출
function processOrder(order) {
  const stockCheck = checkInventory(order.items);
  if(stockCheck.error) return stockCheck;
  
  const subtotal = calculateSubtotal(order.items);
  const discount = calculateDiscount(subtotal, order.customer);
  const total = applyDiscount(subtotal, discount);
  
  return {
    subtotal,
    discount,
    total,
    items: order.items
  };
}

function checkInventory(items) {
  for(let item of items) {
    if(inventory[item.id] < item.quantity) {
      return {error: `재고 부족: ${item.name}`};
    }
  }
  return {success: true};
}

function calculateDiscount(amount, customer) {
  const discountRules = {
    vip: [{min: 100000, rate: 0.2}, {min: 50000, rate: 0.1}],
    regular: [{min: 200000, rate: 0.1}]
  };
  
  const rules = discountRules[customer.type] || [];
  const applicable = rules.find(rule => amount >= rule.min);
  return applicable ? applicable.rate : 0;
}
```

## 고급 리팩토링 패턴

### 1. 디자인 패턴 적용

```typescript
// 원본: 조건문으로 가득한 코드
class NotificationService {
  send(type: string, message: string) {
    if(type === 'email') {
      // 이메일 전송 로직
    } else if(type === 'sms') {
      // SMS 전송 로직
    } else if(type === 'push') {
      // 푸시 알림 로직
    }
  }
}

// Claude 리팩토링: Strategy 패턴 적용
interface NotificationStrategy {
  send(message: string): Promise<void>;
}

class EmailNotification implements NotificationStrategy {
  async send(message: string) {
    // 이메일 전송 구현
  }
}

class SMSNotification implements NotificationStrategy {
  async send(message: string) {
    // SMS 전송 구현
  }
}

class NotificationService {
  private strategies: Map<string, NotificationStrategy>;
  
  constructor() {
    this.strategies = new Map([
      ['email', new EmailNotification()],
      ['sms', new SMSNotification()],
      ['push', new PushNotification()]
    ]);
  }
  
  async send(type: string, message: string) {
    const strategy = this.strategies.get(type);
    if(!strategy) throw new Error(`Unknown notification type: ${type}`);
    return strategy.send(message);
  }
}
```

### 2. 함수형 프로그래밍 전환

```javascript
// 명령형 코드
let result = [];
for(let i = 0; i < users.length; i++) {
  if(users[i].age >= 18) {
    result.push({
      name: users[i].name,
      email: users[i].email
    });
  }
}

// Claude 리팩토링: 함수형 스타일
const result = users
  .filter(user => user.age >= 18)
  .map(({name, email}) => ({name, email}));

// 더 복잡한 예시
// 원본
let groupedOrders = {};
for(let order of orders) {
  if(!groupedOrders[order.customerId]) {
    groupedOrders[order.customerId] = [];
  }
  groupedOrders[order.customerId].push(order);
}

let totals = {};
for(let customerId in groupedOrders) {
  totals[customerId] = 0;
  for(let order of groupedOrders[customerId]) {
    totals[customerId] += order.amount;
  }
}

// 리팩토링
const totals = orders
  .reduce((acc, order) => {
    const {customerId, amount} = order;
    acc[customerId] = (acc[customerId] || 0) + amount;
    return acc;
  }, {});
```

## 성능 최적화

### 1. 알고리즘 최적화

```python
# 원본: O(n²) 복잡도
def find_pairs_with_sum(arr, target):
    pairs = []
    for i in range(len(arr)):
        for j in range(i+1, len(arr)):
            if arr[i] + arr[j] == target:
                pairs.append((arr[i], arr[j]))
    return pairs

# Claude 최적화: O(n) 복잡도
def find_pairs_with_sum_optimized(arr, target):
    seen = set()
    pairs = []
    
    for num in arr:
        complement = target - num
        if complement in seen:
            pairs.append((complement, num))
        seen.add(num)
    
    return pairs

# 성능 비교
"""
원본 알고리즘:
- 1,000개 요소: 0.15초
- 10,000개 요소: 15초
- 100,000개 요소: 25분

최적화 버전:
- 1,000개 요소: 0.001초
- 10,000개 요소: 0.01초
- 100,000개 요소: 0.1초
"""
```

### 2. 메모리 최적화

```javascript
// 메모리 비효율적 코드
class DataProcessor {
  constructor() {
    this.cache = {};  // 무제한 캐시
  }
  
  process(id) {
    if(!this.cache[id]) {
      this.cache[id] = expensiveOperation(id);
    }
    return this.cache[id];
  }
}

// Claude 최적화: LRU 캐시 구현
class DataProcessor {
  constructor(maxSize = 1000) {
    this.cache = new Map();
    this.maxSize = maxSize;
  }
  
  process(id) {
    if(this.cache.has(id)) {
      // LRU: 최근 사용 항목을 끝으로 이동
      const value = this.cache.get(id);
      this.cache.delete(id);
      this.cache.set(id, value);
      return value;
    }
    
    const value = expensiveOperation(id);
    this.cache.set(id, value);
    
    // 크기 제한 적용
    if(this.cache.size > this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    return value;
  }
}
```

### 3. 데이터베이스 쿼리 최적화

```sql
-- 원본: N+1 쿼리 문제
SELECT * FROM users;
-- 각 사용자마다 추가 쿼리
SELECT * FROM orders WHERE user_id = ?;

-- Claude 최적화: JOIN 사용
SELECT 
  u.*,
  o.id as order_id,
  o.amount,
  o.created_at as order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.active = true
ORDER BY u.id, o.created_at DESC;

-- 또는 ORM 레벨 최적화
// Sequelize 예시
const users = await User.findAll({
  include: [{
    model: Order,
    as: 'orders',
    required: false
  }],
  where: { active: true }
});
```

## 코드 현대화

### 1. 레거시 코드 업그레이드

```javascript
// ES5 코드
var that = this;
function callback(data) {
  that.data = data;
  that.process();
}

// Claude 현대화: ES6+
const callback = (data) => {
  this.data = data;
  this.process();
};

// 더 복잡한 예시
// 원본: 콜백 지옥
getData(function(a) {
  getMoreData(a, function(b) {
    getMoreData(b, function(c) {
      getMoreData(c, function(d) {
        getMoreData(d, function(e) {
          console.log(e);
        });
      });
    });
  });
});

// 현대화: async/await
try {
  const a = await getData();
  const b = await getMoreData(a);
  const c = await getMoreData(b);
  const d = await getMoreData(c);
  const e = await getMoreData(d);
  console.log(e);
} catch(error) {
  console.error('Error:', error);
}
```

### 2. 타입 안정성 추가

```typescript
// JavaScript → TypeScript 변환
// 원본
function createUser(name, email, age) {
  return {
    id: generateId(),
    name: name,
    email: email,
    age: age,
    createdAt: new Date()
  };
}

// Claude 타입 추가
interface User {
  id: string;
  name: string;
  email: string;
  age: number;
  createdAt: Date;
}

interface CreateUserInput {
  name: string;
  email: string;
  age: number;
}

function createUser({name, email, age}: CreateUserInput): User {
  return {
    id: generateId(),
    name,
    email,
    age,
    createdAt: new Date()
  };
}

// 유효성 검사 추가
function createUser(input: CreateUserInput): User {
  const {name, email, age} = validateUserInput(input);
  
  return {
    id: generateId(),
    name,
    email,
    age,
    createdAt: new Date()
  };
}

function validateUserInput(input: CreateUserInput): CreateUserInput {
  if(!input.name || input.name.length < 2) {
    throw new Error('이름은 2자 이상이어야 합니다');
  }
  
  if(!isValidEmail(input.email)) {
    throw new Error('유효한 이메일 주소가 아닙니다');
  }
  
  if(input.age < 0 || input.age > 150) {
    throw new Error('나이는 0-150 사이여야 합니다');
  }
  
  return input;
}
```

## 일괄 리팩토링

### 1. 프로젝트 전체 업그레이드

```bash
# Claude Code 명령어
claude refactor --project-wide --type modernize

# 실행 계획
다음 변경사항이 적용됩니다:
1. var → const/let 변환 (234개 파일)
2. 콜백 → Promise/async 변환 (45개 파일)
3. require → import 변환 (156개 파일)
4. 클래스 문법 현대화 (23개 파일)
5. 옵셔널 체이닝 적용 (89개 위치)

예상 시간: 5분
백업 생성: refactor-backup-20250726

계속하시겠습니까? (Y/n)
```

### 2. 코딩 스타일 통일

```yaml
# .claude/refactor-rules.yaml
style:
  quotes: single
  semicolons: true
  trailing_comma: es5
  indent: 2
  line_length: 80

naming:
  variables: camelCase
  constants: UPPER_SNAKE_CASE
  classes: PascalCase
  files: kebab-case

structure:
  max_file_length: 300
  max_function_length: 50
  max_parameters: 4
```

## 리팩토링 전략

### 점진적 개선 로드맵

```mermaid
graph LR
    A[현재 상태] --> B[1단계: 긴급]
    B --> C[보안 취약점]
    B --> D[성능 병목]
    
    E[2단계: 중요]
    E --> F[코드 중복]
    E --> G[복잡도 감소]
    
    H[3단계: 개선]
    H --> I[가독성]
    H --> J[현대화]
    
    B --> E --> H
```

### 리팩토링 우선순위 매트릭스

| 영향도/노력 | 낮은 노력 | 높은 노력 |
|------------|----------|----------|
| **높은 영향** | 즉시 실행 | 계획 수립 |
| **낮은 영향** | 시간 날 때 | 보류 |

### 일반적인 리팩토링 패턴과 효과

| 패턴 | 적용 전 | 적용 후 | 개선 효과 |
|------|---------|---------|-----------|
| **메서드 추출** | 100줄 함수 | 10-20줄 함수 5개 | 가독성 80% ↑ |
| **조건문 단순화** | 중첩 if 5단계 | Guard clause | 복잡도 70% ↓ |
| **루프 최적화** | 중첩 forEach | map/filter/reduce | 성능 60% ↑ |
| **타입 추가** | JavaScript | TypeScript | 버그 40% ↓ |
| **비동기 현대화** | 콜백 지옥 | async/await | 가독성 90% ↑ |

## 테스트 주도 리팩토링

### 리팩토링 전 테스트 확보

```javascript
// 1. 기존 동작 테스트 작성
describe('OrderService', () => {
  it('should calculate total with discount', () => {
    const order = {
      items: [{price: 100, quantity: 2}],
      customer: {type: 'vip'}
    };
    
    const result = processOrder(order);
    expect(result.total).toBe(160); // 20% 할인 적용
  });
});

// 2. 리팩토링 진행
// Claude가 테스트를 통과하는 새로운 구현 생성

// 3. 테스트 재실행으로 동작 보장
```

## 성능 모니터링

### 리팩토링 전후 비교

```javascript
// Claude 성능 리포트
{
  "before": {
    "executionTime": "234ms",
    "memoryUsage": "45MB",
    "cpuUsage": "78%"
  },
  "after": {
    "executionTime": "89ms", // 62% 개선
    "memoryUsage": "23MB",  // 49% 감소
    "cpuUsage": "34%"       // 56% 감소
  },
  "improvements": {
    "algorithmic": "O(n²) → O(n log n)",
    "memory": "메모리 누수 제거",
    "caching": "LRU 캐시 도입"
  }
}
```

## 2025년 Claude Code 최신 리팩토링 기능

### 새로운 AI 기반 기능들

```mermaid
graph TD
    A[Claude Code 2025] --> B[지능형 분석]
    B --> C[컨텍스트 인식 리팩토링]
    B --> D[비즈니스 로직 보존]
    B --> E[테스트 자동 생성]
    
    A --> F[실시간 제안]
    F --> G[코딩 중 즉시 피드백]
    F --> H[대안 구현 제시]
    
    A --> I[팀 협업]
    I --> J[코드 스타일 학습]
    I --> K[팀 규칙 자동 적용]
```

### 최신 리팩토링 명령어

| 명령어 | 기능 | 사용 예시 |
|--------|------|----------|
| `claude refactor --ai-suggest` | AI 기반 제안 | 최적 구현 패턴 추천 |
| `claude refactor --performance` | 성능 특화 최적화 | 병목 지점 자동 개선 |
| `claude refactor --security` | 보안 중심 리팩토링 | 취약점 자동 수정 |
| `claude refactor --test-safe` | 테스트 보장 리팩토링 | 동작 변경 없이 개선 |
| `claude refactor --team-style` | 팀 스타일 적용 | 코딩 컨벤션 자동화 |

## 다음 편 예고

다음 편에서는 "테스트 코드 작성 자동화"를 다룰 예정입니다. Claude Code를 활용해 효과적인 테스트를 자동으로 생성하는 방법을 알아보겠습니다.

---

💡 **오늘의 과제**: 가장 복잡한 함수 하나를 선택해 Claude Code로 리팩토링해보세요. 리팩토링 전후의 복잡도와 성능을 비교해보세요!