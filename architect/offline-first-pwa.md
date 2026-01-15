# Offline-First & PWA - Kỹ Thuật Xử Lý Khi Web Offline

> **Mục tiêu**: Hiểu và implement các kỹ thuật để web app hoạt động khi offline, tạo trải nghiệm tốt như native app.

---

## 📚 Mục Lục

1. [Tổng Quan Offline-First](#tổng-quan-offline-first)
2. [Service Workers](#service-workers)
3. [Caching Strategies](#caching-strategies)
4. [IndexedDB](#indexeddb)
5. [Background Sync](#background-sync)
6. [Push Notifications](#push-notifications)
7. [Progressive Web Apps (PWA)](#progressive-web-apps-pwa)
8. [Offline UI/UX Patterns](#offline-uiux-patterns)
9. [Real-World Implementation](#real-world-implementation)

---

## 🎯 Tổng Quan Offline-First

### Offline-First Philosophy

```
Traditional Approach:
Online → Works ✅
Offline → Broken ❌

Offline-First Approach:
Offline → Works ✅ (cached data)
Online → Syncs ✅ (fresh data)
```

### Tại Sao Cần Offline-First?

1. **Unreliable Networks**: Subway, airplane, rural areas
2. **Better Performance**: Instant load from cache
3. **User Experience**: App vẫn dùng được
4. **Data Persistence**: Không mất data khi offline

### Technologies Stack

```
┌─────────────────────────────────────────┐
│     OFFLINE-FIRST TECH STACK            │
├─────────────────────────────────────────┤
│                                         │
│  1. Service Workers                     │
│     → Intercept network requests        │
│     → Cache management                  │
│                                         │
│  2. Cache API                           │
│     → Store static assets               │
│     → Store API responses               │
│                                         │
│  3. IndexedDB                           │
│     → Store structured data             │
│     → Large storage (50MB+)             │
│                                         │
│  4. Background Sync                     │
│     → Retry failed requests             │
│     → Sync when online                  │
│                                         │
│  5. Push Notifications                  │
│     → Engage users                      │
│     → Update notifications              │
│                                         │
└─────────────────────────────────────────┘
```

---

## 🔧 Service Workers

### Service Worker Là Gì?

**Service Worker** = JavaScript chạy trong background, independent từ web page, có thể intercept network requests.

### Lifecycle

```
1. Register → 2. Install → 3. Activate → 4. Fetch/Message
```

### Basic Implementation

#### 1. Register Service Worker

```typescript
// app.tsx
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const registration = await navigator.serviceWorker.register('/sw.js');
      console.log('SW registered:', registration.scope);
      
      // Listen for updates
      registration.addEventListener('updatefound', () => {
        const newWorker = registration.installing;
        newWorker?.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            // New version available
            showUpdateNotification();
          }
        });
      });
    } catch (error) {
      console.error('SW registration failed:', error);
    }
  });
}
```

#### 2. Service Worker File

```javascript
// sw.js
const CACHE_NAME = 'my-app-v1';
const urlsToCache = [
  '/',
  '/styles/main.css',
  '/scripts/main.js',
  '/images/logo.png'
];

// Install: Cache static assets
self.addEventListener('install', (event) => {
  console.log('SW: Install event');
  
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => {
        console.log('SW: Caching files');
        return cache.addAll(urlsToCache);
      })
      .then(() => self.skipWaiting()) // Activate immediately
  );
});

// Activate: Clean up old caches
self.addEventListener('activate', (event) => {
  console.log('SW: Activate event');
  
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames.map((cacheName) => {
          if (cacheName !== CACHE_NAME) {
            console.log('SW: Deleting old cache:', cacheName);
            return caches.delete(cacheName);
          }
        })
      );
    }).then(() => self.clients.claim()) // Take control immediately
  );
});

// Fetch: Serve from cache or network
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        // Cache hit - return response
        if (response) {
          return response;
        }
        
        // Cache miss - fetch from network
        return fetch(event.request);
      })
  );
});
```

---

## 📦 Caching Strategies

### 1. Cache First (Offline-First)

**Use case**: Static assets (CSS, JS, images)

```javascript
// Cache first, fallback to network
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        return response || fetch(event.request);
      })
  );
});
```

**Flow:**
```
Request → Check Cache → Found? Return : Fetch Network
```

### 2. Network First (Online-First)

**Use case**: API calls, dynamic content

```javascript
// Network first, fallback to cache
self.addEventListener('fetch', (event) => {
  event.respondWith(
    fetch(event.request)
      .then((response) => {
        // Clone response (can only read once)
        const responseClone = response.clone();
        
        // Update cache
        caches.open(CACHE_NAME).then((cache) => {
          cache.put(event.request, responseClone);
        });
        
        return response;
      })
      .catch(() => {
        // Network failed, try cache
        return caches.match(event.request);
      })
  );
});
```

**Flow:**
```
Request → Fetch Network → Success? Cache & Return : Check Cache
```

### 3. Stale-While-Revalidate

**Use case**: Balance freshness & speed

```javascript
// Return cache immediately, update in background
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.match(event.request).then((cachedResponse) => {
        // Fetch from network in background
        const fetchPromise = fetch(event.request).then((networkResponse) => {
          // Update cache
          cache.put(event.request, networkResponse.clone());
          return networkResponse;
        });
        
        // Return cached response immediately, or wait for network
        return cachedResponse || fetchPromise;
      });
    })
  );
});
```

**Flow:**
```
Request → Return Cache (if exists) + Fetch Network (update cache)
```

### 4. Network Only

**Use case**: Always need fresh data (analytics, POST requests)

```javascript
self.addEventListener('fetch', (event) => {
  // Only fetch from network
  event.respondWith(fetch(event.request));
});
```

### 5. Cache Only

**Use case**: App shell (never changes)

```javascript
self.addEventListener('fetch', (event) => {
  // Only serve from cache
  event.respondWith(caches.match(event.request));
});
```

### Strategy Selection

```typescript
// sw.js - Different strategies for different resources
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);
  
  // API calls: Network first
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(request));
  }
  // Static assets: Cache first
  else if (request.destination === 'image' || 
           request.destination === 'script' ||
           request.destination === 'style') {
    event.respondWith(cacheFirst(request));
  }
  // HTML: Stale-while-revalidate
  else if (request.destination === 'document') {
    event.respondWith(staleWhileRevalidate(request));
  }
  // Default: Network only
  else {
    event.respondWith(fetch(request));
  }
});
```

---

## 💾 IndexedDB

### IndexedDB Là Gì?

**IndexedDB** = NoSQL database trong browser, lưu trữ structured data lớn (50MB+).

### Basic Usage

```typescript
// Open database
function openDB(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open('MyAppDB', 1);
    
    // Create object stores
    request.onupgradeneeded = (event) => {
      const db = (event.target as IDBOpenDBRequest).result;
      
      // Create object store
      if (!db.objectStoreNames.contains('posts')) {
        const objectStore = db.createObjectStore('posts', { keyPath: 'id' });
        objectStore.createIndex('createdAt', 'createdAt', { unique: false });
      }
    };
    
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

// Add data
async function addPost(post: Post) {
  const db = await openDB();
  const transaction = db.transaction(['posts'], 'readwrite');
  const objectStore = transaction.objectStore('posts');
  
  return new Promise((resolve, reject) => {
    const request = objectStore.add(post);
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

// Get data
async function getPost(id: string): Promise<Post> {
  const db = await openDB();
  const transaction = db.transaction(['posts'], 'readonly');
  const objectStore = transaction.objectStore('posts');
  
  return new Promise((resolve, reject) => {
    const request = objectStore.get(id);
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

// Get all data
async function getAllPosts(): Promise<Post[]> {
  const db = await openDB();
  const transaction = db.transaction(['posts'], 'readonly');
  const objectStore = transaction.objectStore('posts');
  
  return new Promise((resolve, reject) => {
    const request = objectStore.getAll();
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}

// Delete data
async function deletePost(id: string) {
  const db = await openDB();
  const transaction = db.transaction(['posts'], 'readwrite');
  const objectStore = transaction.objectStore('posts');
  
  return new Promise((resolve, reject) => {
    const request = objectStore.delete(id);
    request.onsuccess = () => resolve(request.result);
    request.onerror = () => reject(request.error);
  });
}
```

### Using idb Library (Easier)

```typescript
import { openDB } from 'idb';

// Open database
const db = await openDB('MyAppDB', 1, {
  upgrade(db) {
    // Create object store
    const postStore = db.createObjectStore('posts', { keyPath: 'id' });
    postStore.createIndex('createdAt', 'createdAt');
  }
});

// Add data
await db.add('posts', post);

// Get data
const post = await db.get('posts', id);

// Get all
const allPosts = await db.getAll('posts');

// Delete
await db.delete('posts', id);

// Query with index
const recentPosts = await db.getAllFromIndex(
  'posts',
  'createdAt',
  IDBKeyRange.lowerBound(yesterday)
);
```

---

## 🔄 Background Sync

### Background Sync Là Gì?

**Background Sync** = Retry failed requests khi device online lại.

### Implementation

```typescript
// Register sync when offline
async function createPost(post: Post) {
  try {
    // Try to send immediately
    const response = await fetch('/api/posts', {
      method: 'POST',
      body: JSON.stringify(post)
    });
    
    if (!response.ok) throw new Error('Failed');
    
    return response.json();
  } catch (error) {
    // Save to IndexedDB
    await savePostToIndexedDB(post);
    
    // Register background sync
    if ('serviceWorker' in navigator && 'sync' in ServiceWorkerRegistration.prototype) {
      const registration = await navigator.serviceWorker.ready;
      await registration.sync.register('sync-posts');
    }
    
    throw error;
  }
}
```

```javascript
// sw.js - Handle sync event
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-posts') {
    event.waitUntil(syncPosts());
  }
});

async function syncPosts() {
  // Get pending posts from IndexedDB
  const db = await openDB('MyAppDB', 1);
  const pendingPosts = await db.getAll('pendingPosts');
  
  // Try to sync each post
  const syncPromises = pendingPosts.map(async (post) => {
    try {
      const response = await fetch('/api/posts', {
        method: 'POST',
        body: JSON.stringify(post)
      });
      
      if (response.ok) {
        // Success - remove from pending
        await db.delete('pendingPosts', post.id);
      }
    } catch (error) {
      console.error('Sync failed:', error);
      // Will retry on next sync
    }
  });
  
  await Promise.allSettled(syncPromises);
}
```

---

## 🔔 Push Notifications

### Setup

```typescript
// Request permission
async function requestNotificationPermission() {
  const permission = await Notification.requestPermission();
  
  if (permission === 'granted') {
    // Subscribe to push
    const registration = await navigator.serviceWorker.ready;
    const subscription = await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY)
    });
    
    // Send subscription to server
    await fetch('/api/push/subscribe', {
      method: 'POST',
      body: JSON.stringify(subscription)
    });
  }
}
```

```javascript
// sw.js - Handle push event
self.addEventListener('push', (event) => {
  const data = event.data.json();
  
  const options = {
    body: data.body,
    icon: '/icon.png',
    badge: '/badge.png',
    data: {
      url: data.url
    }
  };
  
  event.waitUntil(
    self.registration.showNotification(data.title, options)
  );
});

// Handle notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close();
  
  event.waitUntil(
    clients.openWindow(event.notification.data.url)
  );
});
```

---

## 📱 Progressive Web Apps (PWA)

### PWA Checklist

```
✅ HTTPS
✅ Service Worker
✅ Web App Manifest
✅ Responsive Design
✅ Offline Functionality
✅ Fast Loading
✅ Installable
```

### Web App Manifest

```json
// public/manifest.json
{
  "name": "My Social App",
  "short_name": "MySocial",
  "description": "A social media app",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#000000",
  "orientation": "portrait",
  "icons": [
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/home.png",
      "sizes": "540x720",
      "type": "image/png"
    }
  ]
}
```

```html
<!-- index.html -->
<link rel="manifest" href="/manifest.json" />
<meta name="theme-color" content="#000000" />
<link rel="apple-touch-icon" href="/icons/icon-192x192.png" />
```

### Install Prompt

```typescript
let deferredPrompt: any;

window.addEventListener('beforeinstallprompt', (e) => {
  // Prevent default prompt
  e.preventDefault();
  
  // Save event
  deferredPrompt = e;
  
  // Show custom install button
  showInstallButton();
});

async function handleInstallClick() {
  if (!deferredPrompt) return;
  
  // Show install prompt
  deferredPrompt.prompt();
  
  // Wait for user choice
  const { outcome } = await deferredPrompt.userChoice;
  
  if (outcome === 'accepted') {
    console.log('User accepted install');
  }
  
  deferredPrompt = null;
}

// Detect if already installed
window.addEventListener('appinstalled', () => {
  console.log('PWA installed');
  hideInstallButton();
});
```

---

## 🎨 Offline UI/UX Patterns

### 1. Offline Indicator

```typescript
function OfflineIndicator() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  
  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);
  
  if (isOnline) return null;
  
  return (
    <div className="offline-banner">
      <span>⚠️ You are offline</span>
      <p>Changes will sync when you're back online</p>
    </div>
  );
}
```

### 2. Optimistic UI

```typescript
function LikeButton({ postId }: { postId: string }) {
  const [isLiked, setIsLiked] = useState(false);
  const [isPending, setIsPending] = useState(false);
  
  const handleLike = async () => {
    // Optimistic update
    setIsLiked(true);
    setIsPending(true);
    
    try {
      await likePost(postId);
      setIsPending(false);
    } catch (error) {
      // Revert on error
      setIsLiked(false);
      setIsPending(false);
      
      // Show retry option
      showRetryNotification();
    }
  };
  
  return (
    <button onClick={handleLike} disabled={isPending}>
      {isLiked ? '❤️' : '🤍'}
      {isPending && <Spinner />}
    </button>
  );
}
```

### 3. Sync Status

```typescript
function SyncStatus() {
  const [pendingCount, setPendingCount] = useState(0);
  const [isSyncing, setIsSyncing] = useState(false);
  
  useEffect(() => {
    // Check pending items
    const checkPending = async () => {
      const db = await openDB('MyAppDB', 1);
      const pending = await db.getAll('pendingPosts');
      setPendingCount(pending.length);
    };
    
    checkPending();
    
    // Listen for online event
    const handleOnline = async () => {
      setIsSyncing(true);
      
      // Trigger sync
      const registration = await navigator.serviceWorker.ready;
      await registration.sync.register('sync-posts');
      
      // Wait a bit then check again
      setTimeout(async () => {
        await checkPending();
        setIsSyncing(false);
      }, 2000);
    };
    
    window.addEventListener('online', handleOnline);
    return () => window.removeEventListener('online', handleOnline);
  }, []);
  
  if (pendingCount === 0) return null;
  
  return (
    <div className="sync-status">
      {isSyncing ? (
        <span>🔄 Syncing {pendingCount} items...</span>
      ) : (
        <span>📤 {pendingCount} items pending sync</span>
      )}
    </div>
  );
}
```

---

## 🛠️ Real-World Implementation

### Complete Example: Offline-First Social App

```typescript
// hooks/useOfflinePost.ts
import { useState, useEffect } from 'react';
import { openDB } from 'idb';

interface Post {
  id: string;
  content: string;
  createdAt: number;
  synced: boolean;
}

export function useOfflinePost() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  
  // Load posts from IndexedDB
  useEffect(() => {
    loadPosts();
  }, []);
  
  // Listen for online/offline
  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true);
      syncPosts();
    };
    const handleOffline = () => setIsOnline(false);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);
  
  async function loadPosts() {
    const db = await openDB('SocialAppDB', 1, {
      upgrade(db) {
        db.createObjectStore('posts', { keyPath: 'id' });
      }
    });
    
    const allPosts = await db.getAll('posts');
    setPosts(allPosts);
  }
  
  async function createPost(content: string) {
    const post: Post = {
      id: crypto.randomUUID(),
      content,
      createdAt: Date.now(),
      synced: false
    };
    
    // Save to IndexedDB immediately
    const db = await openDB('SocialAppDB', 1);
    await db.add('posts', post);
    
    // Update UI
    setPosts(prev => [post, ...prev]);
    
    // Try to sync
    if (isOnline) {
      try {
        await fetch('/api/posts', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(post)
        });
        
        // Mark as synced
        post.synced = true;
        await db.put('posts', post);
        setPosts(prev => prev.map(p => p.id === post.id ? post : p));
      } catch (error) {
        // Will sync later via background sync
        console.error('Failed to sync:', error);
      }
    }
  }
  
  async function syncPosts() {
    const db = await openDB('SocialAppDB', 1);
    const unsyncedPosts = (await db.getAll('posts'))
      .filter(p => !p.synced);
    
    for (const post of unsyncedPosts) {
      try {
        await fetch('/api/posts', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(post)
        });
        
        post.synced = true;
        await db.put('posts', post);
      } catch (error) {
        console.error('Sync failed:', error);
      }
    }
    
    await loadPosts();
  }
  
  return {
    posts,
    createPost,
    isOnline,
    pendingCount: posts.filter(p => !p.synced).length
  };
}
```

```typescript
// components/PostComposer.tsx
function PostComposer() {
  const { createPost, isOnline, pendingCount } = useOfflinePost();
  const [content, setContent] = useState('');
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!content.trim()) return;
    
    await createPost(content);
    setContent('');
  };
  
  return (
    <div>
      {!isOnline && (
        <div className="offline-warning">
          ⚠️ You're offline. Posts will sync when you're back online.
        </div>
      )}
      
      {pendingCount > 0 && (
        <div className="sync-status">
          📤 {pendingCount} posts pending sync
        </div>
      )}
      
      <form onSubmit={handleSubmit}>
        <textarea
          value={content}
          onChange={(e) => setContent(e.target.value)}
          placeholder="What's on your mind?"
        />
        <button type="submit">
          Post {!isOnline && '(Offline)'}
        </button>
      </form>
    </div>
  );
}
```

---

## 📊 Offline-First Checklist

### Implementation
- [ ] ✅ Service Worker registered
- [ ] ✅ Caching strategy implemented
- [ ] ✅ IndexedDB for data storage
- [ ] ✅ Background Sync for failed requests
- [ ] ✅ Offline detection
- [ ] ✅ Web App Manifest

### UX
- [ ] ✅ Offline indicator
- [ ] ✅ Optimistic UI updates
- [ ] ✅ Sync status display
- [ ] ✅ Retry mechanisms
- [ ] ✅ Clear error messages

### Testing
- [ ] ✅ Test offline scenarios
- [ ] ✅ Test sync behavior
- [ ] ✅ Test storage limits
- [ ] ✅ Test on slow networks

---

## 🎯 Best Practices

### 1. Storage Limits

```typescript
// Check storage quota
async function checkStorageQuota() {
  if ('storage' in navigator && 'estimate' in navigator.storage) {
    const estimate = await navigator.storage.estimate();
    const percentUsed = (estimate.usage! / estimate.quota!) * 100;
    
    console.log(`Storage: ${percentUsed.toFixed(2)}% used`);
    console.log(`Available: ${(estimate.quota! - estimate.usage!) / 1024 / 1024} MB`);
    
    if (percentUsed > 80) {
      // Warn user or clean up old data
      cleanupOldData();
    }
  }
}
```

### 2. Cache Versioning

```javascript
// sw.js
const CACHE_VERSION = 'v2';
const CACHE_NAME = `my-app-${CACHE_VERSION}`;

// Clean up old versions
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter(name => name.startsWith('my-app-') && name !== CACHE_NAME)
          .map(name => caches.delete(name))
      );
    })
  );
});
```

### 3. Update Notification

```typescript
function UpdateNotification() {
  const [showUpdate, setShowUpdate] = useState(false);
  
  useEffect(() => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.ready.then((registration) => {
        registration.addEventListener('updatefound', () => {
          const newWorker = registration.installing;
          
          newWorker?.addEventListener('statechange', () => {
            if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
              setShowUpdate(true);
            }
          });
        });
      });
    }
  }, []);
  
  const handleUpdate = () => {
    window.location.reload();
  };
  
  if (!showUpdate) return null;
  
  return (
    <div className="update-notification">
      <p>New version available!</p>
      <button onClick={handleUpdate}>Update Now</button>
    </div>
  );
}
```

---

**Remember**: Offline-first không chỉ là technical feature, mà là UX philosophy - app phải hoạt động mọi lúc, mọi nơi! 🚀
