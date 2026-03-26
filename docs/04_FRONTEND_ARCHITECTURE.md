# 04. Frontend Architecture
## Enterprise Men's Clothing Manufacturing & Sales Platform

**Version:** 2.0  
**Last Updated:** March 26, 2026  
**Stack:** React.js + TypeScript + Vite  
**Status:** ✅ Production-Ready

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Project Structure](#project-structure)
3. [State Management](#state-management)
4. [Component Architecture](#component-architecture)
5. [Routing Strategy](#routing-strategy)
6. [API Integration](#api-integration)
7. [Performance Optimization](#performance-optimization)
8. [UI/UX Patterns](#uiux-patterns)

---

## 1. Architecture Overview

### 1.1 Architecture Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                      Component Layer                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Smart Components (Containers)                     │  │
│  │  • Business Logic  • State Management  • API Calls       │  │
│  └────────────────────────┬─────────────────────────────────┘  │
│                           │                                      │
│  ┌────────────────────────▼─────────────────────────────────┐  │
│  │         Presentational Components (UI)                    │  │
│  │  • Pure Functions  • Props  • Stateless                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      State Management Layer                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Global State (Redux Toolkit)                      │  │
│  │  • User State  • App State  • UI State                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Server State (React Query)                        │  │
│  │  • Caching  • Invalidation  • Optimistic Updates         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Service Layer                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         API Client (Axios)                                │  │
│  │  • HTTP Requests  • Interceptors  • Error Handling       │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Backend API                                 │
│                  (Express.js REST API)                           │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Design Principles

```yaml
Component Design:
  - Atomic Design Pattern
  - Container/Presentational Pattern
  - Composition over Inheritance
  - Single Responsibility
  - Props Immutability

State Management:
  - Single Source of Truth
  - State is Read-Only
  - Changes Made with Pure Functions
  - Normalized State Shape

Code Quality:
  - TypeScript Strict Mode
  - ESLint + Prettier
  - Component Testing
  - Accessibility (WCAG 2.1)
```

---

## 2. Project Structure

### 2.1 Directory Structure

```
frontend/
├── public/                      # Static assets
│   ├── images/                 # Public images
│   ├── fonts/                  # Custom fonts
│   └── favicon.ico
│
├── src/
│   ├── app/                    # Application setup
│   │   ├── App.tsx            # Root component
│   │   ├── AppRoutes.tsx      # Route configuration
│   │   └── AppProviders.tsx   # Context providers
│   │
│   ├── features/              # Feature modules
│   │   ├── auth/              # Authentication
│   │   │   ├── components/    # Feature-specific components
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   ├── RegisterForm.tsx
│   │   │   │   └── ForgotPasswordForm.tsx
│   │   │   ├── hooks/         # Custom hooks
│   │   │   │   ├── useAuth.ts
│   │   │   │   └── useLogin.ts
│   │   │   ├── pages/         # Route pages
│   │   │   │   ├── LoginPage.tsx
│   │   │   │   ├── RegisterPage.tsx
│   │   │   │   └── ForgotPasswordPage.tsx
│   │   │   ├── store/         # Redux slices
│   │   │   │   └── authSlice.ts
│   │   │   ├── api/           # API calls
│   │   │   │   └── authApi.ts
│   │   │   ├── types/         # TypeScript types
│   │   │   │   └── auth.types.ts
│   │   │   └── index.ts       # Exports
│   │   │
│   │   ├── products/          # Product management
│   │   │   ├── components/
│   │   │   │   ├── ProductCard.tsx
│   │   │   │   ├── ProductList.tsx
│   │   │   │   ├── ProductGrid.tsx
│   │   │   │   ├── ProductFilters.tsx
│   │   │   │   ├── ProductForm.tsx
│   │   │   │   └── ProductImageGallery.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useProducts.ts
│   │   │   │   ├── useProductDetails.ts
│   │   │   │   └── useProductFilters.ts
│   │   │   ├── pages/
│   │   │   │   ├── ProductListPage.tsx
│   │   │   │   ├── ProductDetailPage.tsx
│   │   │   │   ├── ProductCreatePage.tsx
│   │   │   │   └── ProductEditPage.tsx
│   │   │   ├── store/
│   │   │   │   └── productSlice.ts
│   │   │   ├── api/
│   │   │   │   └── productApi.ts
│   │   │   └── types/
│   │   │       └── product.types.ts
│   │   │
│   │   ├── orders/            # Order management
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   ├── store/
│   │   │   ├── api/
│   │   │   └── types/
│   │   │
│   │   ├── inventory/         # Inventory management
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   ├── store/
│   │   │   ├── api/
│   │   │   └── types/
│   │   │
│   │   ├── manufacturing/     # Manufacturing
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   ├── store/
│   │   │   ├── api/
│   │   │   └── types/
│   │   │
│   │   ├── analytics/         # Analytics & Reports
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   ├── store/
│   │   │   ├── api/
│   │   │   └── types/
│   │   │
│   │   └── dashboard/         # Dashboard
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── pages/
│   │       └── widgets/
│   │
│   ├── components/            # Shared components
│   │   ├── ui/               # Base UI components
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Select.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Badge.tsx
│   │   │   ├── Table.tsx
│   │   │   ├── Pagination.tsx
│   │   │   ├── Spinner.tsx
│   │   │   └── index.ts
│   │   │
│   │   ├── forms/            # Form components
│   │   │   ├── FormInput.tsx
│   │   │   ├── FormSelect.tsx
│   │   │   ├── FormTextarea.tsx
│   │   │   ├── FormCheckbox.tsx
│   │   │   ├── FormRadio.tsx
│   │   │   ├── FormDatePicker.tsx
│   │   │   ├── FormFileUpload.tsx
│   │   │   └── index.ts
│   │   │
│   │   ├── layout/           # Layout components
│   │   │   ├── Header.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Footer.tsx
│   │   │   ├── MainLayout.tsx
│   │   │   ├── AuthLayout.tsx
│   │   │   └── index.ts
│   │   │
│   │   ├── navigation/       # Navigation components
│   │   │   ├── Navbar.tsx
│   │   │   ├── Breadcrumb.tsx
│   │   │   ├── Tabs.tsx
│   │   │   └── index.ts
│   │   │
│   │   └── feedback/         # Feedback components
│   │       ├── Toast.tsx
│   │       ├── Alert.tsx
│   │       ├── Notification.tsx
│   │       ├── EmptyState.tsx
│   │       ├── ErrorBoundary.tsx
│   │       └── index.ts
│   │
│   ├── hooks/                # Global custom hooks
│   │   ├── useDebounce.ts
│   │   ├── useLocalStorage.ts
│   │   ├── useOnClickOutside.ts
│   │   ├── usePagination.ts
│   │   ├── useInfiniteScroll.ts
│   │   ├── useMediaQuery.ts
│   │   └── index.ts
│   │
│   ├── services/             # Service layer
│   │   ├── api/              # API client
│   │   │   ├── client.ts     # Axios instance
│   │   │   ├── interceptors.ts
│   │   │   └── endpoints.ts
│   │   ├── storage/          # Local/Session storage
│   │   │   └── storage.service.ts
│   │   ├── auth/             # Auth service
│   │   │   └── auth.service.ts
│   │   └── websocket/        # WebSocket service
│   │       └── websocket.service.ts
│   │
│   ├── store/                # Redux store
│   │   ├── index.ts          # Store configuration
│   │   ├── rootReducer.ts    # Root reducer
│   │   └── slices/           # Global slices
│   │       ├── appSlice.ts
│   │       ├── uiSlice.ts
│   │       └── index.ts
│   │
│   ├── lib/                  # Third-party lib configs
│   │   ├── react-query.ts    # React Query config
│   │   ├── axios.ts          # Axios config
│   │   └── i18n.ts           # i18n config
│   │
│   ├── utils/                # Utility functions
│   │   ├── formatters.ts     # Format functions
│   │   ├── validators.ts     # Validation functions
│   │   ├── helpers.ts        # Helper functions
│   │   ├── constants.ts      # Constants
│   │   └── index.ts
│   │
│   ├── types/                # Global TypeScript types
│   │   ├── common.types.ts
│   │   ├── api.types.ts
│   │   └── index.ts
│   │
│   ├── styles/               # Global styles
│   │   ├── index.css         # Main styles
│   │   ├── tailwind.css      # Tailwind imports
│   │   └── variables.css     # CSS variables
│   │
│   ├── assets/               # Static assets
│   │   ├── images/
│   │   ├── icons/
│   │   └── fonts/
│   │
│   ├── config/               # Configuration
│   │   ├── env.ts            # Environment variables
│   │   ├── routes.ts         # Route paths
│   │   └── constants.ts      # App constants
│   │
│   ├── main.tsx              # Entry point
│   └── vite-env.d.ts         # Vite types
│
├── tests/                    # Test files
│   ├── unit/                 # Unit tests
│   ├── integration/          # Integration tests
│   ├── e2e/                  # E2E tests
│   └── setup.ts              # Test setup
│
├── .env.example              # Environment template
├── .env                      # Environment variables
├── .eslintrc.cjs            # ESLint config
├── .prettierrc              # Prettier config
├── tsconfig.json            # TypeScript config
├── tailwind.config.js       # Tailwind config
├── vite.config.ts           # Vite config
├── package.json             # Dependencies
└── README.md                # Documentation
```

---

## 3. State Management

### 3.1 Redux Toolkit Setup

```typescript
// src/store/index.ts

import { configureStore } from '@reduxjs/toolkit';
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import rootReducer from './rootReducer';

export const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST'],
      },
    }),
  devTools: process.env.NODE_ENV !== 'production',
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Typed hooks
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

```typescript
// src/store/rootReducer.ts

import { combineReducers } from '@reduxjs/toolkit';
import authReducer from '../features/auth/store/authSlice';
import productReducer from '../features/products/store/productSlice';
import cartReducer from '../features/cart/store/cartSlice';
import uiReducer from './slices/uiSlice';
import appReducer from './slices/appSlice';

const rootReducer = combineReducers({
  auth: authReducer,
  products: productReducer,
  cart: cartReducer,
  ui: uiReducer,
  app: appReducer,
});

export default rootReducer;
```

```typescript
// src/features/auth/store/authSlice.ts

import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import type { User } from '../types/auth.types';

interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  error: string | null;
}

const initialState: AuthState = {
  user: null,
  accessToken: localStorage.getItem('accessToken'),
  refreshToken: localStorage.getItem('refreshToken'),
  isAuthenticated: false,
  isLoading: false,
  error: null,
};

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    loginStart: (state) => {
      state.isLoading = true;
      state.error = null;
    },
    loginSuccess: (state, action: PayloadAction<{ user: User; accessToken: string; refreshToken: string }>) => {
      state.isLoading = false;
      state.isAuthenticated = true;
      state.user = action.payload.user;
      state.accessToken = action.payload.accessToken;
      state.refreshToken = action.payload.refreshToken;
      state.error = null;

      // Store tokens in localStorage
      localStorage.setItem('accessToken', action.payload.accessToken);
      localStorage.setItem('refreshToken', action.payload.refreshToken);
    },
    loginFailure: (state, action: PayloadAction<string>) => {
      state.isLoading = false;
      state.error = action.payload;
    },
    logout: (state) => {
      state.user = null;
      state.accessToken = null;
      state.refreshToken = null;
      state.isAuthenticated = false;
      state.error = null;

      // Clear localStorage
      localStorage.removeItem('accessToken');
      localStorage.removeItem('refreshToken');
    },
    updateUser: (state, action: PayloadAction<Partial<User>>) => {
      if (state.user) {
        state.user = { ...state.user, ...action.payload };
      }
    },
  },
});

export const { loginStart, loginSuccess, loginFailure, logout, updateUser } = authSlice.actions;
export default authSlice.reducer;
```

### 3.2 React Query Setup

```typescript
// src/lib/react-query.ts

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
      refetchOnWindowFocus: false,
      retry: 1,
    },
  },
});

// Query key factory
export const queryKeys = {
  products: {
    all: ['products'] as const,
    lists: () => [...queryKeys.products.all, 'list'] as const,
    list: (filters: string) => [...queryKeys.products.lists(), { filters }] as const,
    details: () => [...queryKeys.products.all, 'detail'] as const,
    detail: (id: number) => [...queryKeys.products.details(), id] as const,
  },
  orders: {
    all: ['orders'] as const,
    lists: () => [...queryKeys.orders.all, 'list'] as const,
    list: (filters: string) => [...queryKeys.orders.lists(), { filters }] as const,
    details: () => [...queryKeys.orders.all, 'detail'] as const,
    detail: (id: number) => [...queryKeys.orders.details(), id] as const,
  },
  // Add more as needed
};
```

```typescript
// src/features/products/hooks/useProducts.ts

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { productApi } from '../api/productApi';
import { queryKeys } from '../../../lib/react-query';
import type { ProductFilters, Product } from '../types/product.types';

export function useProducts(filters: ProductFilters) {
  return useQuery({
    queryKey: queryKeys.products.list(JSON.stringify(filters)),
    queryFn: () => productApi.getProducts(filters),
    keepPreviousData: true,
  });
}

export function useProduct(id: number) {
  return useQuery({
    queryKey: queryKeys.products.detail(id),
    queryFn: () => productApi.getProduct(id),
    enabled: !!id,
  });
}

export function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: productApi.createProduct,
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({ queryKey: queryKeys.products.all });
    },
  });
}

export function useUpdateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: Partial<Product> }) =>
      productApi.updateProduct(id, data),
    onSuccess: (data, variables) => {
      // Update cache
      queryClient.setQueryData(
        queryKeys.products.detail(variables.id),
        data
      );
      // Invalidate list
      queryClient.invalidateQueries({ queryKey: queryKeys.products.lists() });
    },
  });
}

export function useDeleteProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: productApi.deleteProduct,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.products.all });
    },
  });
}
```

---

## 4. Component Architecture

### 4.1 Atomic Design Pattern

```
Atoms (Basic building blocks)
  ├── Button
  ├── Input
  ├── Label
  ├── Icon
  └── Badge

Molecules (Combination of atoms)
  ├── FormField (Label + Input + Error)
  ├── SearchBar (Input + Button + Icon)
  ├── ProductPrice (Badge + Text)
  └── UserAvatar (Image + Badge)

Organisms (Complex UI sections)
  ├── Navigation (Logo + Menu + SearchBar + UserMenu)
  ├── ProductCard (Image + Title + Price + Button)
  ├── OrderTable (Table + Pagination + Filters)
  └── LoginForm (Form + Inputs + Buttons)

Templates (Page layouts)
  ├── DashboardTemplate
  ├── AuthTemplate
  ├── ProductListTemplate
  └── OrderDetailTemplate

Pages (Specific instances)
  ├── HomePage
  ├── LoginPage
  ├── ProductListPage
  └── OrderDetailPage
```

### 4.2 Component Examples

```typescript
// src/components/ui/Button.tsx

import React from 'react';
import { cn } from '../../utils/cn';

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'outline' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  children: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({
  variant = 'primary',
  size = 'md',
  isLoading = false,
  className,
  disabled,
  children,
  ...props
}) => {
  const baseStyles = 'inline-flex items-center justify-center font-medium rounded-lg transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50 disabled:pointer-events-none';

  const variants = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500',
    secondary: 'bg-gray-600 text-white hover:bg-gray-700 focus:ring-gray-500',
    outline: 'border-2 border-blue-600 text-blue-600 hover:bg-blue-50 focus:ring-blue-500',
    ghost: 'text-gray-700 hover:bg-gray-100 focus:ring-gray-500',
    danger: 'bg-red-600 text-white hover:bg-red-700 focus:ring-red-500',
  };

  const sizes = {
    sm: 'text-sm px-3 py-1.5',
    md: 'text-base px-4 py-2',
    lg: 'text-lg px-6 py-3',
  };

  return (
    <button
      className={cn(baseStyles, variants[variant], sizes[size], className)}
      disabled={disabled || isLoading}
      {...props}
    >
      {isLoading && (
        <svg
          className="animate-spin -ml-1 mr-2 h-4 w-4"
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
        >
          <circle
            className="opacity-25"
            cx="12"
            cy="12"
            r="10"
            stroke="currentColor"
            strokeWidth="4"
          />
          <path
            className="opacity-75"
            fill="currentColor"
            d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
          />
        </svg>
      )}
      {children}
    </button>
  );
};
```

```typescript
// src/features/products/components/ProductCard.tsx

import React from 'react';
import { Link } from 'react-router-dom';
import { Button } from '../../../components/ui/Button';
import { Badge } from '../../../components/ui/Badge';
import type { Product } from '../types/product.types';

interface ProductCardProps {
  product: Product;
  onAddToCart?: (productId: number) => void;
}

export const ProductCard: React.FC<ProductCardProps> = ({
  product,
  onAddToCart,
}) => {
  const handleAddToCart = (e: React.MouseEvent) => {
    e.preventDefault();
    onAddToCart?.(product.id);
  };

  return (
    <Link
      to={`/products/${product.id}`}
      className="group block bg-white rounded-lg shadow hover:shadow-lg transition-shadow"
    >
      {/* Image */}
      <div className="relative aspect-square overflow-hidden rounded-t-lg">
        <img
          src={product.images[0]?.url || '/placeholder.png'}
          alt={product.name}
          className="object-cover w-full h-full group-hover:scale-105 transition-transform duration-300"
        />
        
        {/* Badges */}
        <div className="absolute top-2 left-2 flex flex-col gap-1">
          {product.isFeatured && <Badge variant="primary">Featured</Badge>}
          {product.status === 'inactive' && (
            <Badge variant="danger">Out of Stock</Badge>
          )}
        </div>
      </div>

      {/* Content */}
      <div className="p-4">
        {/* Category */}
        <p className="text-sm text-gray-500 mb-1">{product.category?.name}</p>

        {/* Title */}
        <h3 className="text-lg font-semibold text-gray-900 mb-2 line-clamp-2">
          {product.name}
        </h3>

        {/* Price */}
        <div className="flex items-center justify-between mb-3">
          <div>
            <span className="text-xl font-bold text-gray-900">
              ₹{product.variants[0]?.retailPrice}
            </span>
            {product.variants[0]?.compareAtPrice && (
              <span className="text-sm text-gray-500 line-through ml-2">
                ₹{product.variants[0].compareAtPrice}
              </span>
            )}
          </div>
        </div>

        {/* Actions */}
        <Button
          onClick={handleAddToCart}
          className="w-full"
          disabled={product.status === 'inactive'}
        >
          Add to Cart
        </Button>
      </div>
    </Link>
  );
};
```

```typescript
// src/features/products/pages/ProductListPage.tsx

import React, { useState } from 'react';
import { ProductCard } from '../components/ProductCard';
import { ProductFilters } from '../components/ProductFilters';
import { Pagination } from '../../../components/ui/Pagination';
import { Spinner } from '../../../components/ui/Spinner';
import { EmptyState } from '../../../components/feedback/EmptyState';
import { useProducts } from '../hooks/useProducts';
import { useCart } from '../../cart/hooks/useCart';

export const ProductListPage: React.FC = () => {
  const [filters, setFilters] = useState({
    page: 1,
    limit: 20,
    category: '',
    status: '',
    search: '',
  });

  const { data, isLoading, isError, error } = useProducts(filters);
  const { addToCart } = useCart();

  const handleFilterChange = (newFilters: Partial<typeof filters>) => {
    setFilters((prev) => ({ ...prev, ...newFilters, page: 1 }));
  };

  const handlePageChange = (page: number) => {
    setFilters((prev) => ({ ...prev, page }));
  };

  if (isLoading) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <Spinner size="lg" />
      </div>
    );
  }

  if (isError) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="text-center">
          <h2 className="text-2xl font-bold text-gray-900 mb-2">
            Error loading products
          </h2>
          <p className="text-gray-600">{error?.message}</p>
        </div>
      </div>
    );
  }

  return (
    <div className="container mx-auto px-4 py-8">
      {/* Header */}
      <div className="mb-8">
        <h1 className="text-3xl font-bold text-gray-900 mb-2">Products</h1>
        <p className="text-gray-600">
          Browse our collection of premium men's clothing
        </p>
      </div>

      {/* Filters */}
      <ProductFilters filters={filters} onChange={handleFilterChange} />

      {/* Product Grid */}
      {data?.data.length === 0 ? (
        <EmptyState
          title="No products found"
          description="Try adjusting your filters or search criteria"
        />
      ) : (
        <>
          <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6 mb-8">
            {data?.data.map((product) => (
              <ProductCard
                key={product.id}
                product={product}
                onAddToCart={addToCart}
              />
            ))}
          </div>

          {/* Pagination */}
          {data?.pagination && (
            <Pagination
              currentPage={data.pagination.page}
              totalPages={data.pagination.totalPages}
              onPageChange={handlePageChange}
            />
          )}
        </>
      )}
    </div>
  );
};
```

---

## 5. Routing Strategy

### 5.1 Route Configuration

```typescript
// src/app/AppRoutes.tsx

import React from 'react';
import { Routes, Route, Navigate } from 'react-router-dom';
import { MainLayout } from '../components/layout/MainLayout';
import { AuthLayout } from '../components/layout/AuthLayout';
import { ProtectedRoute } from '../components/navigation/ProtectedRoute';
import { RoleBasedRoute } from '../components/navigation/RoleBasedRoute';

// Lazy load pages
const HomePage = React.lazy(() => import('../pages/HomePage'));
const LoginPage = React.lazy(() => import('../features/auth/pages/LoginPage'));
const RegisterPage = React.lazy(() => import('../features/auth/pages/RegisterPage'));
const DashboardPage = React.lazy(() => import('../features/dashboard/pages/DashboardPage'));
const ProductListPage = React.lazy(() => import('../features/products/pages/ProductListPage'));
const ProductDetailPage = React.lazy(() => import('../features/products/pages/ProductDetailPage'));
const ProductCreatePage = React.lazy(() => import('../features/products/pages/ProductCreatePage'));
const OrderListPage = React.lazy(() => import('../features/orders/pages/OrderListPage'));
const OrderDetailPage = React.lazy(() => import('../features/orders/pages/OrderDetailPage'));
const NotFoundPage = React.lazy(() => import('../pages/NotFoundPage'));

export const AppRoutes: React.FC = () => {
  return (
    <Routes>
      {/* Public Routes */}
      <Route path="/" element={<MainLayout />}>
        <Route index element={<HomePage />} />
        <Route path="products" element={<ProductListPage />} />
        <Route path="products/:id" element={<ProductDetailPage />} />
      </Route>

      {/* Auth Routes */}
      <Route element={<AuthLayout />}>
        <Route path="login" element={<LoginPage />} />
        <Route path="register" element={<RegisterPage />} />
      </Route>

      {/* Protected Routes */}
      <Route element={<ProtectedRoute />}>
        <Route path="dashboard" element={<MainLayout />}>
          <Route index element={<DashboardPage />} />
          
          {/* Orders */}
          <Route path="orders" element={<OrderListPage />} />
          <Route path="orders/:id" element={<OrderDetailPage />} />

          {/* Admin Only Routes */}
          <Route element={<RoleBasedRoute roles={['admin', 'staff']} />}>
            <Route path="products/create" element={<ProductCreatePage />} />
            <Route path="products/:id/edit" element={<ProductCreatePage />} />
          </Route>
        </Route>
      </Route>

      {/* 404 */}
      <Route path="*" element={<NotFoundPage />} />
    </Routes>
  );
};
```

```typescript
// src/components/navigation/ProtectedRoute.tsx

import React from 'react';
import { Navigate, Outlet, useLocation } from 'react-router-dom';
import { useAppSelector } from '../../store';

export const ProtectedRoute: React.FC = () => {
  const { isAuthenticated } = useAppSelector((state) => state.auth);
  const location = useLocation();

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
};
```

```typescript
// src/components/navigation/RoleBasedRoute.tsx

import React from 'react';
import { Navigate, Outlet } from 'react-router-dom';
import { useAppSelector } from '../../store';

interface RoleBasedRouteProps {
  roles: string[];
}

export const RoleBasedRoute: React.FC<RoleBasedRouteProps> = ({ roles }) => {
  const { user } = useAppSelector((state) => state.auth);

  const hasRequiredRole = user?.roles?.some((role) => roles.includes(role));

  if (!hasRequiredRole) {
    return <Navigate to="/dashboard" replace />;
  }

  return <Outlet />;
};
```

---

## 6. API Integration

### 6.1 Axios Client

```typescript
// src/services/api/client.ts

import axios, { AxiosInstance, AxiosError, AxiosRequestConfig } from 'axios';
import { config } from '../../config/env';
import { authService } from '../auth/auth.service';

class ApiClient {
  private client: AxiosInstance;

  constructor() {
    this.client = axios.create({
      baseURL: config.apiUrl,
      timeout: 30000,
      headers: {
        'Content-Type': 'application/json',
      },
    });

    this.setupInterceptors();
  }

  private setupInterceptors() {
    // Request interceptor
    this.client.interceptors.request.use(
      (config) => {
        const token = authService.getAccessToken();
        if (token) {
          config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
      },
      (error) => Promise.reject(error)
    );

    // Response interceptor
    this.client.interceptors.response.use(
      (response) => response.data,
      async (error: AxiosError) => {
        const originalRequest = error.config as AxiosRequestConfig & {
          _retry?: boolean;
        };

        // Handle token refresh
        if (error.response?.status === 401 && !originalRequest._retry) {
          originalRequest._retry = true;

          try {
            const newToken = await authService.refreshToken();
            if (originalRequest.headers) {
              originalRequest.headers.Authorization = `Bearer ${newToken}`;
            }
            return this.client(originalRequest);
          } catch (refreshError) {
            authService.logout();
            window.location.href = '/login';
            return Promise.reject(refreshError);
          }
        }

        // Handle other errors
        const errorMessage =
          error.response?.data?.message || error.message || 'An error occurred';

        return Promise.reject({
          message: errorMessage,
          status: error.response?.status,
          data: error.response?.data,
        });
      }
    );
  }

  async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    return this.client.get(url, config);
  }

  async post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    return this.client.post(url, data, config);
  }

  async put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    return this.client.put(url, data, config);
  }

  async patch<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
    return this.client.patch(url, data, config);
  }

  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
    return this.client.delete(url, config);
  }
}

export const apiClient = new ApiClient();
```

### 6.2 Feature API Modules

```typescript
// src/features/products/api/productApi.ts

import { apiClient } from '../../../services/api/client';
import type { Product, ProductFilters, ProductCreateDto } from '../types/product.types';

export const productApi = {
  /**
   * Get products with filters
   */
  getProducts: async (filters: ProductFilters) => {
    const response = await apiClient.get<{
      data: Product[];
      pagination: any;
    }>('/products', { params: filters });
    return response;
  },

  /**
   * Get single product
   */
  getProduct: async (id: number) => {
    const response = await apiClient.get<{ data: Product }>(`/products/${id}`);
    return response.data;
  },

  /**
   * Create product
   */
  createProduct: async (data: ProductCreateDto) => {
    const response = await apiClient.post<{ data: Product }>('/products', data);
    return response.data;
  },

  /**
   * Update product
   */
  updateProduct: async (id: number, data: Partial<Product>) => {
    const response = await apiClient.put<{ data: Product }>(`/products/${id}`, data);
    return response.data;
  },

  /**
   * Delete product
   */
  deleteProduct: async (id: number) => {
    await apiClient.delete(`/products/${id}`);
  },

  /**
   * Upload product images
   */
  uploadImages: async (productId: number, files: File[]) => {
    const formData = new FormData();
    files.forEach((file) => formData.append('images', file));

    const response = await apiClient.post<{ data: string[] }>(
      `/products/${productId}/images`,
      formData,
      {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      }
    );
    return response.data;
  },
};
```

---

## 7. Performance Optimization

### 7.1 Code Splitting & Lazy Loading

```typescript
// Route-based code splitting
const ProductListPage = React.lazy(() => import('../features/products/pages/ProductListPage'));

// Component-based code splitting
const HeavyChart = React.lazy(() => import('../components/charts/HeavyChart'));

// Usage with Suspense
<Suspense fallback={<Spinner />}>
  <ProductListPage />
</Suspense>
```

### 7.2 Memoization

```typescript
import React, { useMemo, useCallback } from 'react';

export const ProductList: React.FC<{ products: Product[] }> = ({ products }) => {
  // Memoize expensive calculations
  const sortedProducts = useMemo(() => {
    return products.sort((a, b) => b.price - a.price);
  }, [products]);

  // Memoize callbacks
  const handleProductClick = useCallback((id: number) => {
    console.log('Product clicked:', id);
  }, []);

  return (
    <div>
      {sortedProducts.map((product) => (
        <ProductCard
          key={product.id}
          product={product}
          onClick={handleProductClick}
        />
      ))}
    </div>
  );
};

// Memoize component
export const ProductCard = React.memo<ProductCardProps>(({ product, onClick }) => {
  return (
    <div onClick={() => onClick(product.id)}>
      {product.name}
    </div>
  );
});
```

### 7.3 Image Optimization

```typescript
// src/components/ui/OptimizedImage.tsx

import React, { useState } from 'react';
import { cn } from '../../utils/cn';

interface OptimizedImageProps {
  src: string;
  alt: string;
  className?: string;
  width?: number;
  height?: number;
}

export const OptimizedImage: React.FC<OptimizedImageProps> = ({
  src,
  alt,
  className,
  width,
  height,
}) => {
  const [isLoaded, setIsLoaded] = useState(false);
  const [hasError, setHasError] = useState(false);

  // Generate srcset for responsive images
  const srcSet = `
    ${src}?w=400 400w,
    ${src}?w=800 800w,
    ${src}?w=1200 1200w
  `;

  return (
    <div className={cn('relative overflow-hidden bg-gray-100', className)}>
      {!isLoaded && (
        <div className="absolute inset-0 flex items-center justify-center">
          <div className="w-8 h-8 border-4 border-gray-300 border-t-blue-600 rounded-full animate-spin" />
        </div>
      )}
      
      {hasError ? (
        <div className="absolute inset-0 flex items-center justify-center text-gray-400">
          <span>Image not available</span>
        </div>
      ) : (
        <img
          src={src}
          srcSet={srcSet}
          sizes="(max-width: 640px) 400px, (max-width: 1024px) 800px, 1200px"
          alt={alt}
          width={width}
          height={height}
          loading="lazy"
          onLoad={() => setIsLoaded(true)}
          onError={() => setHasError(true)}
          className={cn(
            'transition-opacity duration-300',
            isLoaded ? 'opacity-100' : 'opacity-0'
          )}
        />
      )}
    </div>
  );
};
```

### 7.4 Virtual Scrolling

```typescript
// src/components/ui/VirtualList.tsx

import React from 'react';
import { useVirtual } from 'react-virtual';

interface VirtualListProps<T> {
  items: T[];
  height: number;
  itemHeight: number;
  renderItem: (item: T, index: number) => React.ReactNode;
}

export function VirtualList<T>({
  items,
  height,
  itemHeight,
  renderItem,
}: VirtualListProps<T>) {
  const parentRef = React.useRef<HTMLDivElement>(null);

  const rowVirtualizer = useVirtual({
    size: items.length,
    parentRef,
    estimateSize: React.useCallback(() => itemHeight, [itemHeight]),
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height, overflow: 'auto' }}>
      <div
        style={{
          height: `${rowVirtualizer.totalSize}px`,
          width: '100%',
          position: 'relative',
        }}
      >
        {rowVirtualizer.virtualItems.map((virtualRow) => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            {renderItem(items[virtualRow.index], virtualRow.index)}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 8. UI/UX Patterns

### 8.1 Loading States

```typescript
// src/components/feedback/SkeletonLoader.tsx

export const ProductCardSkeleton: React.FC = () => {
  return (
    <div className="bg-white rounded-lg shadow animate-pulse">
      <div className="aspect-square bg-gray-300 rounded-t-lg" />
      <div className="p-4 space-y-3">
        <div className="h-4 bg-gray-300 rounded w-3/4" />
        <div className="h-4 bg-gray-300 rounded" />
        <div className="h-4 bg-gray-300 rounded w-1/2" />
      </div>
    </div>
  );
};

// Usage
{isLoading ? (
  <div className="grid grid-cols-4 gap-4">
    {Array(8).fill(0).map((_, i) => (
      <ProductCardSkeleton key={i} />
    ))}
  </div>
) : (
  <ProductGrid products={products} />
)}
```

### 8.2 Error Handling

```typescript
// src/components/feedback/ErrorBoundary.tsx

import React, { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error?: Error;
}

export class ErrorBoundary extends Component<Props, State> {
  public state: State = {
    hasError: false,
  };

  public static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  public componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Uncaught error:', error, errorInfo);
    // Send to error tracking service (Sentry)
  }

  public render() {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <div className="min-h-screen flex items-center justify-center bg-gray-50">
          <div className="text-center">
            <h1 className="text-4xl font-bold text-gray-900 mb-4">
              Oops! Something went wrong
            </h1>
            <p className="text-gray-600 mb-4">
              We're sorry for the inconvenience. Please try refreshing the page.
            </p>
            <button
              onClick={() => window.location.reload()}
              className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
            >
              Refresh Page
            </button>
          </div>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### 8.3 Toast Notifications

```typescript
// src/components/feedback/Toast.tsx

import { Toaster, toast } from 'react-hot-toast';

// Setup in App.tsx
<Toaster
  position="top-right"
  toastOptions={{
    duration: 3000,
    style: {
      background: '#363636',
      color: '#fff',
    },
    success: {
      duration: 3000,
      iconTheme: {
        primary: '#10b981',
        secondary: '#fff',
      },
    },
    error: {
      duration: 4000,
      iconTheme: {
        primary: '#ef4444',
        secondary: '#fff',
      },
    },
  }}
/>

// Usage
toast.success('Product added to cart!');
toast.error('Failed to add product');
toast.loading('Processing...');
```

---

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Next Review:** April 26, 2026  
**Status:** ✅ Production-Ready

---

[← Back to Index](./00_MASTER_INDEX.md) | [Next: API Contracts →](./05_API_CONTRACTS.md)
