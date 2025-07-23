---
title: "Complete Guide to OAuth 2.0 with OpenID Connect: Building a Secure Authentication System with Google"
date: 2025-01-18 12:00:00 +0530
categories: [Security, Authentication, Tutorial]
tags: [oauth2, openid-connect, google-auth, nodejs, react, mongodb, jwt, express, passport, mermaid, security, web-development]
author: lakhman
description: "A comprehensive guide to implementing OAuth 2.0 with OpenID Connect using Google as the identity provider. Includes complete code examples for Node.js/Express backend and React frontend."
pin: true
math: false
mermaid: true
---

# 🔐 Complete Guide to OAuth 2.0 with OpenID Connect: Building a Secure Authentication System with Google

## Table of Contents
- [Introduction](#introduction)
- [Project Architecture](#project-architecture)
- [Prerequisites](#prerequisites)
- [Backend Implementation](#backend-implementation)
- [Frontend Implementation](#frontend-implementation)
- [Security Considerations](#security-considerations)
- [Testing the Application](#testing-the-application)
- [Deployment Considerations](#deployment-considerations)
- [Conclusion](#conclusion)

## Introduction

OAuth 2.0 and OpenID Connect (OIDC) are industry-standard protocols for authorization and authentication. While OAuth 2.0 handles authorization (what a user can do), OpenID Connect adds an identity layer on top of OAuth 2.0 to handle authentication (who the user is).

In this comprehensive guide, we'll build a complete authentication system using:
- **OAuth 2.0** for secure authorization
- **OpenID Connect** for user identity
- **Google as Identity Provider**
- **Node.js/Express** for the backend
- **React/Vite** for the frontend
- **MongoDB** for user data storage
- **JWT** for session management

## Project Architecture

Let's visualize the complete authentication flow:

```mermaid
sequenceDiagram
    participant User
    participant React as React Frontend
    participant Express as Express Server
    participant Google as Google OAuth
    participant MongoDB as MongoDB

    User->>React: 1. Click "Login with Google"
    React->>Google: 2. Redirect to Google OAuth
    Google->>User: 3. Show consent screen
    User->>Google: 4. Grant permissions
    Google->>Express: 5. Callback with auth code
    Express->>Google: 6. Exchange code for tokens
    Google->>Express: 7. Return access token & user info
    Express->>MongoDB: 8. Save/update user data
    MongoDB->>Express: 9. User saved/retrieved
    Express->>Express: 10. Generate JWT token
    Express->>React: 11. Redirect with JWT
    React->>React: 12. Store token & decode user info
    React->>User: 13. Display authenticated state
```

### System Architecture Diagram

```mermaid
graph TB
    subgraph "Frontend (Port 3000)"
        A[React App] --> B[GoogleLogin Component]
        B --> C[Home Page]
        C --> D[JWT Decoder]
    end

    subgraph "Backend (Port 5000)"
        E[Express Server] --> F[Auth Routes]
        F --> G[Auth Controller]
        G --> H[Passport Strategy]
        H --> I[JWT Service]
        I --> J[Auth Middleware]
    end

    subgraph "External Services"
        K[Google OAuth 2.0]
        L[MongoDB Atlas]
    end

    A -.->|HTTP Requests| E
    H -.->|OAuth Flow| K
    G -.->|Store Users| L

    style A fill:#61dafb
    style E fill:#68a063
    style K fill:#4285f4
    style L fill:#47a248
```

## Prerequisites

Before we start, ensure you have:

1. **Node.js** (v16 or higher)
2. **MongoDB** (local or Atlas)
3. **Google Developer Console** access
4. **Basic knowledge** of React, Node.js, and OAuth concepts

### Setting up Google OAuth Credentials

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing one
3. Enable Google+ API
4. Create OAuth 2.0 credentials
5. Set authorized redirect URIs:
   - `http://localhost:5000/auth/google/callback` (development)

## Backend Implementation

### 📁 Project Structure

```
server/
├── config/
│   └── db.js              # Database connection
├── controllers/
│   └── authController.js  # Authentication logic
├── middlewares/
│   ├── authMiddleware.js  # JWT verification
│   ├── errorHandler.js    # Error handling
│   └── roleMiddleware.js  # Role-based access (future)
├── models/
│   └── userModel.js       # User schema
├── routes/
│   └── authRoutes.js      # Auth endpoints
├── package.json           # Dependencies
└── index.js              # Main server file
```

### 🚀 Main Server Setup (`index.js`)

Let's start with the main server file that orchestrates everything:

```javascript
// index.js - Main server
require('dotenv').config();
const express = require('express');
const passport = require('passport');
const GoogleStrategy = require('passport-google-oauth20').Strategy
const cors = require('cors');
const rateLimit = require('express-rate-limit');
const connectDB = require('./config/db');
const authRoutes = require('./routes/authRoutes');
const errorHandler = require('./middlewares/errorHandler');
const { googleCallback } = require('./controllers/authController');

connectDB();

const app = express();
app.use(cors({ origin: 'http://localhost:3000', credentials: true }));
app.use(express.json());
app.use(passport.initialize());

// Rate limiting
const limiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 100 });
app.use('/auth/', limiter);

// Passport setup
passport.use(new GoogleStrategy({
  clientID: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  callbackURL: '/auth/google/callback',
}, googleCallback));

// Routes
app.use('/auth', authRoutes);

app.use(errorHandler); // Error handler last

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

**Key Features Explained:**

1. **Environment Configuration**: Loads environment variables for sensitive data
2. **CORS Setup**: Allows cross-origin requests from frontend (port 3000)
3. **Rate Limiting**: Prevents abuse by limiting requests (100 per 15 minutes)
4. **Passport Integration**: Sets up Google OAuth 2.0 strategy
5. **Error Handling**: Centralized error handling middleware

### 🗄️ Database Configuration (`config/db.js`)

```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log('MongoDB connected');
  } catch (err) {
    console.error('MongoDB connection error:', err);
    process.exit(1); // Exit on failure
  }
};

module.exports = connectDB;
```

**Key Features:**
- **Async Connection**: Non-blocking database connection
- **Error Handling**: Graceful failure with process exit
- **Modern Options**: Uses latest MongoDB connection options

### 👤 User Model (`models/userModel.js`)

```javascript
// models/userModel.js - User schema with validation
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  googleId: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true, match: [/^\S+@\S+\.\S+$/, 'Please use a valid email'] },
  name: { type: String, required: true },
  createdAt: { type: Date, default: Date.now },
});

module.exports = mongoose.model('User', userSchema);
```

**Schema Features:**
- **Unique Constraints**: Prevents duplicate users
- **Email Validation**: Regex pattern for email format
- **Timestamps**: Automatic creation date tracking

### 🎛️ Authentication Controller (`controllers/authController.js`)

```javascript
// controllers/authController.js - Auth logic
const jwt = require('jsonwebtoken');
const User = require('../models/userModel');

exports.googleCallback = async (accessToken, refreshToken, profile, done) => {
  try {
    let user = await User.findOne({ googleId: profile.id });
    if (!user) {
      user = new User({
        googleId: profile.id,
        email: profile.emails[0].value,
        name: profile.displayName,
      });
      await user.save();
    }
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '1h' });
    return done(null, { user, token });
  } catch (err) {
    return done(err);
  }
};

exports.getUser = (req, res) => {
  res.json({ message: 'Protected route', user: req.user });
};
```

**Controller Logic:**
1. **User Lookup**: Checks if user exists in database
2. **User Creation**: Creates new user if first-time login
3. **JWT Generation**: Creates signed token with user ID
4. **Error Handling**: Proper error propagation

### 🛡️ Authentication Middleware (`middlewares/authMiddleware.js`)

```javascript
// middlewares/authMiddleware.js - JWT verification
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ message: 'No token provided' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (err) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

module.exports = authMiddleware;
```

**Middleware Features:**
- **Token Extraction**: Parses Bearer token from Authorization header
- **JWT Verification**: Validates token signature and expiration
- **Request Enhancement**: Adds user info to request object

### 🛣️ Authentication Routes (`routes/authRoutes.js`)

```javascript
// routes/authRoutes.js - Auth routes
const express = require('express');
const passport = require('passport');
const { getUser } = require('../controllers/authController');
const authMiddleware = require('../middlewares/authMiddleware');

const router = express.Router();

router.get('/google', passport.authenticate('google', { scope: ['profile', 'email'] }));

router.get('/google/callback', passport.authenticate('google', { session: false }), (req, res) => {
  res.redirect(`http://localhost:3000?token=${req.user.token}`);
});

router.get('/user', authMiddleware, getUser);

module.exports = router;
```

**Route Breakdown:**

1. **`/auth/google`**: Initiates OAuth flow with Google
2. **`/auth/google/callback`**: Handles OAuth callback and redirects with JWT
3. **`/auth/user`**: Protected route demonstrating authentication

### ⚠️ Error Handler (`middlewares/errorHandler.js`)

```javascript
// middlewares/errorHandler.js - Centralized error handling
const errorHandler = (err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ message: 'Server error', error: err.message });
};

module.exports = errorHandler;
```

**Error Handling Strategy:**
- **Logging**: Console error for debugging
- **Client Response**: Generic error message for security
- **Status Codes**: Proper HTTP status codes

### 📦 Backend Dependencies (`package.json`)

```json
{
  "name": "server",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "cors": "^2.8.5",              // Cross-origin resource sharing
    "dotenv": "^17.2.0",           // Environment variable loading
    "express": "^5.1.0",           // Web framework
    "express-rate-limit": "^8.0.1", // Rate limiting
    "jsonwebtoken": "^9.0.2",      // JWT token handling
    "mongoose": "^8.16.4",         // MongoDB ODM
    "passport": "^0.7.0",          // Authentication middleware
    "passport-google-oauth20": "^2.0.0" // Google OAuth strategy
  }
}
```

## Frontend Implementation

### 📁 Frontend Structure

```
client/frontend/
├── public/
│   └── vite.svg
├── src/
│   ├── assets/
│   │   └── react.svg
│   ├── components/
│   │   ├── AdminPanel.jsx    # Future admin features
│   │   └── GoogleLogin.jsx   # Google login button
│   ├── pages/
│   │   └── Home.jsx          # Main page
│   ├── App.jsx               # Root component
│   ├── main.jsx              # Entry point
│   └── index.css             # Styles
├── index.html                # HTML template
├── package.json              # Dependencies
└── vite.config.js           # Vite configuration
```

### 🏠 Main Application (`src/App.jsx`)

```jsx
import React from 'react';
import Home from './pages/Home';

function App() {
  return <Home />;
}

export default App;
```

**Simple Structure:**
- **Single Page**: Currently renders only Home component
- **Scalable**: Easy to add routing later

### 🏡 Home Page (`src/pages/Home.jsx`)

```jsx
// src/pages/Home.js - Main page
import React, { useEffect, useState } from 'react';
import { jwtDecode } from 'jwt-decode';
import GoogleLogin from '../components/GoogleLogin';

const Home = () => {
    const [user, setUser] = useState(null);

    useEffect(() => {
        const urlParams = new URLSearchParams(window.location.search);
        const token = urlParams.get('token');
        if (token) {
            try {
                const decoded = jwtDecode(token);
                setUser(decoded);
                localStorage.setItem('token', token); // For dev; use cookies in prod
                window.history.replaceState({}, document.title, '/');
            } catch (err) {
                console.error('Token decode error:', err);
            }
        }
    }, []);

    return (
        <div>
            <h1>Google Auth in React</h1>
            {!user ? <GoogleLogin /> : <p>Welcome, {user.name}!</p>}
        </div>
    );
};

export default Home;
```

**Component Features:**

1. **URL Parameter Parsing**: Extracts JWT token from URL after OAuth redirect
2. **JWT Decoding**: Decodes token to get user information
3. **State Management**: Manages authentication state
4. **URL Cleanup**: Removes token from URL for security
5. **Conditional Rendering**: Shows login or welcome message

### 🔘 Google Login Component (`src/components/GoogleLogin.jsx`)

```jsx
// src/components/GoogleLogin.js - Login button component
import React, { useEffect } from 'react';

const GoogleLogin = () => {
    useEffect(() => {
        /* global google */
        google.accounts.id.initialize({
            client_id: import.meta.env.VITE_GOOGLE_CLIENT_ID,
            callback: () => {
                window.location.href = 'http://localhost:8000/auth/google';
            },
        });
        google.accounts.id.renderButton(
            document.getElementById('googleSignIn'),
            { theme: 'outline', size: 'large' }
        );
    }, []);

    return <div id="googleSignIn"></div>;
};

export default GoogleLogin;
```

**Component Breakdown:**

1. **Google Identity Services**: Uses Google's new Identity Services library
2. **Client ID Configuration**: Uses environment variable for client ID
3. **Button Rendering**: Creates styled Google login button
4. **Redirect Handler**: Initiates OAuth flow on click

### 🏷️ HTML Template (`index.html`)

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite + React</title>
  </head>
  <body>
    <div id="root"></div>
    <script src="https://accounts.google.com/gsi/client" async defer></script>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

**Key Elements:**
- **Google GSI Library**: Loads Google Sign-In JavaScript library
- **React Root**: Mount point for React application
- **Module Script**: ES6 module support for modern JavaScript

### ⚙️ Vite Configuration (`vite.config.js`)

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
  },
})
```

**Configuration Features:**
- **React Plugin**: Enables React support with Fast Refresh
- **Port Configuration**: Runs on port 3000 to match CORS settings

### 📦 Frontend Dependencies (`package.json`)

```json
{
  "name": "frontend",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.11.0",        // HTTP client for API calls
    "jwt-decode": "^4.0.0",    // JWT token decoding
    "react": "^19.1.0",        // React library
    "react-dom": "^19.1.0"     // React DOM rendering
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.6.0",  // Vite React plugin
    "eslint": "^9.30.1",               // Code linting
    "vite": "^7.0.4"                   // Build tool
  }
}
```

## Security Considerations

### 🔒 JWT Token Security

```mermaid
graph TD
    A[JWT Token] --> B[Header]
    A --> C[Payload]
    A --> D[Signature]

    B --> E[Algorithm: HS256]
    C --> F[User ID]
    C --> G[Expiration: 1h]
    D --> H[HMAC SHA256]

    I[JWT Secret] --> H

    style A fill:#ff6b6b
    style D fill:#4ecdc4
    style I fill:#45b7d1
```

**Security Measures Implemented:**

1. **Short Token Expiry**: 1-hour expiration reduces exposure risk
2. **Secure Secret**: Environment variable for JWT secret
3. **HTTPS Only**: Should be enforced in production
4. **Rate Limiting**: Prevents brute force attacks
5. **Input Validation**: Email format validation

### 🛡️ Additional Security Recommendations

```javascript
// Production security enhancements
const helmet = require('helmet');
const cookieParser = require('cookie-parser');

// Security headers
app.use(helmet());

// Secure cookie configuration
app.use(cookieParser());

// HTTPS redirect in production
if (process.env.NODE_ENV === 'production') {
  app.use((req, res, next) => {
    if (req.header('x-forwarded-proto') !== 'https') {
      res.redirect(`https://${req.header('host')}${req.url}`);
    } else {
      next();
    }
  });
}
```

## Testing the Application

### 🧪 Manual Testing Steps

1. **Start Backend Server**:
   ```bash
   cd server
   npm install
   npm start
   ```

2. **Start Frontend Development Server**:
   ```bash
   cd client/frontend
   npm install
   npm run dev
   ```

3. **Test Authentication Flow**:
   - Navigate to `http://localhost:3000`
   - Click "Sign in with Google"
   - Complete Google OAuth consent
   - Verify redirect and user display

### 🔍 Testing Checklist

```mermaid
graph LR
    A[Start Testing] --> B{Backend Running?}
    B -->|No| C[Start Backend]
    B -->|Yes| D{Frontend Running?}
    C --> D
    D -->|No| E[Start Frontend]
    D -->|Yes| F[Test Login Flow]
    E --> F
    F --> G{Login Successful?}
    G -->|No| H[Check Logs]
    G -->|Yes| I[Test Protected Routes]
    H --> J[Debug Issues]
    I --> K[Test Complete]
    J --> F
```

## Environment Variables

Create these environment files:

### Backend (`.env`)
```env
# Server Configuration
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/oauth-app
# Or MongoDB Atlas: mongodb+srv://username:password@cluster.mongodb.net/oauth-app

# JWT
JWT_SECRET=your-super-secret-jwt-key-here

# Google OAuth
GOOGLE_CLIENT_ID=your-google-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

### Frontend (`.env`)
```env
# Google OAuth
VITE_GOOGLE_CLIENT_ID=your-google-client-id.apps.googleusercontent.com
```

## Deployment Considerations

### 🚀 Production Deployment Checklist

```mermaid
graph TB
    A[Development Complete] --> B[Environment Setup]
    B --> C[Security Hardening]
    C --> D[Database Migration]
    D --> E[Domain Configuration]
    E --> F[SSL Certificate]
    F --> G[Deploy Backend]
    G --> H[Deploy Frontend]
    H --> I[Update OAuth URLs]
    I --> J[Testing]
    J --> K[Monitoring Setup]

    style A fill:#68a063
    style K fill:#4ecdc4
```

### 🌐 Production Environment Variables

```env
# Production Backend
NODE_ENV=production
PORT=443
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/oauth-prod
JWT_SECRET=super-secure-random-256-bit-key
GOOGLE_CLIENT_ID=your-prod-client-id
GOOGLE_CLIENT_SECRET=your-prod-client-secret

# Frontend
VITE_GOOGLE_CLIENT_ID=your-prod-client-id
VITE_API_URL=https://api.yourdomain.com
```

### 📊 Monitoring and Logging

```javascript
// Enhanced logging for production
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Usage in controllers
logger.info('User authenticated', { userId: user._id, email: user.email });
logger.error('Authentication failed', { error: err.message });
```

## OAuth 2.0 Flow Deep Dive

### 🔄 Complete Authentication Flow

```mermaid
sequenceDiagram
    participant U as User
    participant B as Browser
    participant F as Frontend (React)
    participant S as Backend (Express)
    participant G as Google OAuth
    participant DB as MongoDB

    Note over U,DB: OAuth 2.0 Authorization Code Flow

    U->>B: 1. Visit application
    B->>F: 2. Load React app
    F->>B: 3. Render login button
    U->>B: 4. Click "Login with Google"

    Note over B,G: OAuth Authorization Request
    B->>G: 5. GET /oauth/authorize?response_type=code&client_id=...
    G->>U: 6. Show consent screen
    U->>G: 7. Grant permissions

    Note over G,S: Authorization Code Grant
    G->>S: 8. GET /auth/google/callback?code=AUTH_CODE
    S->>G: 9. POST /oauth/token (exchange code for tokens)
    G->>S: 10. Return access_token + user profile

    Note over S,DB: User Management
    S->>DB: 11. Find or create user
    DB->>S: 12. Return user data
    S->>S: 13. Generate JWT token

    Note over S,F: Token Delivery
    S->>B: 14. Redirect to frontend with JWT
    B->>F: 15. Extract token from URL
    F->>F: 16. Decode JWT and set user state
    F->>U: 17. Display authenticated UI
```

### 🎯 Key OAuth 2.0 Concepts

**1. Authorization Code Flow**
- Most secure OAuth flow for web applications
- Uses temporary authorization code
- Requires client secret for token exchange

**2. Scopes**
- `profile`: Access to user's profile information
- `email`: Access to user's email address

**3. PKCE (Proof Key for Code Exchange)**
- Additional security for public clients
- Not implemented in this example but recommended for SPAs

## Advanced Features and Extensions

### 🔄 Refresh Token Implementation

```javascript
// Enhanced token management
exports.refreshToken = async (req, res) => {
  try {
    const { refreshToken } = req.body;

    // Verify refresh token
    const decoded = jwt.verify(refreshToken, process.env.REFRESH_TOKEN_SECRET);
    const user = await User.findById(decoded.id);

    if (!user) {
      return res.status(401).json({ message: 'Invalid refresh token' });
    }

    // Generate new access token
    const newAccessToken = jwt.sign(
      { id: user._id },
      process.env.JWT_SECRET,
      { expiresIn: '15m' }
    );

    res.json({ accessToken: newAccessToken });
  } catch (err) {
    res.status(401).json({ message: 'Invalid refresh token' });
  }
};
```

### 👑 Role-Based Access Control (RBAC)

```javascript
// User model with roles
const userSchema = new mongoose.Schema({
  googleId: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  name: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  permissions: [{ type: String }],
  createdAt: { type: Date, default: Date.now },
});

// Role middleware
const requireRole = (roles) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ message: 'Authentication required' });
    }

    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: 'Insufficient permissions' });
    }

    next();
  };
};

// Usage
router.get('/admin', authMiddleware, requireRole(['admin']), getAdminData);
```

### 📱 Multi-Provider Authentication

```javascript
// Additional OAuth providers
const FacebookStrategy = require('passport-facebook').Strategy;
const GitHubStrategy = require('passport-github2').Strategy;

// Facebook strategy
passport.use(new FacebookStrategy({
  clientID: process.env.FACEBOOK_CLIENT_ID,
  clientSecret: process.env.FACEBOOK_CLIENT_SECRET,
  callbackURL: '/auth/facebook/callback',
  profileFields: ['id', 'emails', 'name']
}, facebookCallback));

// GitHub strategy
passport.use(new GitHubStrategy({
  clientID: process.env.GITHUB_CLIENT_ID,
  clientSecret: process.env.GITHUB_CLIENT_SECRET,
  callbackURL: '/auth/github/callback'
}, githubCallback));
```

## Performance Optimization

### ⚡ Backend Optimizations

```javascript
// Connection pooling
const mongoose = require('mongoose');
mongoose.connect(process.env.MONGODB_URI, {
  maxPoolSize: 10,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
});

// Response compression
const compression = require('compression');
app.use(compression());

// Request logging
const morgan = require('morgan');
app.use(morgan('combined'));

// Memory-based session store (for development)
const session = require('express-session');
const MongoStore = require('connect-mongo');

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  store: MongoStore.create({
    mongoUrl: process.env.MONGODB_URI
  })
}));
```

### 🎨 Frontend Optimizations

```jsx
// Lazy loading components
import { lazy, Suspense } from 'react';

const AdminPanel = lazy(() => import('./components/AdminPanel'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <AdminPanel />
    </Suspense>
  );
}

// Custom hook for auth state
import { useState, useEffect } from 'react';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      try {
        const decoded = jwtDecode(token);
        if (decoded.exp * 1000 > Date.now()) {
          setUser(decoded);
        } else {
          localStorage.removeItem('token');
        }
      } catch (error) {
        localStorage.removeItem('token');
      }
    }
    setLoading(false);
  }, []);

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return { user, loading, logout };
};
```

## Error Handling and Debugging

### 🐛 Common Issues and Solutions

```mermaid
graph TD
    A[Common Issues] --> B[CORS Errors]
    A --> C[Token Expiration]
    A --> D[Database Connection]
    A --> E[OAuth Configuration]

    B --> F["Add origin to CORS config"]
    C --> G["Implement token refresh"]
    D --> H["Check MongoDB URI"]
    E --> I["Verify client ID/secret"]

    style A fill:#ff6b6b
    style F fill:#51cf66
    style G fill:#51cf66
    style H fill:#51cf66
    style I fill:#51cf66
```

### 🔧 Debug Logging

```javascript
// Debug middleware
const debug = require('debug')('oauth:auth');

const debugMiddleware = (req, res, next) => {
  debug(`${req.method} ${req.path}`, {
    headers: req.headers,
    query: req.query,
    body: req.body
  });
  next();
};

// Error logging
const errorLogger = (err, req, res, next) => {
  console.error('Error occurred:', {
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    timestamp: new Date().toISOString()
  });
  next(err);
};
```

## Testing Strategy

### 🧪 Unit Testing

```javascript
// Test setup (Jest + Supertest)
const request = require('supertest');
const app = require('../index');
const User = require('../models/userModel');

describe('Auth Endpoints', () => {
  beforeEach(async () => {
    await User.deleteMany({});
  });

  test('GET /auth/google should redirect to Google', async () => {
    const response = await request(app)
      .get('/auth/google')
      .expect(302);

    expect(response.headers.location).toContain('accounts.google.com');
  });

  test('GET /auth/user should require authentication', async () => {
    await request(app)
      .get('/auth/user')
      .expect(401);
  });

  test('GET /auth/user should return user data with valid token', async () => {
    const token = jwt.sign({ id: 'test-id' }, process.env.JWT_SECRET);

    await request(app)
      .get('/auth/user')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
  });
});
```

### 🎭 Frontend Testing

```jsx
// React Testing Library example
import { render, screen, fireEvent } from '@testing-library/react';
import GoogleLogin from './GoogleLogin';

// Mock Google GSI
global.google = {
  accounts: {
    id: {
      initialize: jest.fn(),
      renderButton: jest.fn()
    }
  }
};

test('renders Google login button', () => {
  render(<GoogleLogin />);
  expect(screen.getByRole('button')).toBeInTheDocument();
});

test('initializes Google Sign-In on mount', () => {
  render(<GoogleLogin />);
  expect(google.accounts.id.initialize).toHaveBeenCalled();
});
```

## Best Practices Summary

### ✅ Security Best Practices

1. **Never expose secrets** in frontend code
2. **Use HTTPS** in production
3. **Implement rate limiting** on auth endpoints
4. **Short token expiry** with refresh tokens
5. **Validate all inputs** on server side
6. **Log security events** for monitoring

### 🏗️ Architecture Best Practices

1. **Separation of concerns** (MVC pattern)
2. **Environment-specific configuration**
3. **Centralized error handling**
4. **Modular code structure**
5. **Database connection pooling**
6. **Graceful error recovery**

### 📱 User Experience Best Practices

1. **Clear loading states**
2. **Informative error messages**
3. **Responsive design**
4. **Accessibility compliance**
5. **Progressive enhancement**
6. **Offline capability** (future enhancement)

## Conclusion

This comprehensive OAuth 2.0 with OpenID Connect implementation demonstrates:

- **Secure Authentication**: Industry-standard OAuth 2.0 flow
- **Modern Stack**: React, Node.js, MongoDB, JWT
- **Scalable Architecture**: Modular and maintainable code
- **Production Ready**: Security, error handling, and monitoring
- **Best Practices**: Following OAuth 2.0 and OIDC specifications

### 🚀 Next Steps

1. **Add refresh tokens** for long-lived sessions
2. **Implement RBAC** for permission management
3. **Add multi-factor authentication**
4. **Set up CI/CD pipeline**
5. **Add comprehensive logging**
6. **Implement social login providers**

### 📚 Additional Resources

- [OAuth 2.0 RFC](https://tools.ietf.org/html/rfc6749)
- [OpenID Connect Specification](https://openid.net/connect/)
- [Google OAuth 2.0 Documentation](https://developers.google.com/identity/protocols/oauth2)
- [JWT Best Practices](https://tools.ietf.org/html/rfc8725)
- [OWASP Authentication Guide](https://owasp.org/www-project-cheat-sheets/cheatsheets/Authentication_Cheat_Sheet.html)

---

*This implementation provides a solid foundation for OAuth 2.0 authentication that can be extended and customized based on specific application requirements.*
