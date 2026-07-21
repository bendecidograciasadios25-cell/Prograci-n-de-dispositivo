# 🛠️ ManosExpertas - Plataforma de Servicios Técnicos bajo Demanda

**ManosExpertas** es una aplicación móvil diseñada para conectar a clientes que requieren reparaciones, mantenimientos o servicios instalativos en el hogar u oficina con profesionales y técnicos verificados en áreas como electricidad, plomería, redes, climatización y mantenimiento general.

---

## 📋 Tabla de Contenidos

1. [Descripción del Proyecto y MVP](#1-descripción-del-proyecto-y-mvp)
2. [Arquitectura y Stack Tecnológico](#2-arquitectura-y-stack-tecnológico)
3. [Base de Datos (PostgreSQL)](#3-base-de-datos-postgresql)
4. [Backend (Node.js + Express + FCM)](#4-backend-nodejs--express--fcm)
5. [Frontend Móvil (Flutter)](#5-frontend-móvil-flutter)
6. [Notificaciones Push (Firebase Cloud Messaging)](#6-notificaciones-push-firebase-cloud-messaging)
7. [Guía de Despliegue en Railway](#7-guía-de-despliegue-en-railway)
8. [Instrucciones de Instalación Local](#8-instrucciones-de-instalación-local)

---

## 1. Descripción del Proyecto y MVP

El Producto Mínimo Viable (MVP) de **ManosExpertas** cubre las siguientes funcionalidades básicas:

- **Autenticación con Roles:** Registro e inicio de sesión diferenciado para **Clientes** y **Técnicos**.
- **Publicación de Solicitudes:** Los clientes pueden crear solicitudes de trabajo especificando categoría, título y descripción.
- **Toma de Trabajos:** Los técnicos pueden ver un listado público de trabajos pendientes en tiempo real y aceptarlos.
- **Seguimiento de Estado:** Los clientes pueden consultar el estado de sus solicitudes (`pendiente`, `aceptado`, `finalizado`) y conocer qué técnico fue asignado.
- **Notificaciones Push:** Notificaciones automáticas al teléfono del cliente cuando un técnico acepta su trabajo.

---

## 2. Arquitectura y Stack Tecnológico

```
  [ App Móvil (Flutter) ]
     (Android / iOS)
            │
            │  Peticiones HTTPS (JSON) + FCM Tokens
            ▼
  [ Backend API (Node.js + Express) ]  ───► [ Firebase Admin SDK (FCM) ]
            │                                           │
            │  Consultas SQL                            │ Notificaciones Push
            ▼                                           ▼
  [ Base de Datos PostgreSQL (Railway) ]       [ Dispositivo Cliente ]
```

- **Frontend Móvil:** Flutter (Dart)
- **Backend API:** Node.js + Express.js
- **Base de Datos:** PostgreSQL
- **Hosting Cloud:** Railway (Servidor Web + PostgreSQL)
- **Notificaciones:** Firebase Cloud Messaging (FCM)

---

## 3. Base de Datos (PostgreSQL)

Ejecuta el siguiente script SQL en tu instancia de PostgreSQL (disponible en Railway):

```sql
-- Tabla de Usuarios con Soporte de Roles y FCM Token
CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    rol VARCHAR(20) CHECK (rol IN ('cliente', 'tecnico')) DEFAULT 'cliente',
    telefono VARCHAR(20),
    fcm_token TEXT,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla de Solicitudes de Servicio
CREATE TABLE servicios (
    id SERIAL PRIMARY KEY,
    cliente_id INT REFERENCES usuarios(id) ON DELETE CASCADE,
    tecnico_id INT REFERENCES usuarios(id) ON DELETE SET NULL,
    titulo VARCHAR(150) NOT NULL,
    descripcion TEXT NOT NULL,
    categoria VARCHAR(50) CHECK (categoria IN (
        'Electricidad', 
        'Plomería', 
        'Redes y Telecomunicaciones', 
        'Climatización', 
        'Mantenimiento General'
    )) NOT NULL,
    estado VARCHAR(20) CHECK (estado IN ('pendiente', 'aceptado', 'finalizado')) DEFAULT 'pendiente',
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 4. Backend (Node.js + Express + FCM)

### Estrutura del Proyecto Backend
```
backend/
├── package.json
├── server.js
└── .env
```

### `package.json`
```json
{
  "name": "manos-expertas-backend",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "firebase-admin": "^12.1.0",
    "jsonwebtoken": "^9.0.2",
    "pg": "^8.11.5"
  },
  "scripts": {
    "start": "node server.js"
  }
}
```

### Código Fuente de `server.js`
```javascript
const express = require('express');
const cors = require('cors');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const { Pool } = require('pg');
const admin = require('firebase-admin');

const app = express();
app.use(express.json());
app.use(cors());

// Inicialización de Firebase Admin SDK
if (process.env.FIREBASE_CREDENTIALS) {
  const serviceAccount = JSON.parse(process.env.FIREBASE_CREDENTIALS);
  admin.initializeApp({
    credential: admin.credential.cert(serviceAccount)
  });
}

// Conexión a la base de datos PostgreSQL en Railway
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.DATABASE_URL ? { rejectUnauthorized: false } : false
});

// --- AUTENTICACIÓN ---

// Registro de Usuario (Cliente o Técnico)
app.post('/api/auth/register', async (req, res) => {
  const { nombre, email, password, rol, telefono } = req.body;
  try {
    const hashedPassword = await bcrypt.hash(password, 10);
    const result = await pool.query(
      'INSERT INTO usuarios (nombre, email, password_hash, rol, telefono) VALUES ($1, $2, $3, $4, $5) RETURNING id, nombre, email, rol',
      [nombre, email, hashedPassword, rol || 'cliente', telefono]
    );
    res.status(201).json(result.rows[0]);
  } catch (err) {
    res.status(400).json({ error: 'El correo electrónico ya se encuentra registrado.' });
  }
});

// Login de Usuario
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;
  try {
    const user = await pool.query('SELECT * FROM usuarios WHERE email = $1', [email]);
    if (user.rows.length === 0) return res.status(400).json({ error: 'Usuario no encontrado' });

    const validPassword = await bcrypt.compare(password, user.rows[0].password_hash);
    if (!validPassword) return res.status(400).json({ error: 'Contraseña incorrecta' });

    const token = jwt.sign({ id: user.rows[0].id }, process.env.JWT_SECRET || 'secreto', { expiresIn: '7d' });
    res.json({
      token,
      user: {
        id: user.rows[0].id,
        nombre: user.rows[0].nombre,
        email: user.rows[0].email,
        rol: user.rows[0].rol
      }
    });
  } catch (err) {
    res.status(500).json({ error: 'Error interno del servidor.' });
  }
});

// Guardar o actualizar Token FCM
app.post('/api/usuarios/fcm-token', async (req, res) => {
  const { usuario_id, fcm_token } = req.body;
  try {
    await pool.query('UPDATE usuarios SET fcm_token = $1 WHERE id = $2', [fcm_token, usuario_id]);
    res.json({ mensaje: 'Token FCM actualizado correctamente.' });
  } catch (err) {
    res.status(500).json({ error: 'Error al actualizar el token FCM.' });
  }
});

// --- SERVICIOS ---

// Crear Solicitud de Servicio (Cliente)
app.post('/api/servicios', async (req, res) => {
  const { cliente_id, titulo, descripcion, categoria } = req.body;
  try {
    const result = await pool.query(
      'INSERT INTO servicios (cliente_id, titulo, descripcion, categoria) VALUES ($1, $2, $3, $4) RETURNING *',
      [cliente_id, titulo, descripcion, categoria]
    );
    res.status(201).json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: 'Error al publicar la solicitud de servicio.' });
  }
});

// Obtener servicios creados por un cliente
app.get('/api/servicios/cliente/:clienteId', async (req, res) => {
  const { clienteId } = req.params;
  try {
    const result = await pool.query(
      `SELECT s.*, u.nombre AS tecnico_nombre 
       FROM servicios s 
       LEFT JOIN usuarios u ON s.tecnico_id = u.id 
       WHERE s.cliente_id = $1 
       ORDER BY s.creado_en DESC`,
      [clienteId]
    );
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: 'Error al obtener servicios del cliente.' });
  }
});

// Listar servicios pendientes (para Técnicos)
app.get('/api/servicios/pendientes', async (req, res) => {
  try {
    const result = await pool.query(
      `SELECT s.*, u.nombre AS cliente_nombre 
       FROM servicios s 
       JOIN usuarios u ON s.cliente_id = u.id 
       WHERE s.estado = 'pendiente' 
       ORDER BY s.creado_en DESC`
    );
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: 'Error al obtener servicios pendientes.' });
  }
});

// Aceptar Servicio (Técnico) + Notificación Push al Cliente
app.put('/api/servicios/:id/aceptar', async (req, res) => {
  const { id } = req.params;
  const { tecnico_id } = req.body;

  try {
    const result = await pool.query(
      'UPDATE servicios SET tecnico_id = $1, estado = $2 WHERE id = $3 AND estado = $4 RETURNING *',
      [tecnico_id, 'aceptado', id, 'pendiente']
    );

    if (result.rows.length === 0) {
      return res.status(400).json({ error: 'El servicio ya no está disponible.' });
    }

    const servicio = result.rows[0];

    // Datos del cliente y técnico para la notificación
    const clienteRes = await pool.query('SELECT fcm_token FROM usuarios WHERE id = $1', [servicio.cliente_id]);
    const tecnicoRes = await pool.query('SELECT nombre FROM usuarios WHERE id = $1', [tecnico_id]);

    const clienteToken = clienteRes.rows[0]?.fcm_token;
    const nombreTecnico = tecnicoRes.rows[0]?.nombre || 'Un técnico';

    // Enviar notificación Push vía Firebase FCM si el token existe
    if (clienteToken && admin.apps.length > 0) {
      const message = {
        notification: {
          title: '¡Trabajo Aceptado! 🛠️',
          body: `${nombreTecnico} ha aceptado tu solicitud "${servicio.titulo}".`,
        },
        data: {
          servicio_id: String(servicio.id),
          tipo: 'servicio_aceptado'
        },
        token: clienteToken
      };
      await admin.messaging().send(message);
    }

    res.json({ mensaje: 'Servicio asignado correctamente', servicio });
  } catch (err) {
    res.status(500).json({ error: 'Error al asignar el servicio.' });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Servidor ManosExpertas activo en puerto ${PORT}`));
```

---

## 5. Frontend Móvil (Flutter)

### Estructura de Proyecto en Flutter
```
lib/
├── main.dart
├── screens/
│   ├── login_screen.dart
│   ├── registro_screen.dart
│   ├── cliente_dashboard_screen.dart
│   ├── tecnico_dashboard_screen.dart
│   └── crear_servicio_screen.dart
└── services/
    └── notification_service.dart
```

### `pubspec.yaml` (Dependencias recomendadas)
```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.1
  firebase_core: ^3.0.0
  firebase_messaging: ^15.0.0
```

---

## 6. Notificaciones Push (Firebase Cloud Messaging)

1. Crea un proyecto en la [Consola de Firebase](https://console.firebase.google.com/).
2. Vincula tus aplicaciones Android e iOS descargando `google-services.json` (Android) y `GoogleService-Info.plist` (iOS).
3. Ve a **Configuración del proyecto** -> **Cuentas de servicio** y haz clic en **Generar nueva clave privada**.
4. Copia todo el contenido del archivo JSON generado y colócalo como variable de entorno `FIREBASE_CREDENTIALS` en Railway.

---

## 7. Guía de Despliegue en Railway

1. **Crear Repositorio:** Sube la carpeta `backend` a un repositorio en GitHub.
2. **Nuevo Proyecto en Railway:** 
   - Ve a [Railway.app](https://railway.app/).
   - Selecciona **New Project** -> **Deploy from GitHub repo**.
   - Conecta tu repositorio.
3. **Añadir PostgreSQL:**
   - En la vista del proyecto en Railway, haz clic en **+ New** -> **Database** -> **Add PostgreSQL**.
4. **Configurar Variables de Entorno en el Backend:**
   - En el servicio Backend de Railway, navega a **Variables** y agrega:
     - `DATABASE_URL`: `${{Postgres.DATABASE_URL}}`
     - `JWT_SECRET`: `tu_clave_secreta_super_segura`
     - `FIREBASE_CREDENTIALS`: `{"type": "service_account", ...}`
5. **Generar Dominio Público:**
   - En **Settings** -> **Networking**, presiona **Generate Domain**.
   - Usa esta URL pública (ejemplo: `https://tu-app.up.railway.app`) en tu código de Flutter.

---

## 8. Instrucciones de Instalación Local

### Requisitos Previos
- Node.js v18+
- Flutter SDK 3.x+
- PostgreSQL local o acceso al PostgreSQL de Railway

### Iniciar Backend
```bash
cd backend
npm install
node server.js
```

### Iniciar App Flutter
```bash
flutter pub get
flutter run
```

---
© 2026 ManosExpertas - Plataforma de Servicios Técnicos bajo Demanda.
