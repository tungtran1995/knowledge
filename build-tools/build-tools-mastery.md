# Build Tools Mastery - Làm Chủ Build Tools Từ A-Z

> **Mục tiêu**: Hiểu sâu về build tools, từ concepts cơ bản đến advanced configurations, để có thể tự tin configure, optimize và debug bất kỳ build system nào.

---

## 📚 Mục Lục

1. [Tại Sao Cần Build Tools?](#tại-sao-cần-build-tools)
2. [Core Concepts](#core-concepts)
3. [Webpack Deep Dive](#webpack-deep-dive)
4. [Vite Deep Dive](#vite-deep-dive)
5. [Rollup Deep Dive](#rollup-deep-dive)
6. [esbuild & swc](#esbuild--swc)
7. [So Sánh & Khi Nào Dùng Gì](#so-sánh--khi-nào-dùng-gì)
8. [Advanced Topics](#advanced-topics)
9. [Troubleshooting & Debugging](#troubleshooting--debugging)
10. [Hands-on Projects](#hands-on-projects)

---

## 🤔 Tại Sao Cần Build Tools?

### Vấn Đề Của Modern Web Development

```javascript
// ❌ Vấn đề 1: Browser không hiểu JSX
const App = () => <div>Hello World</div>;

// ❌ Vấn đề 2: Browser không support ES6 modules tốt
import { useState } from 'react';

// ❌ Vấn đề 3: Browser không hiểu TypeScript
const greeting: string = "Hello";

// ❌ Vấn đề 4: CSS modules, SCSS không native
import styles from './App.module.css';

// ❌ Vấn đề 5: Performance - nhiều files nhỏ
// 100 files × 50ms latency = 5 seconds load time!
```

### Build Tools Giải Quyết Gì?

1. **Transpilation**: JSX, TypeScript, ES6+ → ES5
2. **Bundling**: Nhiều files → Ít files (1-3 bundles)
3. **Minification**: Giảm file size (remove whitespace, shorten names)
4. **Code Splitting**: Chia code thành chunks, lazy load
5. **Asset Processing**: Images, fonts, CSS optimization
6. **Development Experience**: Hot reload, fast refresh
7. **Optimization**: Tree shaking, dead code elimination

---

## 🧠 Core Concepts

### 1. Module Systems

#### CommonJS (Node.js traditional)
```javascript
// math.js
module.exports = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b
};

// app.js
const math = require('./math');
console.log(math.add(2, 3)); // 5
```

**Đặc điểm:**
- ✅ Synchronous loading (phù hợp server-side)
- ✅ Dynamic imports: `require(variablePath)`
- ❌ Không tree-shakeable (bundle toàn bộ module)
- ❌ Runtime resolution

#### ES Modules (ESM - Modern standard)
```javascript
// math.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;

// app.js
import { add } from './math';
console.log(add(2, 3)); // 5
```

**Đặc điểm:**
- ✅ Static analysis → Tree shaking
- ✅ Asynchronous loading (phù hợp browser)
- ✅ Named exports & default exports
- ❌ Không dynamic imports (phải dùng `import()`)

#### Dynamic Imports
```javascript
// Lazy loading
const loadMath = async () => {
  const math = await import('./math');
  return math.add(2, 3);
};

// React code splitting
const LazyComponent = React.lazy(() => import('./HeavyComponent'));
```

### 2. Dependency Graph

```
index.js
├── App.jsx
│   ├── Header.jsx
│   │   └── Logo.svg
│   ├── Content.jsx
│   │   ├── Article.jsx
│   │   └── styles.css
│   └── Footer.jsx
└── utils.js
    └── lodash (node_modules)
```

**Build tool sẽ:**
1. Start từ entry point (`index.js`)
2. Parse imports/requires
3. Build dependency graph
4. Bundle theo graph này
5. Optimize (tree shake, minify)

### 3. Loaders vs Plugins

#### Loaders (Webpack concept)
- **Transform files** trước khi add vào bundle
- Chạy **per-file** basis
- Chain được (right to left)

```javascript
// Ví dụ: .scss → .css → CSS-in-JS
{
  test: /\.scss$/,
  use: ['style-loader', 'css-loader', 'sass-loader']
  // sass-loader → css-loader → style-loader
}
```

#### Plugins
- **Transform entire bundle** hoặc build process
- Chạy ở **build lifecycle hooks**
- Powerful hơn loaders

```javascript
// Ví dụ: Generate HTML, optimize images, analyze bundle
plugins: [
  new HtmlWebpackPlugin(),
  new MiniCssExtractPlugin(),
  new BundleAnalyzerPlugin()
]
```

### 4. Tree Shaking

**Concept**: Loại bỏ dead code (code không được sử dụng)

```javascript
// utils.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b; // ❌ Không dùng
export const multiply = (a, b) => a * b; // ❌ Không dùng

// app.js
import { add } from './utils';
console.log(add(2, 3));

// ✅ Bundle chỉ chứa `add`, không có subtract/multiply
```

**Yêu cầu:**
- ✅ ES Modules (static imports)
- ✅ `"sideEffects": false` trong package.json
- ✅ Production mode

**Side Effects Example:**
```javascript
// ❌ Side effect - modify global
window.myGlobal = 'value';

// ❌ Side effect - polyfill
import 'core-js/stable';

// ✅ Pure function - no side effect
export const add = (a, b) => a + b;
```

### 5. Code Splitting

#### Types of Code Splitting

**1. Entry Point Splitting**
```javascript
// webpack.config.js
module.exports = {
  entry: {
    app: './src/app.js',
    admin: './src/admin.js'
  },
  output: {
    filename: '[name].bundle.js'
  }
};
// → app.bundle.js, admin.bundle.js
```

**2. Dynamic Imports**
```javascript
// Lazy load khi cần
button.addEventListener('click', async () => {
  const module = await import('./heavy-module.js');
  module.doSomething();
});
```

**3. SplitChunks (Webpack)**
```javascript
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        priority: 10
      },
      common: {
        minChunks: 2,
        priority: 5,
        reuseExistingChunk: true
      }
    }
  }
}
```

### 6. Source Maps

**Vấn đề**: Minified code khó debug

```javascript
// Original
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// Minified
function a(b){return b.reduce((c,d)=>c+d.price,0)}
// ❌ Error at line 1, column 45 - WTF is this?
```

**Solution**: Source Maps

```javascript
// webpack.config.js
module.exports = {
  devtool: 'source-map' // or 'inline-source-map', 'eval-source-map'
};
```

**Types:**
- `eval`: Fastest, không có column mapping
- `source-map`: Slowest, full mapping, separate file
- `inline-source-map`: Full mapping, inline trong bundle
- `cheap-source-map`: Faster, line-only mapping
- `eval-source-map`: Development best (fast + good mapping)

---

## 🔧 Webpack Deep Dive

### Webpack Là Gì?

**Webpack = Module Bundler**
- Nhận vào: Entry point(s)
- Xử lý: Dependency graph
- Xuất ra: Bundle(s)

### Core Concepts

#### 1. Entry
```javascript
// Single entry
module.exports = {
  entry: './src/index.js'
};

// Multiple entries
module.exports = {
  entry: {
    app: './src/app.js',
    admin: './src/admin.js',
    vendor: ['react', 'react-dom', 'lodash']
  }
};

// Dynamic entry
module.exports = {
  entry: () => ({
    app: './src/app.js',
    admin: './src/admin.js'
  })
};
```

#### 2. Output
```javascript
const path = require('path');

module.exports = {
  output: {
    // Output directory (absolute path)
    path: path.resolve(__dirname, 'dist'),
    
    // Filename pattern
    filename: '[name].[contenthash].js',
    // [name] = entry name
    // [contenthash] = hash based on content (cache busting)
    
    // Public path (CDN URL)
    publicPath: 'https://cdn.example.com/assets/',
    
    // Clean dist before build
    clean: true,
    
    // Asset modules filename
    assetModuleFilename: 'assets/[hash][ext][query]'
  }
};
```

#### 3. Loaders

**Concept**: Transform non-JS files → JS modules

```javascript
module.exports = {
  module: {
    rules: [
      // JavaScript/JSX
      {
        test: /\.(js|jsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env', '@babel/preset-react']
          }
        }
      },
      
      // TypeScript
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/
      },
      
      // CSS
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader']
        // css-loader: Parse CSS imports
        // style-loader: Inject CSS into DOM
      },
      
      // SCSS/SASS
      {
        test: /\.scss$/,
        use: [
          'style-loader',
          {
            loader: 'css-loader',
            options: {
              modules: true, // CSS Modules
              sourceMap: true
            }
          },
          'sass-loader'
        ]
      },
      
      // Images (Webpack 5 Asset Modules)
      {
        test: /\.(png|jpg|gif|svg)$/,
        type: 'asset/resource',
        generator: {
          filename: 'images/[hash][ext][query]'
        }
      },
      
      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        type: 'asset/resource',
        generator: {
          filename: 'fonts/[hash][ext][query]'
        }
      },
      
      // Inline small images as base64
      {
        test: /\.(png|jpg|gif)$/,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024 // 8kb
          }
        }
      }
    ]
  }
};
```

#### 4. Plugins

**Popular Plugins:**

```javascript
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const CopyWebpackPlugin = require('copy-webpack-plugin');
const webpack = require('webpack');

module.exports = {
  plugins: [
    // 1. Generate HTML file
    new HtmlWebpackPlugin({
      template: './src/index.html',
      filename: 'index.html',
      inject: 'body',
      minify: {
        removeComments: true,
        collapseWhitespace: true
      }
    }),
    
    // 2. Extract CSS to separate file
    new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css',
      chunkFilename: '[id].[contenthash].css'
    }),
    
    // 3. Clean dist folder
    new CleanWebpackPlugin(),
    
    // 4. Copy static files
    new CopyWebpackPlugin({
      patterns: [
        { from: 'public', to: 'public' }
      ]
    }),
    
    // 5. Define environment variables
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify('production'),
      'process.env.API_URL': JSON.stringify('https://api.example.com')
    }),
    
    // 6. Bundle analyzer
    new (require('webpack-bundle-analyzer').BundleAnalyzerPlugin)({
      analyzerMode: 'static',
      openAnalyzer: false
    })
  ]
};
```

#### 5. Optimization

```javascript
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = {
  optimization: {
    // Minimize code
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true, // Remove console.log
            drop_debugger: true
          }
        }
      }),
      new CssMinimizerPlugin()
    ],
    
    // Split chunks
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        // Vendor chunk (node_modules)
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
          reuseExistingChunk: true
        },
        
        // Common chunk (used by 2+ modules)
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
          name: 'common'
        },
        
        // React chunk (separate React libs)
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          priority: 20
        }
      }
    },
    
    // Runtime chunk (webpack runtime code)
    runtimeChunk: 'single',
    
    // Module IDs (deterministic for caching)
    moduleIds: 'deterministic'
  }
};
```

### Complete Webpack Config Example

```javascript
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

const isDevelopment = process.env.NODE_ENV !== 'production';

module.exports = {
  // Mode
  mode: isDevelopment ? 'development' : 'production',
  
  // Entry
  entry: {
    app: './src/index.js'
  },
  
  // Output
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: isDevelopment 
      ? '[name].js' 
      : '[name].[contenthash].js',
    chunkFilename: isDevelopment
      ? '[name].chunk.js'
      : '[name].[contenthash].chunk.js',
    publicPath: '/',
    clean: true
  },
  
  // Resolve
  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx', '.json'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@components': path.resolve(__dirname, 'src/components'),
      '@utils': path.resolve(__dirname, 'src/utils')
    }
  },
  
  // Module rules
  module: {
    rules: [
      // JavaScript/TypeScript
      {
        test: /\.(js|jsx|ts|tsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              '@babel/preset-env',
              '@babel/preset-react',
              '@babel/preset-typescript'
            ],
            plugins: [
              isDevelopment && require.resolve('react-refresh/babel')
            ].filter(Boolean)
          }
        }
      },
      
      // CSS/SCSS
      {
        test: /\.(css|scss)$/,
        use: [
          isDevelopment ? 'style-loader' : MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: {
              modules: {
                auto: true,
                localIdentName: isDevelopment
                  ? '[path][name]__[local]'
                  : '[hash:base64]'
              },
              sourceMap: isDevelopment
            }
          },
          'postcss-loader',
          'sass-loader'
        ]
      },
      
      // Images
      {
        test: /\.(png|jpg|jpeg|gif|svg)$/,
        type: 'asset',
        parser: {
          dataUrlCondition: {
            maxSize: 8 * 1024
          }
        },
        generator: {
          filename: 'images/[hash][ext][query]'
        }
      },
      
      // Fonts
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        type: 'asset/resource',
        generator: {
          filename: 'fonts/[hash][ext][query]'
        }
      }
    ]
  },
  
  // Plugins
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html',
      minify: !isDevelopment
    }),
    
    !isDevelopment && new MiniCssExtractPlugin({
      filename: '[name].[contenthash].css',
      chunkFilename: '[id].[contenthash].css'
    }),
    
    !isDevelopment && new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    }),
    
    isDevelopment && new (require('@pmmmwh/react-refresh-webpack-plugin'))()
  ].filter(Boolean),
  
  // Optimization
  optimization: {
    minimize: !isDevelopment,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: !isDevelopment
          }
        }
      }),
      new CssMinimizerPlugin()
    ],
    
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10
        },
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          priority: 20
        }
      }
    },
    
    runtimeChunk: 'single',
    moduleIds: 'deterministic'
  },
  
  // Dev server
  devServer: {
    static: {
      directory: path.join(__dirname, 'public')
    },
    port: 3000,
    hot: true,
    open: true,
    historyApiFallback: true,
    compress: true
  },
  
  // Source maps
  devtool: isDevelopment ? 'eval-source-map' : 'source-map',
  
  // Performance
  performance: {
    hints: isDevelopment ? false : 'warning',
    maxEntrypointSize: 512000,
    maxAssetSize: 512000
  }
};
```

### Webpack Performance Tips

#### 1. Build Speed Optimization

```javascript
module.exports = {
  // Cache builds
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename]
    }
  },
  
  // Parallel builds
  module: {
    rules: [
      {
        test: /\.js$/,
        use: [
          {
            loader: 'thread-loader',
            options: {
              workers: 2
            }
          },
          'babel-loader'
        ]
      }
    ]
  },
  
  // Reduce resolve scope
  resolve: {
    modules: [path.resolve(__dirname, 'src'), 'node_modules'],
    extensions: ['.js', '.jsx'] // Only needed extensions
  },
  
  // Exclude from parsing
  module: {
    noParse: /jquery|lodash/
  }
};
```

#### 2. Bundle Size Optimization

```javascript
// 1. Tree shaking
optimization: {
  usedExports: true,
  sideEffects: false
}

// 2. Analyze bundle
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
plugins: [
  new BundleAnalyzerPlugin()
]

// 3. Dynamic imports
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

// 4. Externals (use CDN)
externals: {
  react: 'React',
  'react-dom': 'ReactDOM'
}
```

---

## ⚡ Vite Deep Dive

### Vite Là Gì?

**Vite = Next Generation Frontend Tooling**
- **Dev**: No bundling, native ESM, instant HMR
- **Build**: Rollup-based, optimized production bundles

### Tại Sao Vite Nhanh?

#### Traditional Bundler (Webpack)
```
Start Dev Server
  ↓
Bundle entire app (5-10s for large apps)
  ↓
Server ready
  ↓
Make change
  ↓
Re-bundle (2-5s)
  ↓
HMR update
```

#### Vite Approach
```
Start Dev Server
  ↓
Pre-bundle dependencies (esbuild - 0.5s)
  ↓
Server ready (instant!)
  ↓
Make change
  ↓
HMR update (instant!)
```

**Key Differences:**
1. **No bundling in dev**: Serve source files as native ESM
2. **esbuild pre-bundling**: Dependencies only (node_modules)
3. **On-demand compilation**: Only compile requested modules
4. **Fast HMR**: Native ESM + smart invalidation

### Vite Config

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  // Plugins
  plugins: [
    react({
      // Fast Refresh
      fastRefresh: true,
      
      // Babel options
      babel: {
        plugins: ['babel-plugin-styled-components']
      }
    })
  ],
  
  // Resolve
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@utils': path.resolve(__dirname, './src/utils')
    },
    extensions: ['.js', '.jsx', '.ts', '.tsx', '.json']
  },
  
  // Dev server
  server: {
    port: 3000,
    open: true,
    cors: true,
    
    // Proxy API requests
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    },
    
    // HMR
    hmr: {
      overlay: true
    }
  },
  
  // Build
  build: {
    // Output directory
    outDir: 'dist',
    
    // Generate sourcemaps
    sourcemap: true,
    
    // Rollup options
    rollupOptions: {
      output: {
        // Manual chunks
        manualChunks: {
          react: ['react', 'react-dom'],
          vendor: ['lodash', 'axios']
        }
      }
    },
    
    // Chunk size warning limit
    chunkSizeWarningLimit: 1000,
    
    // Minify
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true
      }
    },
    
    // CSS code splitting
    cssCodeSplit: true,
    
    // Asset inline limit
    assetsInlineLimit: 4096 // 4kb
  },
  
  // Dependency optimization
  optimizeDeps: {
    include: ['react', 'react-dom', 'lodash'],
    exclude: ['some-large-dep']
  },
  
  // CSS
  css: {
    modules: {
      localsConvention: 'camelCase'
    },
    preprocessorOptions: {
      scss: {
        additionalData: `@import "@/styles/variables.scss";`
      }
    }
  },
  
  // Environment variables
  define: {
    __APP_VERSION__: JSON.stringify('1.0.0')
  },
  
  // Preview server (for production build)
  preview: {
    port: 4173,
    open: true
  }
});
```

### Vite Plugins

```javascript
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';
import viteCompression from 'vite-plugin-compression';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    // React plugin
    react(),
    
    // Bundle analyzer
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true
    }),
    
    // Gzip compression
    viteCompression({
      algorithm: 'gzip',
      ext: '.gz'
    }),
    
    // Brotli compression
    viteCompression({
      algorithm: 'brotliCompress',
      ext: '.br'
    }),
    
    // PWA
    VitePWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'My App',
        short_name: 'App',
        theme_color: '#ffffff'
      }
    })
  ]
});
```

### Vite vs Webpack

| Feature | Vite | Webpack |
|---------|------|---------|
| **Dev Server Start** | Instant | 5-10s (large apps) |
| **HMR Speed** | Instant | 2-5s |
| **Build Tool** | Rollup | Webpack |
| **Config Complexity** | Simple | Complex |
| **Ecosystem** | Growing | Mature |
| **Learning Curve** | Easy | Steep |
| **Best For** | Modern apps, SPAs | Complex builds, legacy support |

---

## 📦 Rollup Deep Dive

### Rollup Là Gì?

**Rollup = Module Bundler for Libraries**
- Tối ưu cho **libraries**, không phải apps
- ES Modules native support
- Tree shaking tốt nhất
- Output clean, readable code

### Khi Nào Dùng Rollup?

✅ **Dùng Rollup khi:**
- Build library/package
- Cần tree shaking tốt
- Output cần readable (cho debugging)
- ES Modules priority

❌ **Không dùng Rollup khi:**
- Build application (dùng Webpack/Vite)
- Cần code splitting phức tạp
- Cần HMR trong dev

### Rollup Config

```javascript
// rollup.config.js
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import babel from '@rollup/plugin-babel';
import terser from '@rollup/plugin-terser';
import typescript from '@rollup/plugin-typescript';
import peerDepsExternal from 'rollup-plugin-peer-deps-external';
import postcss from 'rollup-plugin-postcss';
import { visualizer } from 'rollup-plugin-visualizer';

const packageJson = require('./package.json');

export default {
  // Input
  input: 'src/index.ts',
  
  // Output (multiple formats)
  output: [
    // CommonJS (for Node)
    {
      file: packageJson.main,
      format: 'cjs',
      sourcemap: true,
      exports: 'named'
    },
    
    // ES Module (for bundlers)
    {
      file: packageJson.module,
      format: 'esm',
      sourcemap: true,
      exports: 'named'
    },
    
    // UMD (for browsers)
    {
      file: packageJson.browser,
      format: 'umd',
      name: 'MyLibrary',
      sourcemap: true,
      globals: {
        react: 'React',
        'react-dom': 'ReactDOM'
      }
    }
  ],
  
  // Plugins
  plugins: [
    // Externalize peer dependencies
    peerDepsExternal(),
    
    // Resolve node_modules
    resolve({
      extensions: ['.js', '.jsx', '.ts', '.tsx']
    }),
    
    // Convert CommonJS to ES6
    commonjs(),
    
    // TypeScript
    typescript({
      tsconfig: './tsconfig.json',
      declaration: true,
      declarationDir: 'dist/types'
    }),
    
    // Babel
    babel({
      babelHelpers: 'bundled',
      exclude: 'node_modules/**',
      presets: [
        '@babel/preset-env',
        '@babel/preset-react',
        '@babel/preset-typescript'
      ]
    }),
    
    // CSS
    postcss({
      modules: true,
      extract: true,
      minimize: true
    }),
    
    // Minify
    terser(),
    
    // Visualizer
    visualizer({
      filename: 'bundle-analysis.html',
      open: true
    })
  ],
  
  // External dependencies (don't bundle)
  external: ['react', 'react-dom']
};
```

### Library Package.json

```json
{
  "name": "my-library",
  "version": "1.0.0",
  "main": "dist/index.cjs.js",
  "module": "dist/index.esm.js",
  "browser": "dist/index.umd.js",
  "types": "dist/types/index.d.ts",
  "files": [
    "dist"
  ],
  "sideEffects": false,
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "@rollup/plugin-babel": "^6.0.0",
    "@rollup/plugin-commonjs": "^25.0.0",
    "@rollup/plugin-node-resolve": "^15.0.0",
    "@rollup/plugin-terser": "^0.4.0",
    "@rollup/plugin-typescript": "^11.0.0",
    "rollup": "^4.0.0",
    "rollup-plugin-peer-deps-external": "^2.2.4",
    "rollup-plugin-postcss": "^4.0.2"
  },
  "scripts": {
    "build": "rollup -c",
    "watch": "rollup -c -w"
  }
}
```

---

## 🚀 esbuild & swc

### esbuild

**esbuild = Extremely Fast JavaScript Bundler**
- Written in Go (10-100x faster than JS-based tools)
- Used by Vite for dependency pre-bundling

```javascript
// esbuild.config.js
const esbuild = require('esbuild');

esbuild.build({
  entryPoints: ['src/index.js'],
  bundle: true,
  outfile: 'dist/bundle.js',
  minify: true,
  sourcemap: true,
  target: ['es2020'],
  loader: {
    '.js': 'jsx',
    '.png': 'file',
    '.svg': 'dataurl'
  },
  define: {
    'process.env.NODE_ENV': '"production"'
  }
}).catch(() => process.exit(1));
```

**Pros:**
- ⚡ Extremely fast
- 🎯 Simple API
- 📦 Built-in minification

**Cons:**
- ❌ No HMR
- ❌ Limited plugin ecosystem
- ❌ No advanced optimizations (vs Webpack)

### swc

**swc = Super-fast TypeScript/JavaScript Compiler**
- Written in Rust
- Drop-in replacement for Babel

```javascript
// .swcrc
{
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "tsx": true,
      "decorators": true
    },
    "transform": {
      "react": {
        "runtime": "automatic",
        "development": false,
        "refresh": true
      }
    },
    "target": "es2020"
  },
  "module": {
    "type": "es6"
  },
  "minify": true
}
```

**Use with Webpack:**
```javascript
module.exports = {
  module: {
    rules: [
      {
        test: /\.(js|jsx|ts|tsx)$/,
        use: {
          loader: 'swc-loader'
        }
      }
    ]
  }
};
```

---

## 🔍 So Sánh & Khi Nào Dùng Gì

### Decision Matrix

```
┌─────────────────────────────────────────────────────────┐
│                    Use Case                             │
├─────────────────────────────────────────────────────────┤
│ New React/Vue App (SPA)              → Vite             │
│ Large Enterprise App                 → Webpack          │
│ Library/Package                      → Rollup           │
│ Monorepo                            → Turborepo + Vite  │
│ Next.js/Remix                       → Built-in (Webpack)│
│ Maximum Speed (simple app)          → esbuild           │
└─────────────────────────────────────────────────────────┘
```

### Detailed Comparison

| Tool | Best For | Speed | Config | Ecosystem |
|------|----------|-------|--------|-----------|
| **Webpack** | Complex apps, legacy | ⭐⭐ | Complex | Huge |
| **Vite** | Modern SPAs | ⭐⭐⭐⭐⭐ | Simple | Growing |
| **Rollup** | Libraries | ⭐⭐⭐ | Medium | Good |
| **esbuild** | Simple bundling | ⭐⭐⭐⭐⭐ | Simple | Limited |
| **Parcel** | Zero-config apps | ⭐⭐⭐⭐ | Zero | Medium |

---

## 🎓 Advanced Topics

### 1. Module Federation (Webpack 5)

**Concept**: Share code between separate builds at runtime

```javascript
// App 1 (Host)
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        app2: 'app2@http://localhost:3002/remoteEntry.js'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};

// App 2 (Remote)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app2',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/Button'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};

// Usage in App 1
const RemoteButton = React.lazy(() => import('app2/Button'));
```

### 2. Custom Webpack Plugin

```javascript
class MyCustomPlugin {
  apply(compiler) {
    // Tap into compilation lifecycle
    compiler.hooks.emit.tapAsync('MyCustomPlugin', (compilation, callback) => {
      // Access compilation assets
      const assets = compilation.assets;
      
      // Create new asset
      compilation.assets['custom-file.txt'] = {
        source: () => 'Custom content',
        size: () => 14
      };
      
      callback();
    });
    
    // After emit
    compiler.hooks.afterEmit.tap('MyCustomPlugin', (compilation) => {
      console.log('Build completed!');
    });
  }
}

module.exports = {
  plugins: [new MyCustomPlugin()]
};
```

### 3. Custom Vite Plugin

```javascript
// vite-plugin-custom.js
export default function myCustomPlugin() {
  return {
    name: 'vite-plugin-custom',
    
    // Transform code
    transform(code, id) {
      if (id.endsWith('.custom')) {
        return {
          code: `export default ${JSON.stringify(code)}`,
          map: null
        };
      }
    },
    
    // Handle hot update
    handleHotUpdate({ file, server }) {
      if (file.endsWith('.custom')) {
        server.ws.send({
          type: 'custom',
          event: 'update'
        });
      }
    }
  };
}

// vite.config.js
import myCustomPlugin from './vite-plugin-custom';

export default {
  plugins: [myCustomPlugin()]
};
```

### 4. Monorepo Setup (Turborepo + Vite)

```json
// turbo.json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "dev": {
      "cache": false
    },
    "lint": {
      "outputs": []
    }
  }
}
```

```json
// package.json (root)
{
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint"
  }
}
```

---

## 🐛 Troubleshooting & Debugging

### Common Issues

#### 1. "Module not found"

```bash
# Check resolve configuration
resolve: {
  extensions: ['.js', '.jsx', '.ts', '.tsx'],
  alias: {
    '@': path.resolve(__dirname, 'src')
  }
}

# Check if file exists
# Check import path (case-sensitive on Linux)
```

#### 2. "Cannot find module 'webpack'"

```bash
# Install dependencies
npm install --save-dev webpack webpack-cli

# Clear cache
rm -rf node_modules package-lock.json
npm install
```

#### 3. Slow Build

```javascript
// Enable cache
cache: {
  type: 'filesystem'
}

// Use thread-loader
{
  test: /\.js$/,
  use: ['thread-loader', 'babel-loader']
}

// Reduce resolve scope
resolve: {
  modules: [path.resolve(__dirname, 'src'), 'node_modules']
}
```

#### 4. Large Bundle Size

```bash
# Analyze bundle
npm install --save-dev webpack-bundle-analyzer

# Check for duplicates
npm ls <package-name>

# Use dynamic imports
const Heavy = React.lazy(() => import('./Heavy'));

# Externalize large deps
externals: {
  lodash: 'lodash'
}
```

### Debugging Tools

```javascript
// 1. Webpack stats
webpack --profile --json > stats.json
// Upload to webpack.github.io/analyse

// 2. Speed measure
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin');
const smp = new SpeedMeasurePlugin();

module.exports = smp.wrap({
  // your config
});

// 3. Bundle analyzer
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
plugins: [
  new BundleAnalyzerPlugin()
]
```

---

## 🛠️ Hands-on Projects

### Project 1: Webpack từ Scratch

**Mục tiêu**: Build React app với Webpack config tự viết

```bash
# 1. Setup
mkdir webpack-from-scratch
cd webpack-from-scratch
npm init -y

# 2. Install dependencies
npm install react react-dom
npm install --save-dev \
  webpack webpack-cli webpack-dev-server \
  babel-loader @babel/core @babel/preset-env @babel/preset-react \
  html-webpack-plugin \
  css-loader style-loader \
  file-loader

# 3. Create structure
mkdir -p src public
touch src/index.js src/App.jsx public/index.html
touch webpack.config.js
```

**Tasks:**
- [ ] Configure entry, output
- [ ] Add Babel loader for JSX
- [ ] Add CSS loader
- [ ] Add HtmlWebpackPlugin
- [ ] Setup dev server
- [ ] Add production optimization
- [ ] Implement code splitting
- [ ] Add bundle analyzer

### Project 2: Migrate Webpack → Vite

**Mục tiêu**: Migrate existing Webpack app to Vite

```bash
# 1. Install Vite
npm install --save-dev vite @vitejs/plugin-react

# 2. Create vite.config.js
# 3. Update index.html (move to root, add <script type="module">)
# 4. Update package.json scripts
# 5. Test and fix issues
```

**Challenges:**
- [ ] Handle Webpack-specific loaders
- [ ] Migrate environment variables
- [ ] Fix absolute imports
- [ ] Update CSS imports
- [ ] Test HMR

### Project 3: Build Custom Library

**Mục tiêu**: Create và publish React component library

```bash
# 1. Setup with Rollup
npm init -y
npm install --save-dev rollup @rollup/plugin-babel @rollup/plugin-node-resolve

# 2. Create components
# 3. Configure Rollup for multiple outputs (CJS, ESM, UMD)
# 4. Add TypeScript types
# 5. Publish to npm
```

**Requirements:**
- [ ] Multiple output formats
- [ ] TypeScript definitions
- [ ] Tree-shakeable
- [ ] CSS bundling
- [ ] Documentation (Storybook)

---

## 📚 Learning Resources

### Official Docs
- [Webpack Documentation](https://webpack.js.org/)
- [Vite Guide](https://vitejs.dev/)
- [Rollup Guide](https://rollupjs.org/)
- [esbuild Documentation](https://esbuild.github.io/)

### Courses
- **Webpack Academy** (Sean Larkin - Webpack core team)
- **Vite Course** (Frontend Masters)
- **JavaScript Build Tools** (Udemy)

### Articles
- "Webpack: The Core Concepts" - Sean Larkin
- "Why Vite" - Evan You
- "Rollup vs Webpack" - Rich Harris

### Tools
- [Webpack Bundle Analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)
- [Bundlephobia](https://bundlephobia.com/) - Check package size
- [Webpack Visualizer](https://chrisbateman.github.io/webpack-visualizer/)

---

## ✅ Mastery Checklist

### Beginner
- [ ] Hiểu module systems (CommonJS vs ESM)
- [ ] Setup basic Webpack config
- [ ] Configure loaders (Babel, CSS)
- [ ] Use HtmlWebpackPlugin
- [ ] Setup dev server

### Intermediate
- [ ] Optimize build performance
- [ ] Implement code splitting
- [ ] Configure source maps
- [ ] Setup production build
- [ ] Use Vite for new projects
- [ ] Understand tree shaking

### Advanced
- [ ] Write custom Webpack plugin
- [ ] Configure Module Federation
- [ ] Setup monorepo with Turborepo
- [ ] Optimize bundle size (<200kb initial)
- [ ] Implement advanced caching strategies
- [ ] Build and publish library with Rollup

### Expert
- [ ] Debug complex build issues
- [ ] Migrate large apps between bundlers
- [ ] Optimize build for CI/CD
- [ ] Contribute to build tool projects
- [ ] Teach others about build tools

---

## 🎯 Next Steps

1. **Practice**: Build 3 projects với different tools
2. **Read Source**: Đọc source code của Webpack/Vite plugins
3. **Contribute**: Contribute to open source build tools
4. **Share**: Write blog posts về build optimization
5. **Stay Updated**: Follow build tool updates (Webpack 6, Vite 5)

---

**Remember**: Build tools là foundation của modern web development. Master chúng sẽ giúp bạn:
- ⚡ Build faster applications
- 🎯 Debug production issues efficiently
- 🚀 Optimize bundle sizes
- 💡 Make better architectural decisions

Good luck! 🚀
