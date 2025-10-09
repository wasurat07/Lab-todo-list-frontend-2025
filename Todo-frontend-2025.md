# Lab: สร้าง Todo Frontend ด้วย Next.js และ Deploy บน GitHub Pages

## 🎯 วัตถุประสงค์
1. สร้าง Frontend Application ด้วย Next.js
2. เชื่อมต่อกับ Backend API (Flask)
3. ตั้งค่า CI/CD ด้วย GitHub Actions
4. Deploy บน GitHub Pages (Static Site)


## 🛠️ เครื่องมือที่ต้องใช้
- Node.js 18+ และ npm
- Git และ GitHub Account
- Text Editor (VS Code แนะนำ)
- Backend API ที่ทำงานแล้ว 

---

## ส่วนที่ 1: เตรียมโปรเจกต์

### ขั้นตอนที่ 1.1: สร้างโปรเจกต์ Next.js

เปิด Terminal และรันคำสั่ง:

```bash
# สร้างโปรเจกต์ Next.js
npx create-next-app@latest todo-frontend

# ตอบคำถามดังนี้:
# ✅ TypeScript? → No
# ✅ ESLint? → Yes
# ✅ Tailwind CSS? → Yes
# ✅ src/ directory? → Yes
# ✅ App Router? → Yes
# ✅ Import alias? → No

# เข้าโปรเจกต์
cd todo-frontend
```

### ขั้นตอนที่ 1.2: ติดตั้ง Dependencies เพิ่มเติม

```bash
npm install axios
```

### ขั้นตอนที่ 1.3: สร้างโครงสร้างโปรเจกต์

```bash
# สร้างโฟลเดอร์และไฟล์
mkdir -p src/components src/lib public .github/workflows
touch src/lib/api.js
touch src/components/AddTodo.jsx
touch src/components/TodoList.jsx
touch src/components/TodoStats.jsx
touch public/.nojekyll
touch .github/workflows/deploy.yml
```

**คำอธิบาย**:
- `src/lib/`: เก็บ utility functions และ API calls
- `src/components/`: เก็บ React components
- `public/.nojekyll`: ป้องกัน GitHub Pages ใช้ Jekyll processing
- `.github/workflows/`: เก็บ CI/CD configuration

---

## ส่วนที่ 2: ตั้งค่า Configuration Files

### ขั้นตอนที่ 2.1: แก้ไข `next.config.js`  กรณีที่เป็น .mjs ให้แก้ไขเป็น .js

แทนที่เนื้อหาทั้งหมดด้วย:

```javascript
/** @type {import('next').NextConfig} */
const repoName = 'todo-frontend';
const nextConfig = {
  output: 'export',
  basePath: `/${repoName}`,     // เพิ่ม Base Path
  assetPrefix: `/${repoName}/`,
  images: {
    unoptimized: true,
  },
  trailingSlash: true,
};

module.exports = nextConfig;

```

**คำอธิบาย**:
- `output: 'export'`: สร้าง static HTML files สำหรับ GitHub Pages
- `images.unoptimized: true`: ปิด Image Optimization (ไม่รองรับใน static export)

### ขั้นตอนที่ 2.2: แก้ไข `tailwind.config.js`  กรณีที่เป็น .mjs แก้ไขให้เป็น .js

แทนที่เนื้อหาทั้งหมดด้วย:

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      animation: {
        'pulse': 'pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite',
        'spin': 'spin 1s linear infinite',
      }
    },
  },
  plugins: [],
}
```

### ขั้นตอนที่ 2.3: แก้ไข `postcss.config.js`

ตรวจสอบว่ามีเนื้อหานี้:

```javascript
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
```

### ขั้นตอนที่ 2.4: อัพเดท `package.json`

แก้ไขส่วน `dependencies` และ `devDependencies`:

```json
{
  "name": "todo-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "axios": "^1.6.2",
    "next": "14.0.4",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "autoprefixer": "^10.4.16",
    "eslint": "^8.55.0",
    "eslint-config-next": "14.0.4",
    "postcss": "^8.4.32",
    "tailwindcss": "^3.4.0"
  }
}

```

หลังแก้ไขแล้วรันคำสั่ง:

```bash
npm install
```

---

## ส่วนที่ 3: สร้าง API Layer

### ขั้นตอนที่ 3.1: สร้าง `src/lib/api.js`

```javascript
import axios from 'axios';

// ตรวจสอบ API URL - ต้องไม่มี trailing slash
const API_URL = process.env.NEXT_PUBLIC_API_URL || 
                'https://flask-todo-cicd.onrender.com/api';

console.log('API_URL:', API_URL); // ← เพิ่ม log เพื่อ debug

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
  timeout: 10000,
});

// Error interceptor
api.interceptors.response.use(
  response => response,
  error => {
    console.error('API Error:', error.response || error); // ← เพิ่ม detailed error
    return Promise.reject(error);
  }
);

export const todoAPI = {
  getTodos: async () => {
    try {
      const response = await api.get('/todos');
      return response.data;
    } catch (error) {
      console.error('getTodos error:', error);
      throw error;
    }
  },

  createTodo: async (todo) => {
    try {
      const response = await api.post('/todos', todo);
      return response.data;
    } catch (error) {
      console.error('createTodo error:', error);
      throw error;
    }
  },

  updateTodo: async (id, updates) => {
    try {
      const response = await api.put(`/todos/${id}`, updates);
      return response.data;
    } catch (error) {
      console.error('updateTodo error:', error);
      throw error;
    }
  },

  deleteTodo: async (id) => {
    try {
      const response = await api.delete(`/todos/${id}`);
      return response.data;
    } catch (error) {
      console.error('deleteTodo error:', error);
      throw error;
    }
  },

  healthCheck: async () => {
    try {
      const response = await api.get('/health');
      return response.data;
    } catch (error) {
      console.error('healthCheck error:', error);
      throw error;
    }
  },
};

export default api;

```

**คำอธิบาย**:
- สร้าง axios instance พร้อม base URL เพื่อใช้เรียก API 
- มี error interceptor สำหรับ logging
- Export ทุก API methods เป็น object เพื่อใช้งานง่าย

---

## ส่วนที่ 4: สร้าง React Components

### ขั้นตอนที่ 4.1: สร้าง `src/components/AddTodo.jsx`

```javascript
'use client';

import { useState } from 'react';

export default function AddTodo({ onAdd, loading }) {
  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');
  const [submitting, setSubmitting] = useState(false);
  const [error, setError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    console.log('Submit clicked', { title, description }); // ← Debug log
    
    if (!title.trim()) {
      setError('Please enter a title');
      return;
    }

    setSubmitting(true);
    setError(null);
    
    try {
      console.log('Calling onAdd...'); // ← Debug log
      await onAdd({ 
        title: title.trim(), 
        description: description.trim() 
      });
      console.log('onAdd success'); // ← Debug log
      setTitle('');
      setDescription('');
    } catch (error) {
      console.error('Submit error:', error); // ← Debug log
      setError('Failed to add todo. Please try again.');
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="bg-white rounded-2xl shadow-xl p-6 mb-8 border-2 border-purple-100">
      <h2 className="text-2xl font-bold text-gray-800 mb-4 flex items-center gap-2">
        <span className="text-3xl">➕</span>
        Add New Task
      </h2>
      
      {error && (
        <div className="mb-4 bg-red-50 border border-red-200 text-red-700 px-4 py-3 rounded">
          {error}
        </div>
      )}
      
      <form onSubmit={handleSubmit} className="space-y-4">
        {/* Title Input */}
        <div>
          <label className="block text-gray-700 font-semibold mb-2 text-sm uppercase tracking-wide">
            Task Title *
          </label>
          <input
            type="text"
            value={title}
            onChange={(e) => {
              console.log('Title changed:', e.target.value); // ← Debug log
              setTitle(e.target.value);
            }}
            placeholder="What do you need to do?"
            className="w-full px-4 py-3 border-2 border-gray-200 rounded-xl focus:border-purple-500 
                     focus:ring-4 focus:ring-purple-100 outline-none transition text-gray-800
                     placeholder-gray-400 disabled:bg-gray-100 disabled:cursor-not-allowed"
            disabled={submitting || loading}
            autoFocus
          />
        </div>

        {/* Description Input */}
        <div>
          <label className="block text-gray-700 font-semibold mb-2 text-sm uppercase tracking-wide">
            Description (Optional)
          </label>
          <textarea
            value={description}
            onChange={(e) => setDescription(e.target.value)}
            placeholder="Add more details..."
            rows="3"
            className="w-full px-4 py-3 border-2 border-gray-200 rounded-xl focus:border-purple-500 
                     focus:ring-4 focus:ring-purple-100 outline-none transition resize-none text-gray-800
                     placeholder-gray-400 disabled:bg-gray-100 disabled:cursor-not-allowed"
            disabled={submitting || loading}
          />
        </div>

        {/* Submit Button */}
        <button
          type="submit"
          disabled={submitting || loading}
          onClick={(e) => console.log('Button clicked', e)} // ← Debug log
          className="w-full bg-gradient-to-r from-purple-600 to-pink-600 text-white font-bold 
                   py-4 px-6 rounded-xl hover:from-purple-700 hover:to-pink-700 
                   disabled:from-gray-300 disabled:to-gray-400 disabled:cursor-not-allowed
                   transform hover:scale-[1.02] active:scale-[0.98] transition-all duration-200
                   shadow-lg hover:shadow-xl flex items-center justify-center gap-2"
        >
          {submitting ? (
            <>
              <div className="w-5 h-5 border-3 border-white border-t-transparent rounded-full animate-spin" />
              Adding...
            </>
          ) : (
            <>
              <span className="text-xl">✨</span>
              Add Task
            </>
          )}
        </button>
        
        {/* Debug Info */}
        <div className="text-xs text-gray-400 mt-2">
          Debug: Title length = {title.length}, Submitting = {submitting ? 'true' : 'false'}
        </div>
      </form>
    </div>
  );
}

```

**คำอธิบาย**:
- Component แยกออกมาเพื่อความเป็นระเบียบ
- มี local state สำหรับ form inputs
- Validation ก่อน submit
- Loading state แยกจาก parent component
- ปุ่มจะ disable เฉพาะตอน submitting

### ขั้นตอนที่ 4.2: สร้าง `src/components/TodoList.jsx`

```javascript
'use client';

import { useState } from 'react';

export default function TodoList({ todos, onUpdate, onDelete, loading }) {
  const [updatingId, setUpdatingId] = useState(null);
  const [deletingId, setDeletingId] = useState(null);

  const handleToggle = async (todo) => {
    setUpdatingId(todo.id);
    try {
      await onUpdate(todo.id, { completed: !todo.completed });
    } finally {
      setUpdatingId(null);
    }
  };

  const handleDelete = async (id) => {
    if (!confirm('Are you sure you want to delete this task?')) return;
    
    setDeletingId(id);
    try {
      await onDelete(id);
    } finally {
      setDeletingId(null);
    }
  };

  if (loading) {
    return (
      <div className="text-center py-16 bg-white rounded-2xl shadow-xl">
        <div className="inline-block w-16 h-16 border-4 border-purple-200 border-t-purple-600 
                      rounded-full animate-spin mb-4" />
        <p className="text-gray-600 text-lg font-medium">Loading your tasks...</p>
      </div>
    );
  }

  if (todos.length === 0) {
    return (
      <div className="text-center py-16 bg-white rounded-2xl shadow-xl border-2 border-dashed border-gray-300">
        <div className="text-6xl mb-4">🎯</div>
        <h3 className="text-2xl font-bold text-gray-700 mb-2">No tasks yet!</h3>
        <p className="text-gray-500 text-lg">Add your first task to get started</p>
      </div>
    );
  }

  return (
    <div className="space-y-3">
      {todos.map((todo) => (
        <div
          key={todo.id}
          className={`bg-white rounded-xl shadow-md hover:shadow-xl transition-all duration-200
                    border-2 p-5 transform hover:scale-[1.01] ${
            todo.completed 
              ? 'border-green-200 bg-green-50' 
              : 'border-purple-100 hover:border-purple-300'
          } ${updatingId === todo.id || deletingId === todo.id ? 'opacity-50' : ''}`}
        >
          <div className="flex items-start gap-4">
            {/* Checkbox */}
            <button
              onClick={() => handleToggle(todo)}
              disabled={updatingId === todo.id || deletingId === todo.id}
              className="flex-shrink-0 mt-1 disabled:cursor-not-allowed"
            >
              <div className={`w-7 h-7 rounded-lg border-3 flex items-center justify-center
                             transition-all duration-200 ${
                todo.completed
                  ? 'bg-green-500 border-green-500'
                  : 'border-gray-300 hover:border-purple-500 hover:bg-purple-50'
              }`}>
                {todo.completed && (
                  <svg className="w-5 h-5 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="3" d="M5 13l4 4L19 7" />
                  </svg>
                )}
              </div>
            </button>

            {/* Content */}
            <div className="flex-1 min-w-0">
              <h3 className={`text-lg font-bold mb-1 break-words ${
                todo.completed 
                  ? 'line-through text-gray-500' 
                  : 'text-gray-800'
              }`}>
                {todo.title}
              </h3>
              
              {todo.description && (
                <p className={`text-sm mb-2 break-words ${
                  todo.completed ? 'text-gray-400' : 'text-gray-600'
                }`}>
                  {todo.description}
                </p>
              )}
              
              <div className="flex items-center gap-2 text-xs text-gray-400">
                <span>🕐</span>
                <span>
                  {new Date(todo.created_at).toLocaleDateString('en-US', {
                    month: 'short',
                    day: 'numeric',
                    year: 'numeric',
                    hour: '2-digit',
                    minute: '2-digit'
                  })}
                </span>
              </div>
            </div>

            {/* Delete Button */}
            <button
              onClick={() => handleDelete(todo.id)}
              disabled={updatingId === todo.id || deletingId === todo.id}
              className="flex-shrink-0 w-10 h-10 rounded-lg bg-red-50 hover:bg-red-100 
                       text-red-600 hover:text-red-700 transition-all duration-200
                       flex items-center justify-center font-bold text-xl
                       hover:scale-110 active:scale-95 disabled:cursor-not-allowed
                       disabled:opacity-50"
              title="Delete task"
            >
              🗑️
            </button>
          </div>
        </div>
      ))}
    </div>
  );
}
```

**คำอธิบาย**:
- แสดง loading state และ empty state
- Checkbox animation เมื่อ toggle
- Confirm dialog ก่อนลบ
- Optimistic UI updates

### ขั้นตอนที่ 4.3: สร้าง `src/components/TodoStats.jsx`

```javascript
'use client';

export default function TodoStats({ todos }) {
  const stats = {
    total: todos.length,
    completed: todos.filter(t => t.completed).length,
    pending: todos.filter(t => !t.completed).length,
  };

  if (todos.length === 0) return null;

  return (
    <div className="grid grid-cols-3 gap-4 mb-8">
      <div className="bg-white rounded-xl shadow-md p-4 border-2 border-blue-100 text-center 
                    transform hover:scale-105 transition-transform duration-200">
        <div className="text-3xl mb-1">📊</div>
        <div className="text-2xl font-bold text-blue-600">{stats.total}</div>
        <div className="text-sm text-gray-600 font-medium">Total Tasks</div>
      </div>
      
      <div className="bg-white rounded-xl shadow-md p-4 border-2 border-green-100 text-center
                    transform hover:scale-105 transition-transform duration-200">
        <div className="text-3xl mb-1">✅</div>
        <div className="text-2xl font-bold text-green-600">{stats.completed}</div>
        <div className="text-sm text-gray-600 font-medium">Completed</div>
      </div>
      
      <div className="bg-white rounded-xl shadow-md p-4 border-2 border-orange-100 text-center
                    transform hover:scale-105 transition-transform duration-200">
        <div className="text-3xl mb-1">⏳</div>
        <div className="text-2xl font-bold text-orange-600">{stats.pending}</div>
        <div className="text-sm text-gray-600 font-medium">Pending</div>
      </div>
    </div>
  );
}

```

**คำอธิบาย**:
- แสดง statistics แบบ real-time
- Hover animation สำหรับ interactivity
- ไม่แสดงถ้าไม่มี todos

---

## ส่วนที่ 5: สร้าง Main Application

### ขั้นตอนที่ 5.1: แก้ไข `src/app/page.js`

```javascript
'use client';

import { useEffect, useState } from 'react';
import { todoAPI } from '@/lib/api';
import AddTodo from '@/components/AddTodo';
import TodoList from '@/components/TodoList';
import TodoStats from '@/components/TodoStats';

export default function Home() {
  const [todos, setTodos] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [apiStatus, setApiStatus] = useState('checking');

  useEffect(() => {
    checkHealth();
    loadTodos();
  }, []);

  const checkHealth = async () => {
    try {
      await todoAPI.healthCheck();
      setApiStatus('healthy');
    } catch (err) {
      setApiStatus('unhealthy');
      setError('Cannot connect to API. Please check if the backend is running.');
    }
  };

  const loadTodos = async () => {
    try {
      setLoading(true);
      const data = await todoAPI.getTodos();
      setTodos(data.data || []);
      setError(null);
    } catch (err) {
      setError('Failed to load todos. Please try again.');
      console.error('Load todos error:', err);
    } finally {
      setLoading(false);
    }
  };

  const handleAdd = async (todo) => {
    try {
      await todoAPI.createTodo(todo);
      await loadTodos();
    } catch (err) {
      alert('Failed to add todo. Please try again.');
      throw err;
    }
  };

  const handleUpdate = async (id, updates) => {
    try {
      await todoAPI.updateTodo(id, updates);
      await loadTodos();
    } catch (err) {
      alert('Failed to update todo. Please try again.');
      throw err;
    }
  };

  const handleDelete = async (id) => {
    try {
      await todoAPI.deleteTodo(id);
      await loadTodos();
    } catch (err) {
      alert('Failed to delete todo. Please try again.');
      throw err;
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-purple-100 via-pink-50 to-blue-100 py-8 px-4">
      <div className="max-w-4xl mx-auto">
        {/* Header */}
        <div className="text-center mb-8">
          <div className="flex items-center justify-center gap-3 mb-3">
            <div className="text-5xl animate-bounce">✨</div>
            <h1 className="text-5xl font-bold bg-gradient-to-r from-purple-600 to-pink-600 bg-clip-text text-transparent">
              My Todo List
            </h1>
            <div className="text-5xl animate-bounce">📝</div>
          </div>
          <p className="text-gray-600 text-lg mb-4">Organize your tasks beautifully</p>
          
          {/* API Status Badge */}
          <div className="inline-flex items-center gap-2 px-4 py-2 rounded-full text-sm font-medium 
                        shadow-sm bg-white border-2 border-gray-200">
            <div className={`w-3 h-3 rounded-full ${
              apiStatus === 'healthy' ? 'bg-green-500 animate-pulse' :
              apiStatus === 'unhealthy' ? 'bg-red-500' :
              'bg-yellow-500 animate-pulse'
            }`} />
            <span className="text-gray-700">
              {apiStatus === 'healthy' ? '🟢 API Connected' :
               apiStatus === 'unhealthy' ? '🔴 API Disconnected' :
               '🟡 Connecting...'}
            </span>
          </div>
        </div>

        {/* Error Message */}
        {error && (
          <div className="mb-6 bg-red-50 border-l-4 border-red-500 p-4 rounded-lg shadow-sm">
            <div className="flex items-center gap-2">
              <span className="text-2xl">⚠️</span>
              <div>
                <p className="text-red-800 font-medium">{error}</p>
                <button 
                  onClick={() => { checkHealth(); loadTodos(); }}
                  className="text-red-600 underline text-sm mt-1 hover:text-red-800"
                >
                  Try again
                </button>
              </div>
            </div>
          </div>
        )}

        {/* Add Todo Form */}
        <AddTodo onAdd={handleAdd} loading={loading} />

        {/* Statistics */}
        <TodoStats todos={todos} />

        {/* Todo List */}
        <TodoList 
          todos={todos} 
          onUpdate={handleUpdate}
          onDelete={handleDelete}
          loading={loading}
        />

        {/* Footer */}
        <div className="mt-12 text-center text-gray-500 text-sm">
          <p className="mb-2">
            Made with <span className="text-red-500 animate-pulse">❤️</span> using Next.js + Flask
          </p>
          <p className="text-xs text-gray-400">
            Frontend: GitHub Pages | Backend: Render | Database: PostgreSQL
          </p>
        </div>
      </div>
    </div>
  );
}

```

**คำอธิบาย**:
- Main component ที่จัดการ state และ data flow
- เชื่อมต่อทุก components เข้าด้วยกัน
- Error handling และ loading states

### ขั้นตอนที่ 5.2: แก้ไข `src/app/layout.js`

```javascript
import { Inter } from 'next/font/google';
import './globals.css';  // ← ต้องมี!

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'My Todo List - Beautiful Task Manager',
  description: 'A modern and beautiful todo application built with Next.js',
};

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body className={inter.className}>{children}</body>
    </html>
  );
}

```

### ขั้นตอนที่ 5.3: แก้ไข `src/app/globals.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom Scrollbar */
::-webkit-scrollbar {
  width: 10px;
}

::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 5px;
}

::-webkit-scrollbar-thumb {
  background: linear-gradient(to bottom, #a855f7, #ec4899);
  border-radius: 5px;
}

::-webkit-scrollbar-thumb:hover {
  background: linear-gradient(to bottom, #9333ea, #db2777);
}

/* Smooth Transitions */
* {
  transition-property: background-color, border-color, color, fill, stroke, opacity, box-shadow, transform;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  transition-duration: 150ms;
}

/* Focus Styles */
button:focus,
input:focus,
textarea:focus {
  outline: none;
}

/* Remove tap highlight on mobile */
* {
  -webkit-tap-highlight-color: transparent;
}

/* Animation keyframes */
@keyframes bounce {
  0%, 100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
}

```

---

## ส่วนที่ 6: ตั้งค่า GitHub Actions CI/CD

### ขั้นตอนที่ 6.1: สร้าง `.github/workflows/deploy.yml`

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    name: Build Next.js App
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      
      - name: Clear cache
        run: |
          rm -rf .next
          rm -rf node_modules/.cache
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
      
      - name: Build Next.js app
        env:
          NEXT_PUBLIC_API_URL: https://flask-todo-cicd.onrender.com/api
          NODE_ENV: production
        run: |
          npm run build
          ls -la out/
          echo "Checking for CSS files..."
          find out -name "*.css" | head -5
      
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./out

  deploy:
    name: Deploy to GitHub Pages
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
        
```

**คำอธิบาย**:
- **Build job**: Build Next.js application
- **Deploy job**: Deploy ไปยัง GitHub Pages
- ใช้ environment variable สำหรับ API URL
- Auto-deploy เมื่อ push ไป main branch

---

## ส่วนที่ 7: อัพเดท Backend CORS

### ขั้นตอนที่ 7.1: แก้ไข Backend `app/__init__.py` ในไฟล์ของ Back-end ที่สร้างจาก Python

กลับไปที่โปรเจกต์ Backend (`flask-todo-cicd`) และแก้ไข `app/__init__.py`:

```python
import os
from flask import Flask, jsonify
from flask_cors import CORS
from app.models import db
from app.routes import api
from app.config import config


def create_app(config_name=None):
    """Application factory pattern"""
    if config_name is None:
        config_name = os.getenv('FLASK_ENV', 'development')

    app = Flask(__name__)
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)

    # Enable CORS for GitHub Pages
    # แก้ไข code บรรทัด "https://your-username.github.io"  ให้เป็นโดเมนของเว็บตนเอง
    CORS(app, resources={
        r"/api/*": {
            "origins": [
                "http://localhost:3000",
                "http://localhost:5000",
                "https://*.github.io",
                "https://your-username.github.io"
            ],
            "methods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
            "allow_headers": ["Content-Type"],
            "supports_credentials": False
        }
    })

    db.init_app(app)
    app.register_blueprint(api, url_prefix='/api')

    @app.route('/')
    def index():
        return jsonify({
            'message': 'Flask Todo API',
            'version': '1.0.0',
            'endpoints': {
                'health': '/api/health',
                'todos': '/api/todos'
            }
        })

    @app.errorhandler(404)
    def not_found(error):
        return jsonify({
            'success': False,
            'error': 'Resource not found'
        }), 404

    @app.errorhandler(500)
    def internal_error(error):
        db.session.rollback()
        return jsonify({
            'success': False,
            'error': 'Internal server error'
        }), 500

    @app.errorhandler(Exception)
    def handle_exception(error):
        """Handle all unhandled exceptions"""
        db.session.rollback()
        return jsonify({
            'success': False,
            'error': 'Internal server error'
        }), 500

    with app.app_context():
        db.create_all()

    return app

```

**คำอธิบาย**:
- เพิ่ม CORS support สำหรับ GitHub Pages domain เพื่ออนุญาตให้เรียกใช้งานมาจากโดเมนของ gitHub ได้
- อนุญาต wildcard `*.github.io` เพื่อรองรับทุก GitHub Pages URLs
- แทนที่ `your-username` ด้วย GitHub username ของตนเอง

### ขั้นตอนที่ 7.2: Commit และ Push Backend Changes

```bash
cd flask-todo-cicd
git add app/__init__.py
git commit -m "feat: add CORS support for GitHub Pages"
git push origin main
```

รอให้ Backend redeploy บน Render (ประมาณ 3-5 นาที)

---

## ส่วนที่ 8: ทดสอบ Local

### ขั้นตอนที่ 8.1: สร้าง Environment Variable File

สร้างไฟล์ `.env.local` ในโฟลเดอร์ `todo-frontend`:

```env
NEXT_PUBLIC_API_URL=http://localhost:5000/api
```

**หมายเหตุ**: ไฟล์นี้จะไม่ถูก commit (อยู่ใน `.gitignore`)

### ขั้นตอนที่ 8.2: รัน Development Server

```bash
# ในโฟลเดอร์ todo-frontend
npm run dev
```

เปิดเบราว์เซอร์ที่ `http://localhost:3000`

### ขั้นตอนที่ 8.3: ทดสอบ Features

ทดสอบว่าทุกอย่างทำงานได้:

- ✅ หน้าเว็บโหลดได้
- ✅ API Status แสดง "Connected"
- ✅ เพิ่ม Todo ใหม่ได้
- ✅ Toggle Complete/Incomplete ได้
- ✅ ลบ Todo ได้
- ✅ Statistics อัพเดทถูกต้อง
- ✅ UI สวยงาม มี animations

## บันทึกรูปผลการทดลอง
```bash
# บันทึกรูปผลการทดลองที่นี่
``` 

### ขั้นตอนที่ 8.4: Test Build

```bash
npm run build
```

ถ้า build สำเร็จ จะเห็นโฟลเดอร์ `out/` ถูกสร้างขึ้น

ทดสอบ static files:
```bash
npx serve@latest out
```

เปิด `http://localhost:3000` และทดสอบอีกครั้ง

---

## ส่วนที่ 9: Deploy บน GitHub

### ขั้นตอนที่ 9.1: สร้าง GitHub Repository

1. ไปที่ [GitHub](https://github.com) และ login
2. คลิก **"New repository"** หรือ **"+"** → **"New repository"**
3. ตั้งค่า:
   - **Repository name**: `todo-frontend`
   - **Description**: "Todo application frontend with Next.js"
   - **Visibility**: Public
   - **ไม่ต้องเลือก** README, .gitignore, license
4. คลิก **"Create repository"**

### ขั้นตอนที่ 9.2: Push Code ไปยัง GitHub

```bash
# ในโฟลเดอร์ todo-frontend

# Initialize git (ถ้ายังไม่ได้ทำ)
git init

# Add all files
git add .

# Commit
git commit -m "Initial commit: Next.js Todo Frontend"

# เชื่อมต่อกับ GitHub repository
git remote add origin https://github.com/your-username/todo-frontend.git

# Push code
git branch -M main
git push -u origin main
```

**แทนที่ `your-username`** ด้วย GitHub username ของคุณ

### ขั้นตอนที่ 9.3: ตั้งค่า GitHub Pages

1. ไปที่ GitHub Repository `todo-frontend`
2. คลิก **Settings** (แท็บบนสุด)
3. ในเมนูซ้าย เลือก **Pages**
4. ในส่วน **Source**:
   - เลือก **"GitHub Actions"**
5. คลิก **Save** (ถ้ามี)

### ขั้นตอนที่ 9.4: ตั้งค่า Workflow Permissions

1. ยังอยู่ใน Settings ไปที่ **Actions** → **General**
2. Scroll ลงไปหา **"Workflow permissions"**
3. เลือก **"Read and write permissions"**
4. ✅ เลือก **"Allow GitHub Actions to create and approve pull requests"**
5. คลิก **Save**

### ขั้นตอนที่ 9.5: แก้ไข API URL ใน Workflow

แก้ไขไฟล์ `.github/workflows/deploy.yml` บรรทัดที่ 38:

```yaml
- name: Build Next.js app
  env:
    NEXT_PUBLIC_API_URL: https://your-backend-app.onrender.com/api
  run: npm run build
```

**แทนที่** `your-backend-app.onrender.com` ด้วย URL ของ Backend จริง

Commit และ Push:
```bash
git add .github/workflows/deploy.yml
git commit -m "fix: update API URL in workflow"
git push origin main
```

---

## ส่วนที่ 10: ตรวจสอบ Deployment

### ขั้นตอนที่ 10.1: ดู GitHub Actions Workflow

1. ไปที่ GitHub Repository → **Actions** tab
2. คลิกที่ workflow run ล่าสุด (ชื่อ "Deploy to GitHub Pages")
3. ดู progress ของ build และ deploy jobs:
   - **Build job**: ติดตั้ง dependencies, build app, upload artifact
   - **Deploy job**: deploy ไปยัง GitHub Pages

### ขั้นตอนที่ 10.2: รอ Deployment เสร็จ

- ใช้เวลาประมาณ 3-5 นาที
- เมื่อเสร็จจะเห็น ✅ สีเขียวทั้ง 2 jobs


## บันทึกรูปผลการ Deploy
```bash
# บันทึกรูปผลการ Deploy ที่นี่
```

### ขั้นตอนที่ 10.3: หา URL ของ Website

วิธีที่ 1: ใน Actions workflow
- คลิกที่ **deploy** job
- ดู URL ใน logs: `https://your-username.github.io/todo-frontend/`

วิธีที่ 2: ใน Settings → Pages
- จะแสดง "Your site is live at..."

### ขั้นตอนที่ 10.4: เปิดและทดสอบ Website

เปิดเบราว์เซอร์ไปที่:
```
https://your-username.github.io/todo-frontend/
```

ทดสอบว่า:
- ✅ หน้าเว็บโหลดได้
- ✅ UI แสดงถูกต้องและสวยงาม
- ✅ API Status เป็น "Connected"
- ✅ สามารถเพิ่ม Todo ได้
- ✅ สามารถ Toggle Complete ได้
- ✅ สามารถลบ Todo ได้
- ✅ Statistics แสดงถูกต้อง

---
## บันทึกรูปผลการรันหน้า Front-end
```bash
# บันทึกรูปผลการรันหน้า Front-end ที่นี่
```

## ส่วนที่ 11: Troubleshooting

### ปัญหา 1: CORS Error

**อาการ**:
```
Access to XMLHttpRequest has been blocked by CORS policy
```

**วิธีแก้**:
1. ตรวจสอบ Backend CORS configuration
2. ตรวจสอบว่าเพิ่ม GitHub Pages domain ใน allowed origins
3. Redeploy Backend
4. ลอง hard refresh (Ctrl+Shift+R หรือ Cmd+Shift+R)

### ปัญหา 2: API Status แสดง "Disconnected"

**วิธีแก้**:
1. ตรวจสอบว่า Backend ทำงานอยู่: เปิด `https://your-backend.onrender.com/api/health`
2. ตรวจสอบ API URL ใน workflow file ว่าถูกต้อง
3. ดู Console ในเบราว์เซอร์ (F12) เพื่อดู error messages

### ปัญหา 3: ปุ่ม Add Todo ไม่ทำงาน

**วิธีแก้**:
1. เปิด Console (F12) ดู error
2. ตรวจสอบว่า API URL ถูกต้อง
3. ลองพิมพ์อะไรก็ได้ใน Title field แล้วกดปุ่ม
4. ถ้ายังไม่ได้ ตรวจสอบ Network tab เพื่อดู API request/response

### ปัญหา 4: Build Failed ใน GitHub Actions

**วิธีแก้**:
1. ดู logs ใน Actions tab
2. ตรวจสอบว่า `package.json` มี dependencies ครบ
3. ตรวจสอบว่า build ผ่านใน local (`npm run build`)
4. ตรวจสอบ syntax errors ใน code

### ปัญหา 5: 404 Not Found หลัง Deploy

**วิธีแก้**:
1. ตรวจสอบว่ามีไฟล์ `public/.nojekyll`
2. ตรวจสอบ `next.config.js` มี `output: 'export'`
3. Redeploy โดย push commit ใหม่

---

## ส่วนที่ 12: การปรับแต่งเพิ่มเติม

### ขั้นตอนที่ 12.1: เปลี่ยนสีธีม

แก้ไข `src/app/globals.css` และ components:

```css
/* เปลี่ยนจาก purple-pink เป็น blue-green */
.bg-gradient-to-r.from-purple-600.to-pink-600 {
  background: linear-gradient(to right, #0ea5e9, #10b981);
}
```
---

## ส่วนที่ 13: สรุปและ Architecture

### Architecture Overview

```
┌─────────────────────────────────────────┐
│  GitHub Repository (todo-frontend)      │
│  ├── src/                               │
│  │   ├── app/ (pages)                   │
│  │   ├── components/ (UI components)    │
│  │   └── lib/ (API layer)               │
│  └── .github/workflows/ (CI/CD)         │
└────────────┬────────────────────────────┘
             │
             │ Push to main
             ▼
┌─────────────────────────────────────────┐
│  GitHub Actions                         │
│  1. npm install                         │
│  2. npm run lint                        │
│  3. npm run build → out/                │
│  4. Deploy to GitHub Pages              │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  GitHub Pages (Static Hosting)          │
│  https://username.github.io/todo-frontend│
│  - HTML, CSS, JS files                  │
│  - Free, Fast CDN                       │
│  - Automatic SSL                        │
└────────────┬────────────────────────────┘
             │
             │ API Calls (fetch/axios)
             ▼
┌─────────────────────────────────────────┐
│  Render Backend (Flask API)             │
│  https://flask-todo-app.onrender.com    │
│  ├── REST API Endpoints                 │
│  ├── CORS enabled for GitHub Pages      │
│  └── PostgreSQL Database                │
└─────────────────────────────────────────┘
```

### Project Structure Summary

```
todo-frontend/
├── .github/
│   └── workflows/
│       └── deploy.yml          # CI/CD configuration
├── public/
│   └── .nojekyll              # Disable Jekyll processing
├── src/
│   ├── app/
│   │   ├── page.js            # Main page component
│   │   ├── layout.js          # Root layout
│   │   └── globals.css        # Global styles
│   ├── components/
│   │   ├── AddTodo.jsx        # Form component
│   │   ├── TodoList.jsx       # List component
│   │   └── TodoStats.jsx      # Statistics component
│   └── lib/
│       └── api.js             # API client
├── .env.local                 # Local environment variables
├── next.config.js             # Next.js configuration
├── tailwind.config.js         # Tailwind configuration
├── postcss.config.js          # PostCSS configuration
└── package.json               # Dependencies
```

### Key Features

| Feature | Description |
|---------|-------------|
| 🎨 **Modern UI** | Gradient colors, animations, responsive design |
| ⚡ **Fast** | Static site generation, CDN delivery |
| 🔄 **CI/CD** | Automatic deployment on git push |
| 🆓 **Free** | GitHub Pages (100% free hosting) |
| 🔒 **Secure** | HTTPS enabled by default |
| 📱 **Responsive** | Works on mobile, tablet, desktop |
| ♿ **Accessible** | Semantic HTML, keyboard navigation |
| 🎯 **SEO Friendly** | Static HTML, meta tags |

---

## ส่วนที่ 14: Checklist การทดลอง

### Pre-deployment Checklist

- [ ] Node.js 18+ ติดตั้งแล้ว
- [ ] Git ติดตั้งแล้ว
- [ ] GitHub Account พร้อม
- [ ] Backend API ทำงานปกติ
- [ ] CORS ตั้งค่าถูกต้อง

### Development Checklist

- [ ] สร้างโปรเจกต์ Next.js
- [ ] ติดตั้ง dependencies ครบ
- [ ] สร้าง API layer (`src/lib/api.js`)
- [ ] สร้าง components ทั้ง 3 ตัว
- [ ] สร้าง main page
- [ ] แก้ไข styling
- [ ] ทดสอบ local ผ่าน

### Deployment Checklist

- [ ] สร้าง GitHub repository
- [ ] สร้าง workflow file
- [ ] ตั้งค่า GitHub Pages
- [ ] ตั้งค่า workflow permissions
- [ ] อัพเดท API URL ใน workflow
- [ ] Push code ไป GitHub
- [ ] Workflow รันสำเร็จ
- [ ] Website เข้าถึงได้
- [ ] ทดสอบ features ครบ

### Testing Checklist

- [ ] เปิดหน้าเว็บได้
- [ ] API Status เป็น "Connected"
- [ ] เพิ่ม Todo ได้
- [ ] ลบ Todo ได้
- [ ] Statistics แสดงถูกต้อง


---

## ส่วนที่ 15: คำถามท้ายการทดลอง

1. **CI/CD Pipeline**: อธิบายขั้นตอนใน GitHub Actions workflow
2. **CORS**: ทำไม Backend ต้อง enable CORS สำหรับ Frontend


## ส่วนที่ 16: แหล่งข้อมูลเพิ่มเติม

### Documentation

- [Next.js Documentation](https://nextjs.org/docs)
- [React Documentation](https://react.dev)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [GitHub Pages](https://docs.github.com/en/pages)
- [GitHub Actions](https://docs.github.com/en/actions)

### Tutorials

- [Next.js Learn](https://nextjs.org/learn)
- [React Tutorial](https://react.dev/learn)
- [Tailwind CSS Tutorial](https://tailwindcss.com/docs/installation)

### Tools

- [VS Code](https://code.visualstudio.com/) - Code Editor
- [Chrome DevTools](https://developer.chrome.com/docs/devtools/) - Debugging
- [Postman](https://www.postman.com/) - API Testing
- [Figma](https://www.figma.com/) - UI Design

---

## สรุป
**URL ของนักศึกษาคือ**:
- Frontend: `https://your-username.github.io/todo-frontend/`
- Backend: `https://your-backend.onrender.com`

---

