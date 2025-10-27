# Front-end/**
 * Single-file React app implementing Routing with Login Protection (all-in-one).
 *
 * How to use:
 * 1. Create a React app (e.g., with Vite or Create React App).
 * 2. Replace src/App.jsx content with this file (or create src/App.jsx and import it in index.js).
 * 3. Ensure react-router-dom is installed:
 *      npm install react-router-dom
 * 4. Start dev server (npm start / npm run dev).
 *
 * This example uses a fake in-memory "AuthService" that simulates a backend (no real server required).
 * It demonstrates:
 * - AuthContext for global auth state
 * - Login (with fake token), logout
 * - Persistence in localStorage
 * - ProtectedRoute (redirects to /login)
 * - RoleBasedRoute (example for admin-only pages)
 * - Token expiry handling
 * - Preserving intended destination (redirect back after login)
 *
 * NOTE: For production, replace AuthService with real API calls and secure token handling (HttpOnly cookies or secure storage).
 */

import React, { useEffect, useState, useContext, createContext, useCallback } from "react";
import { BrowserRouter, Routes, Route, Link, useNavigate, useLocation, Navigate } from "react-router-dom";

/* ---------------------- Fake Auth Service (simulates backend) ---------------------- */
const FakeAuthService = (() => {
  // demo users (passwords in plain text only for demo)
  const users = [
    { id: 1, email: "admin@example.com", password: "adminpass", username: "Admin User", role: "admin" },
    { id: 2, email: "user@example.com", password: "userpass", username: "Normal User", role: "user" },
  ];

  // Create a simple "token" (base64 JSON) with expiry
  function createToken(payload = {}, expiresInSeconds = 3600) {
    const exp = Math.floor(Date.now() / 1000) + expiresInSeconds;
    const tokenObj = { ...payload, exp };
    return btoa(JSON.stringify(tokenObj)); // simple encoding (NOT JWT), for demo only
  }

  function decodeToken(token) {
    try {
      const str = atob(token);
      return JSON.parse(str);
    } catch {
      return null;
    }
  }

  async function login({ email, password }) {
    // simulate network latency
    await new Promise((r) => setTimeout(r, 400));
    const user = users.find((u) => u.email === email && u.password === password);
    if (!user) {
      const err = new Error("Invalid credentials");
      err.status = 401;
      throw err;
    }
    const token = createToken({ sub: user.id, username: user.username, role: user.role }, 2 * 60 * 60); // 2 hours
    return { token };
  }

  async function fetchProfile(token) {
    await new Promise((r) => setTimeout(r, 200));
    const payload = decodeToken(token);
    if (!payload || (payload.exp && Date.now() / 1000 >= payload.exp)) {
      const err = new Error("Token expired or invalid");
      err.status = 401;
      throw err;
    }
    const u = users.find((x) => x.id === payload.sub);
    if (!u) {
      const err = new Error("User not found");
      err.status = 404;
      throw err;
    }
    return { id: u.id, username: u.username, email: u.email, role: u.role };
  }

  return { login, fetchProfile, decodeToken };
})();

/* ---------------------- Auth Context ---------------------- */
const AuthContext = createContext(null);

function useAuthContext() {
  return useContext(AuthContext);
}

function AuthProvider({ children }) {
  const [user, setUser] = useState(() => {
    // load from localStorage (token decode)
    const token = localStorage.getItem("authToken");
    if (!token) return null;
    const payload = FakeAuthService.decodeToken(token);
    if (!payload) {
      localStorage.removeItem("authToken");
      return null;
    }
    if (payload.exp && Date.now() / 1000 >= payload.exp) {
      localStorage.removeItem("authToken");
      return null;
    }
    return { username: payload.username, role: payload.role, id: payload.sub };
  });

  const [loading, setLoading] = useState(false);

  // login: calls FakeAuthService.login, stores token
  const login = useCallback(async (email, password) => {
    setLoading(true);
    try {
      const { token } = await FakeAuthService.login({ email, password });
      localStorage.setItem("authToken", token);
      const payload = FakeAuthService.decodeToken(token);
      setUser({ username: payload.username, role: payload.role, id: payload.sub });
      setLoading(false);
      return { ok: true };
    } catch (err) {
      setLoading(false);
      throw err;
    }
  }, []);

  const logout = useCallback(() => {
    localStorage.removeItem("authToken");
    setUser(null);
  }, []);

  // effect: auto-logout on token expiry
  useEffect(() => {
    let timer = null;
    const token = localStorage.getItem("authToken");
    if (token) {
      const payload = FakeAuthService.decodeToken(token);
      if (payload && payload.exp) {
        const msLeft = payload.exp * 1000 - Date.now();
        if (msLeft <= 0) {
          // expired already
          logout();
        } else {
          timer = setTimeout(() => {
            // token expired: clear and update state
            logout();
            // optional: you could notify user
            // alert('Session expired, please log in again.');
          }, msLeft + 1000);
        }
      }
    }
    return () => {
      if (timer) clearTimeout(timer);
    };
  }, [user, logout]);

  // expose context value
  const value = { user, login, logout, loading, isAuthenticated: !!user };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

/* ---------------------- ProtectedRoute & RoleBasedRoute ---------------------- */
function ProtectedRoute({ children }) {
  const auth = useAuthContext();
  const location = useLocation();
  if (!auth.isAuthenticated) {
    // redirect to login and preserve intended path
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  return children;
}

function RoleBasedRoute({ children, allowedRoles = [] }) {
  const auth = useAuthContext();
  const location = useLocation();
  if (!auth.isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  if (!allowedRoles.includes(auth.user?.role)) {
    return <Navigate to="/unauthorized" replace />;
  }
  return children;
}

/* ---------------------- UI Components (All in one file) ---------------------- */
function Navbar() {
  const auth = useAuthContext();
  return (
    <nav style={styles.nav}>
      <div>
        <Link style={styles.link} to="/">Home</Link>
        <Link style={styles.link} to="/about">About</Link>
        <Link style={styles.link} to="/dashboard">Dashboard</Link>
        <Link style={styles.link} to="/profile">Profile</Link>
        <Link style={styles.link} to="/admin">Admin</Link>
      </div>
      <div>
        {auth.isAuthenticated ? (
          <>
            <span style={{ marginRight: 12 }}>Hi, {auth.user.username}</span>
            <button onClick={auth.logout} style={styles.btn}>Logout</button>
          </>
        ) : (
          <Link to="/login" style={styles.btnLink}>Login</Link>
        )}
      </div>
    </nav>
  );
}

function Home() {
  return (
    <main style={styles.container}>
      <h2>Home</h2>
      <p>Welcome to the demo app showcasing React routing with login protection.</p>
    </main>
  );
}

function About() {
  return (
    <main style={styles.container}>
      <h2>About</h2>
      <p>This example demonstrates ProtectedRoute, AuthContext, and token expiry handling.</p>
    </main>
  );
}

function LoginPage() {
  const auth = useAuthContext();
  const navigate = useNavigate();
  const location = useLocation();
  const from = location.state?.from?.pathname || "/dashboard";

  const [form, setForm] = useState({ email: "", password: "" });
  const [error, setError] = useState(null);

  async function handleSubmit(e) {
    e.preventDefault();
    setError(null);
    try {
      await auth.login(form.email, form.password);
      navigate(from, { replace: true });
    } catch (err) {
      setError(err.message || "Login failed");
    }
  }

  return (
    <main style={styles.container}>
      <h2>Login</h2>
      <form onSubmit={handleSubmit} style={styles.form}>
        <label style={styles.label}>
          Email
          <input style={styles.input} value={form.email} onChange={(e) => setForm({ ...form, email: e.target.value })} type="email" required />
        </label>
        <label style={styles.label}>
          Password
          <input style={styles.input} value={form.password} onChange={(e) => setForm({ ...form, password: e.target.value })} type="password" required />
        </label>
        <div style={{ display: "flex", gap: 8 }}>
          <button type="submit" style={styles.btn} disabled={auth.loading}>{auth.loading ? "Logging in..." : "Login"}</button>
          <button type="button" onClick={() => { setForm({ email: "user@example.com", password: "userpass" }); }} style={styles.btnAlt}>Fill demo user</button>
          <button type="button" onClick={() => { setForm({ email: "admin@example.com", password: "adminpass" }); }} style={styles.btnAlt}>Fill admin</button>
        </div>
        {error && <p style={{ color: "red" }}>{error}</p>}
      </form>
      <p style={{ marginTop: 12 }}>
        Demo credentials: <br />
        user@example.com / userpass <br />
        admin@example.com / adminpass
      </p>
    </main>
  );
}

function Dashboard() {
  const auth = useAuthContext();
  return (
    <main style={styles.container}>
      <h2>Dashboard</h2>
      <p>Protected dashboard for authenticated users.</p>
      <p>Your role: <b>{auth.user?.role}</b></p>
    </main>
  );
}

function Profile() {
  const [profile, setProfile] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    let mounted = true;
    async function load() {
      const token = localStorage.getItem("authToken");
      if (!token) {
        setError("Not authenticated");
        return;
      }
      try {
        const data = await FakeAuthService.fetchProfile(token);
        if (mounted) setProfile(data);
      } catch (err) {
        setError(err.message || "Failed to load");
      }
    }
    load();
    return () => { mounted = false; };
  }, []);

  return (
    <main style={styles.container}>
      <h2>Profile</h2>
      {error && <p style={{ color: "red" }}>{error}</p>}
      {profile ? <pre>{JSON.stringify(profile, null, 2)}</pre> : <p>Loading profile...</p>}
    </main>
  );
}

function AdminPage() {
  return (
    <main style={styles.container}>
      <h2>Admin Area</h2>
      <p>This page is visible only to users with role <code>admin</code>.</p>
    </main>
  );
}

function Unauthorized() {
  return (
    <main style={styles.container}>
      <h2>Unauthorized</h2>
      <p>You don't have permission to view this page.</p>
    </main>
  );
}

/* ---------------------- App (wrap with AuthProvider & Router) ---------------------- */
export default function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <div style={{ minHeight: "100vh", display: "flex", flexDirection: "column" }}>
          <Navbar />
          <div style={{ flex: 1 }}>
            <Routes>
              <Route path="/" element={<Home />} />
              <Route path="/about" element={<About />} />
              <Route path="/login" element={<LoginPage />} />
              <Route path="/unauthorized" element={<Unauthorized />} />

              {/* Protected routes */}
              <Route path="/dashboard" element={<ProtectedRoute><Dashboard /></ProtectedRoute>} />
              <Route path="/profile" element={<ProtectedRoute><Profile /></ProtectedRoute>} />

              {/* Role-based (admin only) */}
              <Route path="/admin" element={<RoleBasedRoute allowedRoles={["admin"]}><AdminPage /></RoleBasedRoute>} />

              <Route path="*" element={<div style={styles.container}><h2>404 - Not Found</h2></div>} />
            </Routes>
          </div>
          <footer style={styles.footer}>
            <small>Demo app — React Routing with Login Protection</small>
          </footer>
        </div>
      </AuthProvider>
    </BrowserRouter>
  );
}

/* ---------------------- Minimal inline styles ---------------------- */
const styles = {
  nav: {
    display: "flex",
    justifyContent: "space-between",
    padding: "12px 20px",
    background: "#20232a",
    color: "#fff",
    alignItems: "center",
    gap: 12,
  },
  link: {
    color: "#61dafb",
    marginRight: 12,
    textDecoration: "none",
  },
  container: {
    padding: 20,
    maxWidth: 900,
    margin: "0 auto",
  },
  form: {
    display: "flex",
    flexDirection: "column",
    gap: 8,
    maxWidth: 420,
  },
  label: {
    display: "flex",
    flexDirection: "column",
    fontSize: 14,
  },
  input: {
    padding: "8px 10px",
    borderRadius: 4,
    border: "1px solid #ddd",
    marginTop: 6,
  },
  btn: {
    padding: "8px 12px",
    background: "#61dafb",
    border: "none",
    borderRadius: 4,
    cursor: "pointer",
  },
  btnAlt: {
    padding: "8px 12px",
    background: "#f0f0f0",
    border: "1px solid #ccc",
    borderRadius: 4,
    cursor: "pointer",
  },
  btnLink: {
    padding: "6px 12px",
    background: "#61dafb",
    borderRadius: 4,
    color: "#000",
    textDecoration: "none",
  },
  footer: {
    padding: 12,
    textAlign: "center",
    borderTop: "1px solid #eee",
  },
};-technology
