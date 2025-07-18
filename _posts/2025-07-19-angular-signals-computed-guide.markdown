---
layout: post
title: "[Angular] Signal과 Computed 완벽 가이드: 반응형 프로그래밍의 새로운 패러다임"
date: 2025-07-19 10:00:00 +0900
categories: [Angular, Frontend]
tags: [angular, signals, computed, reactive-programming, state-management, typescript, frontend, tutorial]
mermaid: true
---

## 들어가며

Angular 16에서 도입된 Signal은 반응형 프로그래밍의 새로운 패러다임을 제시합니다. RxJS의 복잡성 없이도 효율적인 상태 관리와 반응형 프로그래밍이 가능해졌습니다. 이번 포스트에서는 Signal과 Computed의 개념부터 실전 활용법까지 상세히 알아보겠습니다.

## 1. Signal이란?

### Signal의 개념

Signal은 **시간에 따라 변할 수 있는 값을 담는 반응형 컨테이너**입니다. 값이 변경되면 자동으로 의존하는 곳에 알림이 전달되어 UI가 업데이트됩니다.

```mermaid
graph LR
    A[Signal 값 변경] --> B[의존성 추적]
    B --> C[자동 업데이트]
    C --> D[UI 리렌더링]
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#9f9,stroke:#333,stroke-width:2px
```

### 기존 방식과의 비교

| 특징 | 전통적인 방식 | Signal 방식 | 장점 |
|------|--------------|-------------|------|
| **상태 관리** | 일반 변수 | 반응형 컨테이너 | 자동 추적 |
| **변경 감지** | 수동 체크 | 자동 감지 | 효율적 |
| **업데이트** | 전체 컴포넌트 | 필요한 부분만 | 성능 최적화 |
| **구독 관리** | 수동 구독/해제 | 자동 관리 | 메모리 누수 방지 |
| **학습 곡선** | RxJS 필요 | 간단한 API | 쉬운 학습 |

## 2. Signal 기본 사용법

### Signal 생성하기

```typescript
import { signal, WritableSignal } from '@angular/core';

// 기본 Signal 생성
const count = signal(0);  // 초기값 0

// 타입 명시
const name: WritableSignal<string> = signal('Angular');

// 객체 Signal
const user = signal({
  id: 1,
  name: 'John',
  email: 'john@example.com'
});
```

### Signal 값 읽기와 쓰기

```typescript
// 값 읽기 - 함수 호출
console.log(count());  // 0
console.log(name());   // 'Angular'

// 값 쓰기 - set 메서드
count.set(5);
name.set('Angular Signals');

// 값 업데이트 - update 메서드
count.update(value => value + 1);  // 6
user.update(u => ({ ...u, name: 'Jane' }));
```

### 실전 예제: 카운터 컴포넌트

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div class="counter">
      <h2>카운터: {{ count() }}</h2>
      <button (click)="increment()">+1</button>
      <button (click)="decrement()">-1</button>
      <button (click)="reset()">리셋</button>
    </div>
  `,
  styles: [`
    .counter {
      text-align: center;
      padding: 20px;
    }
    button {
      margin: 0 5px;
      padding: 10px 20px;
    }
  `]
})
export class CounterComponent {
  // Signal 생성
  count = signal(0);

  increment() {
    this.count.update(value => value + 1);
  }

  decrement() {
    this.count.update(value => value - 1);
  }

  reset() {
    this.count.set(0);
  }
}
```

## 3. Computed - 파생된 상태 관리

### Computed란?

Computed는 **다른 Signal들로부터 파생된 읽기 전용 Signal**입니다. 의존하는 Signal이 변경될 때마다 자동으로 재계산됩니다.

```mermaid
flowchart TD
    A[Signal A] --> C[Computed]
    B[Signal B] --> C
    C --> D[파생된 값]
    
    style C fill:#bbf,stroke:#333,stroke-width:2px
    style D fill:#9f9,stroke:#333,stroke-width:2px
```

### Computed 기본 사용법

```typescript
import { signal, computed } from '@angular/core';

const price = signal(100);
const quantity = signal(2);
const taxRate = signal(0.1);

// Computed Signal 생성
const subtotal = computed(() => price() * quantity());
const tax = computed(() => subtotal() * taxRate());
const total = computed(() => subtotal() + tax());

console.log(total());  // 220

// 의존 Signal 변경 시 자동 재계산
quantity.set(3);
console.log(total());  // 330
```

### Computed의 특징

| 특징 | 설명 | 예시 |
|------|------|------|
| **읽기 전용** | 직접 값 설정 불가 | `computed.set(X)` ❌ |
| **자동 메모이제이션** | 동일 입력에 대해 캐싱 | 성능 최적화 |
| **지연 평가** | 실제 사용 시에만 계산 | 효율적 실행 |
| **동적 의존성** | 조건부 의존성 가능 | 유연한 로직 |

### 실전 예제: 쇼핑 카트

```typescript
@Component({
  selector: 'app-shopping-cart',
  template: `
    <div class="cart">
      <h2>쇼핑 카트</h2>
      
      <div class="items">
        <div *ngFor="let item of items()">
          <span>{{ item.name }}</span>
          <input type="number" 
                 [value]="item.quantity" 
                 (input)="updateQuantity(item.id, +$event.target.value)">
          <span>₩{{ item.price }}</span>
        </div>
      </div>
      
      <div class="summary">
        <p>소계: ₩{{ subtotal() }}</p>
        <p>부가세 (10%): ₩{{ tax() }}</p>
        <p>배송비: ₩{{ shippingFee() }}</p>
        <hr>
        <h3>총계: ₩{{ total() }}</h3>
      </div>
    </div>
  `
})
export class ShoppingCartComponent {
  items = signal([
    { id: 1, name: '노트북', price: 1500000, quantity: 1 },
    { id: 2, name: '마우스', price: 50000, quantity: 2 }
  ]);

  // Computed Signals
  subtotal = computed(() => 
    this.items().reduce((sum, item) => 
      sum + (item.price * item.quantity), 0
    )
  );

  tax = computed(() => this.subtotal() * 0.1);

  shippingFee = computed(() => 
    this.subtotal() > 100000 ? 0 : 3000
  );

  total = computed(() => 
    this.subtotal() + this.tax() + this.shippingFee()
  );

  updateQuantity(id: number, quantity: number) {
    this.items.update(items => 
      items.map(item => 
        item.id === id ? { ...item, quantity } : item
      )
    );
  }
}
```

## 4. Effect - 부수 효과 처리

### Effect란?

Effect는 **Signal의 변화에 반응하여 부수 효과를 실행하는 함수**입니다. 로깅, API 호출, localStorage 저장 등에 사용됩니다.

```mermaid
sequenceDiagram
    participant S as Signal
    participant E as Effect
    participant SE as Side Effect
    
    S->>E: 값 변경 감지
    E->>SE: 부수 효과 실행
    Note over SE: 로깅, API 호출,<br/>저장소 업데이트 등
```

### Effect 사용법

```typescript
import { Component, signal, effect } from '@angular/core';

@Component({
  selector: 'app-user-tracker',
  template: `...`
})
export class UserTrackerComponent {
  currentUser = signal<string>('Guest');
  loginCount = signal(0);

  constructor() {
    // Effect 생성 - constructor나 injection context에서
    effect(() => {
      console.log(`현재 사용자: ${this.currentUser()}`);
      console.log(`로그인 횟수: ${this.loginCount()}`);
      
      // localStorage에 저장
      localStorage.setItem('currentUser', this.currentUser());
    });

    // 조건부 실행
    effect(() => {
      if (this.loginCount() > 5) {
        console.warn('잦은 로그인 시도 감지');
      }
    });
  }

  login(username: string) {
    this.currentUser.set(username);
    this.loginCount.update(count => count + 1);
  }
}
```

### Effect와 Cleanup

```typescript
effect((onCleanup) => {
  const user = this.currentUser();
  
  // 1초 후 알림
  const timer = setTimeout(() => {
    console.log(`${user}님 환영합니다!`);
  }, 1000);

  // Cleanup 함수 등록
  onCleanup(() => {
    clearTimeout(timer);
  });
});
```

## 5. asReadonly - 읽기 전용 Signal

### asReadonly란?

`asReadonly`는 **WritableSignal을 읽기 전용 Signal로 변환**하는 함수입니다. 이를 통해 Signal의 값을 외부에서 변경할 수 없도록 보호할 수 있습니다.

### 기본 사용법

```typescript
import { signal, Signal, WritableSignal } from '@angular/core';

// WritableSignal 생성
const writableCount = signal(0);

// 읽기 전용 Signal로 변환
const readonlyCount: Signal<number> = writableCount.asReadonly();

// 읽기는 가능
console.log(readonlyCount()); // 0

// 쓰기는 불가능 (컴파일 에러)
// readonlyCount.set(5); // ❌ Error: Property 'set' does not exist
// readonlyCount.update(v => v + 1); // ❌ Error: Property 'update' does not exist
```

### asReadonly 활용 패턴

#### 1. 서비스에서 상태 캡슐화

```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  // Private WritableSignal
  private _currentUser = signal<User | null>(null);
  private _isAuthenticated = signal(false);
  private _permissions = signal<string[]>([]);

  // Public readonly Signal
  public currentUser = this._currentUser.asReadonly();
  public isAuthenticated = this._isAuthenticated.asReadonly();
  public permissions = this._permissions.asReadonly();

  // Computed Signal도 자동으로 readonly
  public hasAdminAccess = computed(() => 
    this.permissions().includes('admin')
  );

  // 메서드를 통해서만 상태 변경 가능
  login(user: User, permissions: string[]) {
    this._currentUser.set(user);
    this._permissions.set(permissions);
    this._isAuthenticated.set(true);
  }

  logout() {
    this._currentUser.set(null);
    this._permissions.set([]);
    this._isAuthenticated.set(false);
  }

  updateProfile(updates: Partial<User>) {
    this._currentUser.update(user => 
      user ? { ...user, ...updates } : null
    );
  }
}
```

#### 2. 컴포넌트 상태 보호

```typescript
@Component({
  selector: 'app-shopping-cart-service',
  template: `
    <div class="cart">
      <h2>장바구니 ({{ cartService.itemCount() }}개)</h2>
      
      <div *ngFor="let item of cartService.items()">
        <span>{{ item.name }} - {{ item.quantity }}개</span>
        <button (click)="cartService.updateQuantity(item.id, item.quantity + 1)">+</button>
        <button (click)="cartService.updateQuantity(item.id, item.quantity - 1)">-</button>
        <button (click)="cartService.removeItem(item.id)">삭제</button>
      </div>
      
      <div class="summary">
        <p>총액: ₩{{ cartService.total() | number }}</p>
      </div>
    </div>
  `
})
export class ShoppingCartComponent {
  constructor(public cartService: CartService) {}
}

@Injectable({ providedIn: 'root' })
export class CartService {
  // Private 상태
  private _items = signal<CartItem[]>([]);
  
  // Public 읽기 전용 접근
  public items = this._items.asReadonly();
  
  // Computed는 자동으로 readonly
  public itemCount = computed(() => 
    this.items().reduce((sum, item) => sum + item.quantity, 0)
  );
  
  public total = computed(() =>
    this.items().reduce((sum, item) => 
      sum + (item.price * item.quantity), 0
    )
  );

  // 제어된 변경 메서드
  addItem(product: Product) {
    this._items.update(items => {
      const existing = items.find(item => item.id === product.id);
      if (existing) {
        return items.map(item =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      }
      return [...items, { ...product, quantity: 1 }];
    });
  }

  updateQuantity(id: string, quantity: number) {
    if (quantity <= 0) {
      this.removeItem(id);
      return;
    }
    
    this._items.update(items =>
      items.map(item =>
        item.id === id ? { ...item, quantity } : item
      )
    );
  }

  removeItem(id: string) {
    this._items.update(items => 
      items.filter(item => item.id !== id)
    );
  }

  clearCart() {
    this._items.set([]);
  }
}
```

#### 3. 상태 공유 패턴

```typescript
@Injectable({ providedIn: 'root' })
export class AppStateService {
  // 내부 상태 관리
  private _theme = signal<'light' | 'dark'>('light');
  private _language = signal<'ko' | 'en'>('ko');
  private _sidebarOpen = signal(true);
  private _notifications = signal<Notification[]>([]);

  // 읽기 전용 공개 API
  public theme = this._theme.asReadonly();
  public language = this._language.asReadonly();
  public sidebarOpen = this._sidebarOpen.asReadonly();
  public notifications = this._notifications.asReadonly();

  // 복합 상태 (Computed)
  public unreadCount = computed(() =>
    this.notifications().filter(n => !n.read).length
  );

  public config = computed(() => ({
    theme: this.theme(),
    language: this.language(),
    sidebarOpen: this.sidebarOpen()
  }));

  // 상태 변경 메서드
  toggleTheme() {
    this._theme.update(theme => 
      theme === 'light' ? 'dark' : 'light'
    );
  }

  setLanguage(lang: 'ko' | 'en') {
    this._language.set(lang);
  }

  toggleSidebar() {
    this._sidebarOpen.update(open => !open);
  }

  addNotification(notification: Notification) {
    this._notifications.update(notifications => 
      [...notifications, notification]
    );
  }

  markAsRead(id: string) {
    this._notifications.update(notifications =>
      notifications.map(n =>
        n.id === id ? { ...n, read: true } : n
      )
    );
  }
}
```

### asReadonly vs Computed 비교

| 특징 | asReadonly | Computed | 사용 시나리오 |
|------|------------|----------|--------------|
| **목적** | 기존 Signal을 읽기 전용으로 | 파생된 값 계산 | 상태 보호 vs 상태 파생 |
| **원본과의 관계** | 같은 값 참조 | 새로운 값 생성 | 직접 노출 vs 변환 노출 |
| **성능** | 오버헤드 없음 | 계산 비용 | 단순 보호 vs 복잡한 로직 |
| **사용 예** | `state.asReadonly()` | `computed(() => state() * 2)` | API 노출 vs 비즈니스 로직 |

### asReadonly 사용 시 주의사항

```typescript
// ⚠️ 객체/배열의 경우 내부 값은 여전히 변경 가능
const userSignal = signal({ name: 'John', age: 30 });
const readonlyUser = userSignal.asReadonly();

// Signal 자체는 변경 불가
// readonlyUser.set({ name: 'Jane', age: 25 }); // ❌ Error

// 하지만 객체 내부는 변경 가능 (참조만 보호됨)
const user = readonlyUser();
user.name = 'Jane'; // ⚠️ 가능함!

// 💡 해결 방법: 깊은 불변성을 위해 Computed 사용
const trulyReadonlyUser = computed(() => 
  Object.freeze({ ...userSignal() })
);
```

### 실전 예제: 권한 관리 시스템

```typescript
@Injectable({ providedIn: 'root' })
export class PermissionService {
  // Private 권한 상태
  private _userRoles = signal<string[]>([]);
  private _featureFlags = signal<Record<string, boolean>>({});

  // Public 읽기 전용 접근
  public userRoles = this._userRoles.asReadonly();
  public featureFlags = this._featureFlags.asReadonly();

  // 권한 체크 Computed Signals
  public canEditContent = computed(() => 
    this.userRoles().some(role => 
      ['admin', 'editor'].includes(role)
    )
  );

  public canDeleteContent = computed(() =>
    this.userRoles().includes('admin')
  );

  public isFeatureEnabled = (feature: string) => 
    computed(() => this.featureFlags()[feature] ?? false);

  // 권한 업데이트 (보호된 메서드)
  updateUserRoles(roles: string[]) {
    // 검증 로직
    const validRoles = roles.filter(role => 
      this.isValidRole(role)
    );
    this._userRoles.set(validRoles);
  }

  enableFeature(feature: string) {
    this._featureFlags.update(flags => ({
      ...flags,
      [feature]: true
    }));
  }

  private isValidRole(role: string): boolean {
    const validRoles = ['admin', 'editor', 'viewer'];
    return validRoles.includes(role);
  }
}

// 컴포넌트에서 사용
@Component({
  selector: 'app-content-editor',
  template: `
    <div class="editor">
      <button 
        *ngIf="permissions.canEditContent()"
        (click)="edit()">
        수정
      </button>
      
      <button 
        *ngIf="permissions.canDeleteContent()"
        (click)="delete()">
        삭제
      </button>
      
      <div *ngIf="permissions.isFeatureEnabled('advancedEditor')()">
        <!-- 고급 편집 기능 -->
      </div>
    </div>
  `
})
export class ContentEditorComponent {
  constructor(public permissions: PermissionService) {}
}
```

## 6. 고급 패턴과 활용법

### 동적 의존성 관리

```typescript
const showDetails = signal(false);
const userData = signal({ name: 'John', age: 30 });
const additionalData = signal({ hobby: 'coding' });

// 조건부 의존성
const displayText = computed(() => {
  const user = userData();
  
  if (showDetails()) {
    // showDetails가 true일 때만 additionalData에 의존
    const additional = additionalData();
    return `${user.name} (${user.age}) - ${additional.hobby}`;
  }
  
  return `${user.name}`;
});
```

### Untracked - 의존성 제외

```typescript
import { untracked } from '@angular/core';

const counter = signal(0);
const logs = signal<string[]>([]);

effect(() => {
  const count = counter();
  
  // untracked 내부의 Signal은 의존성에서 제외
  untracked(() => {
    logs.update(l => [...l, `카운터 값: ${count}`]);
  });
  
  console.log(`카운터 업데이트: ${count}`);
});
```

### 커스텀 동등성 비교

```typescript
import _ from 'lodash';

// 깊은 비교를 사용하는 Signal
const complexData = signal(
  { users: [{ id: 1, name: 'John' }] },
  { equal: _.isEqual }
);

// 같은 내용이면 업데이트 발생 안 함
complexData.set({ users: [{ id: 1, name: 'John' }] });
```

## 7. 실전 활용 예제

### 검색 필터 시스템

```typescript
@Component({
  selector: 'app-product-search',
  template: `
    <div class="search-container">
      <input [(ngModel)]="searchTerm" 
             (ngModelChange)="updateSearch($event)"
             placeholder="검색어 입력">
      
      <select (change)="updateCategory($event.target.value)">
        <option value="">전체 카테고리</option>
        <option *ngFor="let cat of categories" [value]="cat">
          {{ cat }}
        </option>
      </select>
      
      <div class="price-range">
        <input type="range" 
               [value]="maxPrice()" 
               (input)="updatePrice(+$event.target.value)"
               min="0" max="1000000" step="10000">
        <span>최대 가격: ₩{{ maxPrice() | number }}</span>
      </div>
      
      <div class="results">
        <h3>검색 결과 ({{ filteredProducts().length }}개)</h3>
        <div *ngFor="let product of filteredProducts()">
          {{ product.name }} - ₩{{ product.price | number }}
        </div>
      </div>
    </div>
  `
})
export class ProductSearchComponent {
  // 원본 데이터
  private allProducts = signal([
    { id: 1, name: '노트북 Pro', category: '전자제품', price: 1500000 },
    { id: 2, name: '무선 마우스', category: '전자제품', price: 50000 },
    { id: 3, name: '책상', category: '가구', price: 200000 },
    // ... 더 많은 제품
  ]);

  // 필터 상태
  searchTerm = signal('');
  selectedCategory = signal('');
  maxPrice = signal(1000000);

  // 필터링된 결과 (Computed)
  filteredProducts = computed(() => {
    const term = this.searchTerm().toLowerCase();
    const category = this.selectedCategory();
    const max = this.maxPrice();

    return this.allProducts().filter(product => {
      const matchesSearch = !term || 
        product.name.toLowerCase().includes(term);
      const matchesCategory = !category || 
        product.category === category;
      const matchesPrice = product.price <= max;

      return matchesSearch && matchesCategory && matchesPrice;
    });
  });

  // 카테고리 목록 (Computed)
  categories = computed(() => 
    [...new Set(this.allProducts().map(p => p.category))]
  );

  updateSearch(term: string) {
    this.searchTerm.set(term);
  }

  updateCategory(category: string) {
    this.selectedCategory.set(category);
  }

  updatePrice(price: number) {
    this.maxPrice.set(price);
  }
}
```

### 폼 상태 관리

```typescript
@Component({
  selector: 'app-registration-form',
  template: `
    <form class="registration-form">
      <input [(ngModel)]="username" 
             (ngModelChange)="updateUsername($event)"
             placeholder="사용자명 (3-20자)">
      <div *ngIf="usernameError()" class="error">
        {{ usernameError() }}
      </div>

      <input type="email" 
             [(ngModel)]="email"
             (ngModelChange)="updateEmail($event)" 
             placeholder="이메일">
      <div *ngIf="emailError()" class="error">
        {{ emailError() }}
      </div>

      <input type="password" 
             [(ngModel)]="password"
             (ngModelChange)="updatePassword($event)" 
             placeholder="비밀번호 (8자 이상)">
      <div *ngIf="passwordError()" class="error">
        {{ passwordError() }}
      </div>

      <button [disabled]="!isFormValid()" 
              (click)="submit()">
        가입하기
      </button>

      <div class="form-status">
        <p>폼 상태: {{ formStatus() }}</p>
      </div>
    </form>
  `
})
export class RegistrationFormComponent {
  // 폼 필드
  username = signal('');
  email = signal('');
  password = signal('');

  // 터치 상태
  usernameTouched = signal(false);
  emailTouched = signal(false);
  passwordTouched = signal(false);

  // 유효성 검사 (Computed)
  usernameError = computed(() => {
    if (!this.usernameTouched()) return '';
    
    const value = this.username();
    if (!value) return '사용자명은 필수입니다';
    if (value.length < 3) return '최소 3자 이상';
    if (value.length > 20) return '최대 20자 이하';
    return '';
  });

  emailError = computed(() => {
    if (!this.emailTouched()) return '';
    
    const value = this.email();
    if (!value) return '이메일은 필수입니다';
    if (!value.includes('@')) return '올바른 이메일 형식이 아닙니다';
    return '';
  });

  passwordError = computed(() => {
    if (!this.passwordTouched()) return '';
    
    const value = this.password();
    if (!value) return '비밀번호는 필수입니다';
    if (value.length < 8) return '최소 8자 이상';
    return '';
  });

  // 전체 폼 유효성 (Computed)
  isFormValid = computed(() => 
    !this.usernameError() && 
    !this.emailError() && 
    !this.passwordError() &&
    this.username() && 
    this.email() && 
    this.password()
  );

  // 폼 상태 표시 (Computed)
  formStatus = computed(() => {
    if (!this.usernameTouched() && 
        !this.emailTouched() && 
        !this.passwordTouched()) {
      return '폼을 작성해주세요';
    }
    
    if (this.isFormValid()) {
      return '✅ 모든 필드가 올바르게 입력되었습니다';
    }
    
    return '❌ 입력값을 확인해주세요';
  });

  updateUsername(value: string) {
    this.username.set(value);
    this.usernameTouched.set(true);
  }

  updateEmail(value: string) {
    this.email.set(value);
    this.emailTouched.set(true);
  }

  updatePassword(value: string) {
    this.password.set(value);
    this.passwordTouched.set(true);
  }

  submit() {
    if (this.isFormValid()) {
      console.log('폼 제출:', {
        username: this.username(),
        email: this.email(),
        password: this.password()
      });
    }
  }
}
```

## 8. 성능 최적화와 모범 사례

### Signal vs RxJS 성능 비교

| 측면 | Signal | RxJS Observable | 승자 |
|------|--------|-----------------|------|
| **메모리 사용** | 적음 | 많음 | Signal ✅ |
| **구독 관리** | 자동 | 수동 | Signal ✅ |
| **초기 학습** | 쉬움 | 어려움 | Signal ✅ |
| **복잡한 스트림** | 제한적 | 강력함 | RxJS ✅ |
| **번들 크기** | 작음 | 큼 | Signal ✅ |

### 모범 사례

```typescript
// ✅ 좋은 예: 세밀한 Signal 분리
const firstName = signal('John');
const lastName = signal('Doe');
const fullName = computed(() => `${firstName()} ${lastName()}`);

// ❌ 나쁜 예: 큰 객체를 하나의 Signal로
const user = signal({
  firstName: 'John',
  lastName: 'Doe',
  age: 30,
  address: { /* ... */ }
});

// ✅ 좋은 예: 필요한 경우에만 Effect 사용
constructor() {
  // API 호출이나 외부 시스템 연동 시
  effect(() => {
    const data = this.importantData();
    this.apiService.save(data);
  });
}

// ❌ 나쁜 예: 단순 계산에 Effect 사용
constructor() {
  effect(() => {
    // Computed를 사용해야 함
    this.total = this.price() * this.quantity();
  });
}
```

## 9. 마이그레이션 가이드

### 기존 코드에서 Signal로 전환

```typescript
// Before: 전통적인 방식
export class OldComponent {
  count = 0;
  doubleCount = 0;

  increment() {
    this.count++;
    this.doubleCount = this.count * 2;
  }
}

// After: Signal 방식
export class NewComponent {
  count = signal(0);
  doubleCount = computed(() => this.count() * 2);

  increment() {
    this.count.update(v => v + 1);
  }
}
```

### RxJS에서 Signal로

```typescript
// Before: RxJS
export class RxjsComponent {
  private searchSubject = new Subject<string>();
  results$ = this.searchSubject.pipe(
    debounceTime(300),
    switchMap(term => this.api.search(term))
  );

  search(term: string) {
    this.searchSubject.next(term);
  }
}

// After: Signal + RxJS Interop
import { toObservable, toSignal } from '@angular/core/rxjs-interop';

export class SignalComponent {
  searchTerm = signal('');
  
  private searchTerm$ = toObservable(this.searchTerm).pipe(
    debounceTime(300)
  );
  
  results = toSignal(
    this.searchTerm$.pipe(
      switchMap(term => this.api.search(term))
    ),
    { initialValue: [] }
  );

  search(term: string) {
    this.searchTerm.set(term);
  }
}
```

## 10. 디버깅과 개발 도구

### Signal 디버깅 팁

```typescript
// 디버깅을 위한 Effect
effect(() => {
  console.log('=== Signal 상태 ===');
  console.log('Count:', this.count());
  console.log('Items:', this.items());
  console.log('Computed Total:', this.total());
  console.log('==================');
});

// 개발 모드에서만 실행
if (!environment.production) {
  effect(() => {
    // 상태 변화 추적
    console.trace('Signal 변경 감지');
  });
}
```

## 마무리

Angular Signal과 Computed는 반응형 프로그래밍을 더 쉽고 효율적으로 만들어줍니다. RxJS의 강력함은 유지하면서도 학습 곡선을 크게 낮춰, 개발자들이 더 직관적으로 상태를 관리할 수 있게 되었습니다.

### 핵심 포인트

```mermaid
mindmap
  root((Angular Signals))
    기본 개념
      Signal (읽기/쓰기)
      Computed (파생 상태)
      Effect (부수 효과)
    주요 장점
      간단한 API
      자동 의존성 추적
      효율적인 업데이트
      메모리 안전
    활용 분야
      상태 관리
      폼 처리
      반응형 UI
      성능 최적화
    모범 사례
      세밀한 Signal
      Computed 활용
      Effect 최소화
      타입 안전성
```

Signal은 Angular 개발의 미래입니다. 지금부터 프로젝트에 도입해보시면, 더 깔끔하고 유지보수하기 쉬운 코드를 작성할 수 있을 것입니다.

## 참고 자료

- [Angular 공식 Signal 가이드](https://angular.dev/guide/signals)
- [Angular Signal API 문서](https://angular.dev/api/core/signal)
- [RxJS Interop 가이드](https://angular.dev/guide/signals/rxjs-interop)

---

*Angular Signal에 대한 질문이나 경험을 댓글로 공유해주세요!*