<template>
  <div class="shop-container">
    <header class="shop-header">
      <h1>🛒 Vue Shop</h1>
      <p>Лучшие товары для настоящих разработчиков</p>
    </header>

    <div class="main-content">
      <!-- Каталог -->
      <main class="catalog">
        <div class="controls">
          <input 
            type="text" 
            v-model="searchQuery" 
            placeholder="🔍 Поиск товаров..." 
            class="search-input"
          />
          <select v-model="sortOption" class="sort-select">
            <option value="default">Сортировка: По умолчанию</option>
            <option value="asc">Сначала дешевые</option>
            <option value="desc">Сначала дорогие</option>
          </select>
        </div>

        <div class="products-grid">
          <div v-for="product in filteredProducts" :key="product.id" class="product-card">
            <div class="product-image">{{ product.image }}</div>
            <div class="product-info">
              <div class="product-category">{{ product.category }}</div>
              <div class="product-name">{{ product.name }}</div>
              <div class="product-price">
                <span v-if="promoApplied" class="old-price">{{ formatPrice(product.price) }}</span>
                {{ formatPrice(getDiscountedPrice(product.price)) }}
              </div>
              <button class="btn btn-primary" @click="addToCart(product)">
                В корзину
              </button>
            </div>
          </div>
          
          <div v-if="filteredProducts.length === 0" class="empty-state">
            <h3>Товары не найдены 😔</h3>
            <p>Попробуйте изменить поисковый запрос.</p>
          </div>
        </div>
      </main>

      <!-- Корзина -->
      <aside class="cart-panel">
        <div class="cart-header">
          <h2>Корзина <span v-if="cartCount > 0" class="cart-badge">{{ cartCount }}</span></h2>
        </div>

        <div class="cart-items">
          <div v-if="cart.length === 0" class="cart-empty">
            Корзина пуста. Добавьте что-нибудь! 🛍️
          </div>
          
          <div v-for="item in cart" :key="item.id" class="cart-item">
            <div class="cart-item-emoji">{{ item.image }}</div>
            <div class="cart-item-details">
              <div class="cart-item-name">{{ item.name }}</div>
              <div class="cart-item-price">
                <span v-if="promoApplied" class="old-price-small">{{ formatPrice(item.price) }}</span>
                {{ formatPrice(getDiscountedPrice(item.price)) }} × {{ item.quantity }}
              </div>
            </div>
            <div class="cart-item-controls">
              <button class="qty-btn" @click="changeQuantity(item.id, -1)">−</button>
              <span class="qty-value">{{ item.quantity }}</span>
              <button class="qty-btn" @click="changeQuantity(item.id, 1)">+</button>
              <button class="remove-btn" @click="removeFromCart(item.id)" title="Удалить">✕</button>
            </div>
          </div>
        </div>

        <div class="cart-footer" v-if="cart.length > 0">
          <div class="promo-section">
            <input 
              type="text" 
              v-model="promoInput" 
              placeholder="Промокод" 
              :disabled="promoApplied"
              @keyup.enter="applyPromo"
              class="promo-input"
            />
            <button v-if="!promoApplied" class="btn btn-outline" @click="applyPromo">ОК</button>
            <button v-else class="btn btn-outline" @click="cancelPromo">✕</button>
          </div>
          
          <div v-if="promoMessage" :class="['promo-msg', promoApplied ? 'success' : 'error']">
            {{ promoMessage }}
          </div>

          <div class="total-row">
            <span>Товары ({{ cartCount }}):</span>
            <span>{{ formatPrice(subtotal) }}</span>
          </div>
          <div v-if="promoApplied" class="total-row discount">
            <span>Скидка WEB (10%):</span>
            <span>- {{ formatPrice(subtotal - total) }}</span>
          </div>
          <div class="total-row final">
            <span>Итого:</span>
            <span>{{ formatPrice(total) }}</span>
          </div>

          <div class="cart-actions">
            <button class="btn btn-danger" @click="clearCart">Очистить корзину</button>
          </div>
        </div>
      </aside>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue';

// --- Interfaces ---
interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
  image: string;
}

interface CartItem extends Product {
  quantity: number;
}

// --- Data ---
const products: Product[] = [
  { id: 1, name: 'Ноутбук Pro 14', price: 89990, category: 'Электроника', image: '💻' },
  { id: 2, name: 'Механическая клавиатура', price: 7500, category: 'Электроника', image: '⌨️' },
  { id: 3, name: 'Кружка разработчика', price: 690, category: 'Аксессуары', image: '☕' },
  { id: 4, name: 'Худи "Vue Master"', price: 3200, category: 'Одежда', image: '👕' },
  { id: 5, name: 'Мышь беспроводная', price: 2400, category: 'Электроника', image: '🖱️' },
  { id: 6, name: 'Стикерпак с логотипом', price: 150, category: 'Аксессуары', image: '🏷️' },
  { id: 7, name: 'Монитор 27"', price: 24990, category: 'Электроника', image: '🖥️' },
  { id: 8, name: 'Кепка "Frontend"', price: 1100, category: 'Одежда', image: '🧢' },
];

// --- State ---
const cart = ref<CartItem[]>([]);
const searchQuery = ref<string>('');
const sortOption = ref<string>('default');
const promoInput = ref<string>('');
const promoApplied = ref<boolean>(false);
const promoMessage = ref<string>('');

// --- Computed ---
const filteredProducts = computed<Product[]>(() => {
  let result = products.filter(p => 
    p.name.toLowerCase().includes(searchQuery.value.toLowerCase())
  );

  if (sortOption.value === 'asc') {
    result.sort((a, b) => a.price - b.price);
  } else if (sortOption.value === 'desc') {
    result.sort((a, b) => b.price - a.price);
  }

  return result;
});

const cartCount = computed<number>(() => {
  return cart.value.reduce((sum, item) => sum + item.quantity, 0);
});

const subtotal = computed<number>(() => {
  return cart.value.reduce((sum, item) => sum + (item.price * item.quantity), 0);
});

const total = computed<number>(() => {
  return cart.value.reduce((sum, item) => {
    return sum + (getDiscountedPrice(item.price) * item.quantity);
  }, 0);
});

// --- Methods ---
const formatPrice = (price: number): string => {
  return new Intl.NumberFormat('ru-RU', {
    style: 'currency',
    currency: 'RUB',
    minimumFractionDigits: 0,
    maximumFractionDigits: 0
  }).format(price);
};

const getDiscountedPrice = (price: number): number => {
  return promoApplied.value ? Math.round(price * 0.9) : price;
};

const addToCart = (product: Product): void => {
  const existingItem = cart.value.find(item => item.id === product.id);
  if (existingItem) {
    existingItem.quantity++;
  } else {
    cart.value.push({ ...product, quantity: 1 });
  }
};

const removeFromCart = (productId: number): void => {
  cart.value = cart.value.filter(item => item.id !== productId);
};

const changeQuantity = (productId: number, delta: number): void => {
  const item = cart.value.find(i => i.id === productId);
  if (item) {
    item.quantity += delta;
    if (item.quantity <= 0) {
      removeFromCart(productId);
    }
  }
};

const clearCart = (): void => {
  cart.value = [];
};

const applyPromo = (): void => {
  if (promoInput.value.trim().toUpperCase() === 'WEB') {
    promoApplied.value = true;
    promoMessage.value = '✅ Промокод WEB успешно применен! Скидка 10%.';
  } else {
    promoApplied.value = false;
    promoMessage.value = '❌ Неверный промокод. Попробуйте еще раз.';
  }
};

const cancelPromo = (): void => {
  promoApplied.value = false;
  promoInput.value = '';
  promoMessage.value = '';
};

// --- Lifecycle & Watchers (LocalStorage) ---
onMounted(() => {
  const savedCart = localStorage.getItem('vue_shop_cart_ts');
  if (savedCart) {
    try {
      cart.value = JSON.parse(savedCart) as CartItem[];
    } catch (e) {
      console.error('Error parsing cart from localStorage', e);
    }
  }
});

watch(cart, (newCart) => {
  localStorage.setItem('vue_shop_cart_ts', JSON.stringify(newCart));
}, { deep: true });
</script>

