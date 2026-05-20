#Отлично! Давайте добавим **JSON Server** (как простой бэкенд) и **SQLite** (как легкую базу данных).

## ЧАСТЬ 1: Установка JSON Server (простой бэкенд)

### Шаг 1: Установите JSON Server

```powershell
cd C:\курсач\survey-app
npm install -D json-server
```

### Шаг 2: Создайте файл **db.json** в корне проекта

```json
{
  "users": [
    {
      "id": 1,
      "username": "admin",
      "email": "admin@example.com",
      "password": "admin123",
      "role": "admin",
      "createdAt": "2024-01-01T00:00:00.000Z",
      "lastLogin": null,
      "actionsCount": 0
    }
  ],
  "clients": [
    {
      "id": 1,
      "name": "Администратор",
      "email": "admin@example.com",
      "phone": "",
      "tags": "admin",
      "surveysCompleted": 0,
      "testsPassed": 0,
      "history": [],
      "createdAt": "2024-01-01T00:00:00.000Z",
      "userId": 1
    }
  ],
  "surveys": [
    {
      "id": 1,
      "title": "Оценка удовлетворенности",
      "description": "Расскажите о своем опыте",
      "status": "active",
      "questions": [
        { "id": 1, "text": "Как вы оцениваете наш сервис?", "type": "scale", "isRequired": true },
        { "id": 2, "text": "Что можно улучшить?", "type": "text", "isRequired": false },
        { "id": 3, "text": "Порекомендуете ли вы нас друзьям?", "type": "choice", "options": ["Да", "Нет", "Возможно"], "isRequired": true }
      ],
      "responsesCount": 0,
      "createdBy": 1,
      "createdAt": "2024-01-01T00:00:00.000Z"
    },
    {
      "id": 2,
      "title": "UX Тестирование",
      "description": "Оцените удобство интерфейса",
      "status": "active",
      "questions": [
        { "id": 1, "text": "Насколько удобен интерфейс?", "type": "scale", "isRequired": true },
        { "id": 2, "text": "Что бы вы изменили?", "type": "text", "isRequired": false }
      ],
      "responsesCount": 0,
      "createdBy": 1,
      "createdAt": "2024-01-01T00:00:00.000Z"
    }
  ],
  "responses": [],
  "userActions": []
}
```

### Шаг 3: Обновите **package.json** (добавьте скрипты)

```json
"scripts": {
  "dev": "vite",
  "server": "json-server --watch db.json --port 5000 --delay 300",
  "start:all": "npm run server & npm run dev"
}
```

### Шаг 4: Обновите **src/services/api.js** (для работы с JSON Server)

```javascript
import axios from 'axios'

// Определяем базовый URL в зависимости от окружения
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:5000'

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
})

// Добавляем токен к запросам
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('auth_token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

export default api
```

### Шаг 5: Обновите **src/hooks/useAuth.jsx** (для работы с JSON Server)

```jsx
import { useState, useEffect, createContext, useContext } from 'react'
import api from '../services/api'
import toast from 'react-hot-toast'

const AuthContext = createContext()

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const token = localStorage.getItem('auth_token')
    const savedUser = localStorage.getItem('current_user')
    if (token && savedUser) {
      setUser(JSON.parse(savedUser))
    }
    setLoading(false)
  }, [])

  // Регистрация
  const register = async (username, email, password) => {
    try {
      // Проверяем существующих пользователей через API
      const usersRes = await api.get('/users')
      const users = usersRes.data
      
      if (users.find(u => u.username === username)) {
        return { success: false, error: 'Пользователь уже существует' }
      }
      if (users.find(u => u.email === email)) {
        return { success: false, error: 'Email уже используется' }
      }
      if (password.length < 4) {
        return { success: false, error: 'Пароль минимум 4 символа' }
      }
      
      // Создаем пользователя
      const newUser = {
        id: Date.now(),
        username,
        email,
        password,
        role: 'user',
        createdAt: new Date().toISOString(),
        lastLogin: null,
        actionsCount: 0
      }
      
      await api.post('/users', newUser)
      
      // Создаем клиента
      const clientsRes = await api.get('/clients')
      const clients = clientsRes.data
      const newClient = {
        id: Date.now(),
        name: username,
        email: email,
        phone: '',
        tags: 'Новый пользователь',
        surveysCompleted: 0,
        testsPassed: 0,
        history: [],
        createdAt: new Date().toISOString(),
        userId: newUser.id
      }
      
      await api.post('/clients', newClient)
      
      // Добавляем действие
      await api.post('/userActions', {
        userId: newUser.id,
        action: 'register',
        details: { email },
        timestamp: new Date().toISOString()
      })
      
      // Автоматический вход
      setUser(newUser)
      localStorage.setItem('auth_token', 'token_' + newUser.id)
      localStorage.setItem('current_user', JSON.stringify(newUser))
      
      return { success: true, user: newUser }
    } catch (error) {
      console.error('Registration error:', error)
      return { success: false, error: 'Ошибка сервера' }
    }
  }

  // Вход
  const login = async (username, password) => {
    try {
      const usersRes = await api.get('/users')
      const users = usersRes.data
      const user = users.find(u => u.username === username && u.password === password)
      
      if (user) {
        setUser(user)
        localStorage.setItem('auth_token', 'token_' + user.id)
        localStorage.setItem('current_user', JSON.stringify(user))
        
        await api.post('/userActions', {
          userId: user.id,
          action: 'login',
          details: {},
          timestamp: new Date().toISOString()
        })
        
        return { success: true, user }
      }
      return { success: false, error: 'Неверное имя или пароль' }
    } catch (error) {
      return { success: false, error: 'Ошибка сервера' }
    }
  }

  // Выход
  const logout = async () => {
    if (user) {
      try {
        await api.post('/userActions', {
          userId: user.id,
          action: 'logout',
          details: {},
          timestamp: new Date().toISOString()
        })
      } catch (error) {
        console.error('Logout error:', error)
      }
    }
    setUser(null)
    localStorage.removeItem('auth_token')
    localStorage.removeItem('current_user')
  }

  // Добавление действия пользователя
  const addUserAction = async (userId, action, details = {}) => {
    try {
      await api.post('/userActions', {
        userId,
        action,
        details,
        timestamp: new Date().toISOString()
      })
    } catch (error) {
      console.error('Add action error:', error)
    }
  }

  // Получение действий пользователя
  const getUserActions = async (userId = null) => {
    try {
      const res = await api.get('/userActions')
      let actions = res.data
      if (userId) {
        actions = actions.filter(a => a.userId === userId)
      }
      return actions.slice(0, 50)
    } catch (error) {
      console.error('Get actions error:', error)
      return []
    }
  }

  // Получение всех пользователей
  const getAllUsers = async () => {
    try {
      const res = await api.get('/users')
      return res.data
    } catch (error) {
      return []
    }
  }

  // Получение всех клиентов
  const getAllClients = async () => {
    try {
      const res = await api.get('/clients')
      return res.data
    } catch (error) {
      return []
    }
  }

  // Добавление клиента
  const addClient = async (clientData) => {
    try {
      const clientsRes = await api.get('/clients')
      const clients = clientsRes.data
      const newClient = {
        id: Date.now(),
        ...clientData,
        surveysCompleted: 0,
        history: [],
        createdAt: new Date().toISOString()
      }
      await api.post('/clients', newClient)
      await addUserAction(user?.id, 'add_client', { clientName: clientData.name })
      return { success: true }
    } catch (error) {
      return { success: false, error: 'Ошибка добавления' }
    }
  }

  // Удаление клиента
  const deleteClient = async (id, name) => {
    try {
      await api.delete(`/clients/${id}`)
      await addUserAction(user?.id, 'delete_client', { clientName: name })
      return { success: true }
    } catch (error) {
      return { success: false, error: 'Ошибка удаления' }
    }
  }

  return (
    <AuthContext.Provider value={{ 
      user, 
      loading, 
      login, 
      logout, 
      register,
      addUserAction,
      getUserActions,
      getAllUsers,
      getAllClients,
      addClient,
      deleteClient,
      isAuthenticated: !!user 
    }}>
      {children}
    </AuthContext.Provider>
  )
}

export const useAuth = () => useContext(AuthContext)
```

### Шаг 6: Запуск JSON Server и фронтенда

**Терминал 1 (запуск JSON Server):**
```powershell
cd C:\курсач\survey-app
npm run server
```

**Терминал 2 (запуск фронтенда):**
```powershell
cd C:\курсач\survey-app
npm run dev
```

---

## ЧАСТЬ 2: Установка SQLite (настоящая база данных)

### Шаг 1: Установите зависимости для SQLite (Node.js бэкенд)

```powershell
# Создайте папку для бэкенда
cd C:\курсач
mkdir survey-backend
cd survey-backend

# Инициализируйте проект
npm init -y

# Установите зависимости
npm install express sqlite3 sqlite cors bcryptjs jsonwebtoken
npm install -D nodemon
```

### Шаг 2: Создайте файл **server.js**

```javascript
const express = require('express')
const cors = require('cors')
const sqlite3 = require('sqlite3').verbose()
const { open } = require('sqlite')
const bcrypt = require('bcryptjs')
const jwt = require('jsonwebtoken')

const app = express()
app.use(cors())
app.use(express.json())

const JWT_SECRET = 'your-secret-key-2024'
let db

// Инициализация базы данных
async function initDB() {
  db = await open({
    filename: './survey.db',
    driver: sqlite3.Database
  })

  // Создание таблиц
  await db.exec(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      username TEXT UNIQUE NOT NULL,
      email TEXT UNIQUE NOT NULL,
      password TEXT NOT NULL,
      role TEXT DEFAULT 'user',
      created_at TEXT NOT NULL,
      last_login TEXT
    );

    CREATE TABLE IF NOT EXISTS clients (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT NOT NULL,
      phone TEXT,
      tags TEXT,
      surveys_completed INTEGER DEFAULT 0,
      tests_passed INTEGER DEFAULT 0,
      created_at TEXT NOT NULL,
      user_id INTEGER,
      FOREIGN KEY (user_id) REFERENCES users(id)
    );

    CREATE TABLE IF NOT EXISTS surveys (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      description TEXT,
      status TEXT DEFAULT 'active',
      questions TEXT,
      responses_count INTEGER DEFAULT 0,
      created_by INTEGER,
      created_at TEXT,
      FOREIGN KEY (created_by) REFERENCES users(id)
    );

    CREATE TABLE IF NOT EXISTS responses (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      survey_id INTEGER,
      client_id INTEGER,
      answers TEXT,
      score REAL,
      submitted_at TEXT,
      FOREIGN KEY (survey_id) REFERENCES surveys(id),
      FOREIGN KEY (client_id) REFERENCES clients(id)
    );

    CREATE TABLE IF NOT EXISTS user_actions (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER,
      action TEXT,
      details TEXT,
      timestamp TEXT,
      FOREIGN KEY (user_id) REFERENCES users(id)
    );
  `)

  // Добавление админа по умолчанию
  const adminExists = await db.get('SELECT * FROM users WHERE username = ?', ['admin'])
  if (!adminExists) {
    const hashedPassword = await bcrypt.hash('admin123', 10)
    await db.run(
      'INSERT INTO users (username, email, password, role, created_at) VALUES (?, ?, ?, ?, ?)',
      ['admin', 'admin@example.com', hashedPassword, 'admin', new Date().toISOString()]
    )
    console.log('Admin user created')
  }
}

// Middleware для проверки JWT
const authenticate = async (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1]
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' })
  }
  try {
    const decoded = jwt.verify(token, JWT_SECRET)
    req.userId = decoded.userId
    next()
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' })
  }
}

// API endpoints

// Регистрация
app.post('/api/auth/register', async (req, res) => {
  const { username, email, password } = req.body
  
  try {
    const existingUser = await db.get('SELECT * FROM users WHERE username = ? OR email = ?', [username, email])
    if (existingUser) {
      return res.status(400).json({ error: 'User already exists' })
    }
    
    const hashedPassword = await bcrypt.hash(password, 10)
    const now = new Date().toISOString()
    
    const result = await db.run(
      'INSERT INTO users (username, email, password, created_at) VALUES (?, ?, ?, ?)',
      [username, email, hashedPassword, now]
    )
    
    // Автоматическое создание клиента
    await db.run(
      'INSERT INTO clients (name, email, created_at, user_id) VALUES (?, ?, ?, ?)',
      [username, email, now, result.lastID]
    )
    
    const token = jwt.sign({ userId: result.lastID, username }, JWT_SECRET, { expiresIn: '24h' })
    res.json({ token, user: { id: result.lastID, username, email, role: 'user' } })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Вход
app.post('/api/auth/login', async (req, res) => {
  const { username, password } = req.body
  
  try {
    const user = await db.get('SELECT * FROM users WHERE username = ?', [username])
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' })
    }
    
    const valid = await bcrypt.compare(password, user.password)
    if (!valid) {
      return res.status(401).json({ error: 'Invalid credentials' })
    }
    
    await db.run('UPDATE users SET last_login = ? WHERE id = ?', [new Date().toISOString(), user.id])
    
    const token = jwt.sign({ userId: user.id, username: user.username }, JWT_SECRET, { expiresIn: '24h' })
    res.json({ token, user: { id: user.id, username: user.username, email: user.email, role: user.role } })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Получить клиентов
app.get('/api/clients', authenticate, async (req, res) => {
  try {
    const clients = await db.all('SELECT * FROM clients ORDER BY id DESC')
    res.json(clients)
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Добавить клиента
app.post('/api/clients', authenticate, async (req, res) => {
  const { name, email, phone, tags } = req.body
  try {
    const now = new Date().toISOString()
    const result = await db.run(
      'INSERT INTO clients (name, email, phone, tags, created_at, user_id) VALUES (?, ?, ?, ?, ?, ?)',
      [name, email, phone, tags, now, req.userId]
    )
    res.json({ id: result.lastID })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Удалить клиента
app.delete('/api/clients/:id', authenticate, async (req, res) => {
  try {
    await db.run('DELETE FROM clients WHERE id = ?', [req.params.id])
    res.json({ success: true })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Получить опросы
app.get('/api/surveys', async (req, res) => {
  try {
    const surveys = await db.all('SELECT * FROM surveys ORDER BY id DESC')
    res.json(surveys)
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Создать опрос
app.post('/api/surveys', authenticate, async (req, res) => {
  const { title, description, questions } = req.body
  try {
    const now = new Date().toISOString()
    const result = await db.run(
      'INSERT INTO surveys (title, description, questions, created_by, created_at) VALUES (?, ?, ?, ?, ?)',
      [title, description, JSON.stringify(questions), req.userId, now]
    )
    res.json({ id: result.lastID })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Получить действия пользователя
app.get('/api/userActions', authenticate, async (req, res) => {
  try {
    const actions = await db.all('SELECT * FROM user_actions WHERE user_id = ? ORDER BY id DESC LIMIT 50', [req.userId])
    res.json(actions)
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Добавить действие
app.post('/api/userActions', authenticate, async (req, res) => {
  const { action, details } = req.body
  try {
    await db.run(
      'INSERT INTO user_actions (user_id, action, details, timestamp) VALUES (?, ?, ?, ?)',
      [req.userId, action, JSON.stringify(details), new Date().toISOString()]
    )
    res.json({ success: true })
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
})

// Запуск сервера
initDB().then(() => {
  app.listen(5000, () => {
    console.log('Server running on http://localhost:5000')
    console.log('Database: survey.db')
  })
})
```

### Шаг 3: Создайте файл **package.json** для бэкенда

```json
{
  "name": "survey-backend",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "sqlite3": "^5.1.6",
    "sqlite": "^5.1.1",
    "cors": "^2.8.5",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### Шаг 4: Обновите **.env** файл во фронтенде

Создайте файл **.env** в папке фронтенда:

```env
VITE_API_URL=http://localhost:5000
```

### Шаг 5: Запуск полного приложения

**Терминал 1 (Запуск SQLite бэкенда):**
```powershell
cd C:\курсач\survey-backend
npm run dev
```

**Терминал 2 (Запуск фронтенда):**
```powershell
cd C:\курсач\survey-app
npm run dev
```

---

## Итог: что у вас теперь есть

| Компонент | Состояние | Порт |
|-----------|-----------|------|
| **Фронтенд (React)** | ✅ Установлен | 5173 |
| **JSON Server** | ✅ Установлен (опционально) | 5000 |
| **SQLite База данных** | ✅ Установлена | - |
| **Node.js Бэкенд** | ✅ Установлен | 5000 |

## Запуск (выберите один вариант):

### Вариант 1: Только фронтенд + JSON Server (простой)
```powershell
# Терминал 1
cd C:\курсач\survey-app
npm run server

# Терминал 2
cd C:\курсач\survey-app
npm run dev
```

### Вариант 2: Фронтенд + SQLite бэкенд (настоящая БД)
```powershell
# Терминал 1
cd C:\курсач\survey-backend
npm run dev

# Терминал 2
cd C:\курсач\survey-app
npm run dev
```
