<script setup lang="ts">
import { computed, onMounted, ref } from "vue";

type Product = {
  id: string;
  name: string;
  description?: string;
  unit: string;
  price: number;
  image_url?: string;
};

type PublicStoreResponse = {
  store: { name: string; slug: string; logo_url?: string };
  products: Product[];
};

const apiBaseUrl = (import.meta.env.VITE_API_BASE_URL?.trim() || "").replace(/\/+$/, "");
const storeName = ref("");
const storeLogoUrl = ref("");
const products = ref<Product[]>([]);
const search = ref("");
const loading = ref(true);
const errorMessage = ref("");
const reloading = ref(false);
const storeSlug = decodeURIComponent(window.location.pathname.split("/").filter(Boolean)[0] || "");

const filteredProducts = computed(() => {
  const term = search.value.trim().toLocaleLowerCase();
  if (!term) return products.value;
  return products.value.filter((product) => product.name.toLocaleLowerCase().includes(term));
});

const currency = new Intl.NumberFormat("id-ID", {
  style: "currency",
  currency: "IDR",
  maximumFractionDigits: 0,
});

function displayPrice(product: Product) {
  return currency.format(product.price);
}

function productInitial(name: string) {
  return name.trim().charAt(0).toLocaleUpperCase() || "P";
}

function resolveImageUrl(value?: string) {
  if (!value) return "";
  return new URL(value, apiBaseUrl || window.location.origin).toString();
}

async function loadProducts() {
  if (!storeSlug) {
    errorMessage.value = "Tautan toko tidak valid.";
    loading.value = false;
    return;
  }

  errorMessage.value = "";
  reloading.value = true;
  try {
    const collected: Product[] = [];
    let store: PublicStoreResponse["store"] | undefined;
    let offset = 0;
    let page: Product[];

    do {
      const response = await fetch(`${apiBaseUrl}/public/stores/${encodeURIComponent(storeSlug)}/products?limit=100&offset=${offset}`);
      if (!response.ok) {
        const body = (await response.json().catch(() => ({}))) as { error?: string };
        if (response.status === 404) throw new Error("Toko tidak ditemukan atau tautannya sudah tidak berlaku.");
        if (response.status === 409) throw new Error("Tautan toko tidak unik. Minta pemilik toko memperbarui nama toko.");
        throw new Error(body.error || "Daftar produk tidak dapat dimuat. Coba lagi.");
      }

      const data = (await response.json()) as PublicStoreResponse;
      store = data.store;
      page = data.products;
      collected.push(...page);
      offset += page.length;
    } while (page.length === 100);

    storeName.value = store?.name || "Toko Lumi";
    storeLogoUrl.value = resolveImageUrl(store?.logo_url);
    products.value = collected;
    document.title = `${storeName.value}`;
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : "Terjadi kesalahan saat memuat produk.";
  } finally {
    loading.value = false;
    reloading.value = false;
  }
}

onMounted(() => {
  void loadProducts();
});
</script>

<template>
  <main class="page-shell">
    <header class="topbar">
      <div class="brand" :aria-label="storeName || 'Toko'">
        <img v-if="storeLogoUrl" class="store-logo" :src="storeLogoUrl" :alt="`Logo ${storeName}`" @error="storeLogoUrl = ''" />
        <span v-else class="brand-mark">{{ productInitial(storeName) }}</span>
        <span class="store-brand-name">{{ storeName || "Toko" }}</span>
      </div>
      <span class="public-badge">Powered by <strong>Lumi Assistant</strong></span>
    </header>

    <div class="page-scroll">
      <section class="hero">
        <div class="hero-copy">
          <p class="eyebrow">ETALASE PRODUK</p>
          <h1>{{ storeName || "Temukan produk pilihan" }}</h1>
          <p class="hero-description">Lihat koleksi produk dan harga terbaru dari toko ini.</p>
        </div>
        <div class="hero-art" aria-hidden="true">
          <span class="art-orbit orbit-one"></span>
          <span class="art-orbit orbit-two"></span>
          <span class="art-star star-one">✦</span>
          <span class="art-star star-two">✧</span>
          <div class="art-card"><span class="art-card-icon">✳</span><span class="art-card-line"></span><span class="art-card-price"></span></div>
        </div>
      </section>

      <section class="catalog">
        <div class="catalog-heading">
          <div>
            <p class="eyebrow dark-eyebrow">KATALOG TOKO</p>
            <h2>Produk untuk Anda</h2>
          </div>
          <span v-if="!loading && !errorMessage" class="product-count">{{ filteredProducts.length }} produk</span>
        </div>

        <label class="search-box">
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <circle cx="11" cy="11" r="7"></circle>
            <path d="m16 16 4 4"></path>
          </svg>
          <input v-model="search" type="search" placeholder="Cari nama produk..." aria-label="Cari nama produk" />
          <button v-if="search" class="clear-search" type="button" aria-label="Hapus pencarian" @click="search = ''">×</button>
        </label>

        <div v-if="loading" class="product-grid" aria-label="Memuat produk">
          <article v-for="item in 8" :key="item" class="product-card skeleton-card">
            <div class="skeleton-image"></div>
            <div class="skeleton-line skeleton-title"></div>
            <div class="skeleton-line skeleton-price"></div>
          </article>
        </div>

        <div v-else-if="errorMessage" class="message-panel error-panel">
          <div class="message-icon">!</div>
          <h3>Belum bisa memuat katalog</h3>
          <p>{{ errorMessage }}</p>
          <button class="primary-button" type="button" :disabled="reloading" @click="loadProducts">
            {{ reloading ? "Memuat..." : "Coba lagi" }}
          </button>
        </div>

        <div v-else-if="filteredProducts.length" class="product-grid">
          <article v-for="(product, index) in filteredProducts" :key="product.id" class="product-card" :style="{ '--card-index': index }">
            <div class="product-art" :class="`tone-${index % 6}`">
              <img
                v-if="product.image_url"
                class="product-photo"
                :src="resolveImageUrl(product.image_url)"
                :alt="product.name"
                loading="lazy"
                @error="product.image_url = undefined"
              />
              <span v-else class="product-initial">{{ productInitial(product.name) }}</span>
            </div>
            <div class="product-information">
              <p class="product-category">{{ product.unit || "PRODUK" }}</p>
              <h3>{{ product.name }}</h3>
              <p v-if="product.description" class="product-description">{{ product.description }}</p>
              <div class="product-footer">
                <strong>{{ displayPrice(product) }}</strong>
              </div>
            </div>
          </article>
        </div>

        <div v-else class="message-panel">
          <div class="message-icon">⌕</div>
          <h3>{{ search ? "Produk tidak ditemukan" : "Katalog sedang disiapkan" }}</h3>
          <p>{{ search ? "Coba kata kunci lain untuk menemukan produk." : "Belum ada produk yang ditampilkan di toko ini." }}</p>
          <button v-if="search" class="text-button" type="button" @click="search = ''">Hapus pencarian</button>
        </div>
      </section>

      <footer class="footer">
        <span>Ditampilkan dengan <strong>Lumi Assistant</strong></span>
      </footer>
    </div>
  </main>
</template>
