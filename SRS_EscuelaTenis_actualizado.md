# 📋 Documento de Especificación de Requerimientos de Software (SRS)
## Aplicación Móvil — Gestión y Administración de Escuela de Tenis

---

**Versión:** 1.0.0  
**Fecha:** Marzo 2026  
**Estado:** Borrador Final  
**Framework:** React Native (Expo)  
**Backend:** Supabase (PostgreSQL + Auth + Storage + Realtime)

---

## 📑 Tabla de Contenidos

1. [Introducción](#1-introducción)
2. [Visión General del Sistema](#2-visión-general-del-sistema)
3. [Arquitectura del Sistema](#3-arquitectura-del-sistema)
4. [Stack Tecnológico](#4-stack-tecnológico)
5. [Estructura del Proyecto](#5-estructura-del-proyecto)
6. [Diseño de Base de Datos](#6-diseño-de-base-de-datos)
7. [Autenticación y Autorización](#7-autenticación-y-autorización)
8. [Módulos y Funcionalidades](#8-módulos-y-funcionalidades)
9. [Diseño de Interfaces (UI/UX)](#9-diseño-de-interfaces-uiux)
10. [API y Servicios](#10-api-y-servicios)
11. [Requerimientos No Funcionales](#11-requerimientos-no-funcionales)
12. [Flujos de Usuario](#12-flujos-de-usuario)
13. [Notificaciones](#13-notificaciones)
14. [Seguridad](#14-seguridad)
15. [Variables de Entorno](#15-variables-de-entorno)
16. [Guía de Implementación Paso a Paso](#16-guía-de-implementación-paso-a-paso)

---

## 1. Introducción

### 1.1 Propósito
Este documento define la especificación completa de requerimientos para el desarrollo de una aplicación móvil de gestión y administración de una Escuela de Tenis. Está diseñado para servir como contexto completo a un IDE con IA (Google Antigravity u otro), permitiéndole generar el sistema completo desde cero sin ambigüedades.

### 1.2 Alcance
La aplicación permite:
- Gestión de clases de tenis (creación, edición, eliminación)
- Inscripción de alumnos en clases disponibles
- Visualización de disponibilidad en calendario (diario, semanal, mensual)
- Gestión de pagos y estados de cuenta
- Panel de administración para profesores y administradores
- Sistema de autenticación seguro con Supabase Auth

### 1.3 Convenciones del Documento
- **ADMIN**: Usuario con rol administrador
- **COACH**: Profesor/entrenador
- **STUDENT**: Alumno/estudiante inscrito
- **GUEST**: Usuario no autenticado

### 1.4 Definiciones Clave
| Término | Definición |
|---------|-----------|
| Clase | Sesión de tenis con horario, cancha y cupo definido |
| Inscripción | Registro de un alumno en una clase específica |
| Cancha | Pista de tenis disponible en la escuela |
| Bloque horario | Franja de tiempo disponible para una clase |
| Plan | Paquete de clases adquirido por el alumno |

---

## 2. Visión General del Sistema

### 2.1 Perspectiva del Producto
Aplicación móvil multiplataforma (iOS y Android) que centraliza la operación completa de una escuela de tenis: desde la gestión de horarios y canchas hasta el control de pagos e inscripciones de alumnos.

### 2.2 Roles de Usuario

```
┌─────────────────────────────────────────────┐
│                  ROLES                       │
├─────────────┬───────────────┬───────────────┤
│   ADMIN     │    COACH      │   STUDENT     │
│─────────────│───────────────│───────────────│
│ Gestión     │ Ver sus       │ Ver clases    │
│ completa    │ clases        │ disponibles   │
│ del sistema │ asignadas     │               │
│             │               │ Inscribirse   │
│ CRUD de     │ Marcar        │               │
│ clases      │ asistencia    │ Ver pagos     │
│             │               │ pendientes    │
│ CRUD de     │ Ver alumnos   │               │
│ usuarios    │ inscritos     │ Ver su        │
│             │               │ historial     │
│ Gestión de  │ Comunicarse   │               │
│ pagos       │ con alumnos   │ Cancelar      │
│             │               │ inscripción   │
│ Reportes    │               │               │
└─────────────┴───────────────┴───────────────┘
```

### 2.3 Restricciones del Sistema
- La aplicación debe funcionar en iOS 13+ y Android 8+
- Requiere conexión a internet para operaciones en tiempo real
- El almacenamiento de datos es exclusivamente en Supabase (PostgreSQL)
- La autenticación es gestionada 100% por Supabase Auth
- **Horario operativo del recinto:** 08:00 a 22:00 (hora local). No se permite crear clases fuera de este rango, salvo excepción habilitada por ADMIN.
- **Infraestructura:** existen exactamente **2 canchas** disponibles (Cancha 1 y Cancha 2). El ADMIN puede desactivar temporalmente una cancha por mantención.
- **Regla anti-solape:** no se permiten clases que se traslapen en la misma cancha (controlado por constraint en BD).

---

## 3. Arquitectura del Sistema

### 3.1 Diagrama de Arquitectura General

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENTE MÓVIL                            │
│              React Native (Expo SDK 51+)                    │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Screens │  │Components│  │  Hooks   │  │  Store   │  │
│  │ (Views)  │  │(UI Atoms)│  │(Business)│  │ (Zustand)│  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       └─────────────┴──────────────┴──────────────┘        │
│                          │                                  │
│                   ┌──────┴──────┐                          │
│                   │  API Layer  │                          │
│                   │  (Services) │                          │
│                   └──────┬──────┘                          │
└──────────────────────────┼──────────────────────────────────┘
                           │ HTTPS / WebSocket
┌──────────────────────────┼──────────────────────────────────┐
│                   SUPABASE BACKEND                          │
│                          │                                  │
│  ┌───────────┐  ┌────────┴────────┐  ┌──────────────────┐ │
│  │ Supabase  │  │  PostgreSQL DB  │  │ Supabase Storage │ │
│  │   Auth    │  │  (Row Level     │  │  (Avatares,      │ │
│  │           │  │   Security)     │  │   documentos)    │ │
│  └───────────┘  └─────────────────┘  └──────────────────┘ │
│                                                             │
│  ┌───────────┐  ┌─────────────────┐                       │
│  │ Realtime  │  │  Edge Functions │                       │
│  │(WebSocket)│  │  (Notificaciones│                       │
│  │           │  │   y Pagos)      │                       │
│  └───────────┘  └─────────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Patrón de Arquitectura
Se implementará el patrón **Feature-Based Architecture** con separación clara de capas:

- **Presentation Layer**: Screens y Components (React Native)
- **Business Logic Layer**: Custom Hooks y Zustand Store
- **Data Layer**: Services que consumen la API de Supabase
- **Infrastructure Layer**: Configuración de Supabase, constantes, utilidades

---

## 4. Stack Tecnológico

### 4.1 Frontend (React Native)

```json
{
  "framework": "React Native con Expo SDK 51",
  "lenguaje": "TypeScript (strict mode)",
  "navegacion": "Expo Router v3 (file-based routing)",
  "estado_global": "Zustand v4",
  "estado_servidor": "TanStack Query v5 (React Query)",
  "ui_components": "React Native Paper v5",
  "iconos": "Expo Vector Icons (@expo/vector-icons)",
  "formularios": "React Hook Form v7 + Zod (validación)",
  "calendario": "react-native-calendars v1",
  "fechas": "date-fns v3",
  "animaciones": "React Native Reanimated v3",
  "gestos": "React Native Gesture Handler v2",
  "notificaciones": "Expo Notifications",
  "imagenes": "Expo Image",
  "storage_local": "Expo SecureStore",
  "testing": "Jest + React Native Testing Library"
}
```

### 4.2 Backend (Supabase)

```json
{
  "base_de_datos": "PostgreSQL 15 (via Supabase)",
  "autenticacion": "Supabase Auth (Email/Password + OAuth Google)",
  "almacenamiento": "Supabase Storage",
  "tiempo_real": "Supabase Realtime",
  "funciones_serverless": "Supabase Edge Functions (Deno)",
  "seguridad": "Row Level Security (RLS) policies",
  "sdk_cliente": "@supabase/supabase-js v2"
}
```

### 4.3 Herramientas de Desarrollo

```json
{
  "version_control": "Git (GitHub)",
  "ci_cd": "GitHub Actions + EAS Build",
  "linting": "ESLint + Prettier",
  "commits": "Conventional Commits",
  "env_management": "dotenv + EAS Secrets"
}
```

---

## 5. Estructura del Proyecto

```
tennis-school-app/
├── app/                          # Expo Router — rutas de la app
│   ├── (auth)/                   # Rutas públicas (sin auth)
│   │   ├── _layout.tsx
│   │   ├── login.tsx
│   │   ├── register.tsx
│   │   └── forgot-password.tsx
│   ├── (tabs)/                   # Rutas protegidas con tabs
│   │   ├── _layout.tsx           # Tab navigator
│   │   ├── index.tsx             # Home / Dashboard
│   │   ├── schedule.tsx          # Calendario de clases
│   │   ├── my-classes.tsx        # Mis inscripciones
│   │   ├── payments.tsx          # Pagos y estado de cuenta
│   │   └── profile.tsx           # Perfil de usuario
│   ├── (admin)/                  # Rutas solo ADMIN/COACH
│   │   ├── _layout.tsx
│   │   ├── dashboard.tsx
│   │   ├── classes/
│   │   │   ├── index.tsx
│   │   │   ├── [id].tsx
│   │   │   └── create.tsx
│   │   ├── students/
│   │   │   ├── index.tsx
│   │   │   └── [id].tsx
│   │   ├── coaches/
│   │   │   ├── index.tsx
│   │   │   └── [id].tsx
│   │   └── payments/
│   │       ├── index.tsx
│   │       └── [id].tsx
│   ├── class/
│   │   └── [id].tsx              # Detalle de clase (inscripción)
│   ├── _layout.tsx               # Root layout
│   └── +not-found.tsx
│
├── src/
│   ├── components/               # Componentes reutilizables
│   │   ├── ui/                   # Átomos de UI
│   │   │   ├── Button.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Badge.tsx
│   │   │   ├── Avatar.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Skeleton.tsx
│   │   │   └── EmptyState.tsx
│   │   ├── class/
│   │   │   ├── ClassCard.tsx
│   │   │   ├── ClassList.tsx
│   │   │   ├── ClassForm.tsx
│   │   │   └── ClassDetailSheet.tsx
│   │   ├── calendar/
│   │   │   ├── CalendarView.tsx
│   │   │   ├── DayView.tsx
│   │   │   ├── WeekView.tsx
│   │   │   └── MonthView.tsx
│   │   ├── payment/
│   │   │   ├── PaymentCard.tsx
│   │   │   ├── PaymentBadge.tsx
│   │   │   └── PaymentSummary.tsx
│   │   └── shared/
│   │       ├── Header.tsx
│   │       ├── LoadingScreen.tsx
│   │       ├── ErrorBoundary.tsx
│   │       └── ProtectedRoute.tsx
│   │
│   ├── hooks/                    # Custom hooks
│   │   ├── useAuth.ts
│   │   ├── useClasses.ts
│   │   ├── useEnrollments.ts
│   │   ├── usePayments.ts
│   │   ├── useSchedule.ts
│   │   ├── useProfile.ts
│   │   └── useNotifications.ts
│   │
│   ├── services/                 # Capa de acceso a datos
│   │   ├── supabase.ts           # Cliente Supabase inicializado
│   │   ├── auth.service.ts
│   │   ├── classes.service.ts
│   │   ├── enrollments.service.ts
│   │   ├── payments.service.ts
│   │   ├── profiles.service.ts
│   │   ├── courts.service.ts
│   │   └── notifications.service.ts
│   │
│   ├── store/                    # Zustand stores
│   │   ├── auth.store.ts
│   │   ├── ui.store.ts
│   │   └── schedule.store.ts
│   │
│   ├── types/                    # TypeScript types e interfaces
│   │   ├── database.types.ts     # Generado por Supabase CLI
│   │   ├── auth.types.ts
│   │   ├── class.types.ts
│   │   ├── enrollment.types.ts
│   │   ├── payment.types.ts
│   │   └── ui.types.ts
│   │
│   ├── utils/                    # Funciones utilitarias
│   │   ├── date.utils.ts
│   │   ├── format.utils.ts       # Formateo de moneda, etc.
│   │   ├── validation.utils.ts
│   │   └── constants.ts
│   │
│   └── theme/                    # Sistema de diseño
│       ├── colors.ts
│       ├── typography.ts
│       ├── spacing.ts
│       └── theme.ts
│
├── supabase/
│   ├── migrations/               # Migraciones SQL ordenadas
│   │   ├── 001_initial_schema.sql
│   │   ├── 002_rls_policies.sql
│   │   ├── 003_functions.sql
│   │   └── 004_seed_data.sql
│   ├── functions/                # Edge Functions
│   │   ├── send-notification/
│   │   │   └── index.ts
│   │   └── process-payment/
│   │       └── index.ts
│   └── config.toml
│
├── assets/
│   ├── images/
│   ├── fonts/
│   └── icons/
│
├── .env.local                    # Variables de entorno (NO commitear)
├── .env.example                  # Ejemplo de variables
├── app.json                      # Config de Expo
├── eas.json                      # Config de EAS Build
├── tsconfig.json
├── package.json
└── README.md
```

---

## 6. Diseño de Base de Datos

### 6.1 Esquema Completo PostgreSQL

#### Tabla: `profiles`
Extiende la tabla `auth.users` de Supabase con información adicional del usuario.

```sql
CREATE TABLE public.profiles (
  id            UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email         TEXT NOT NULL UNIQUE,
  full_name     TEXT NOT NULL,
  phone         TEXT,
  avatar_url    TEXT,
  role          TEXT NOT NULL DEFAULT 'student' 
                  CHECK (role IN ('admin', 'coach', 'student')),
  rut           TEXT UNIQUE,                    -- RUT chileno
  birth_date    DATE,
  address       TEXT,
  emergency_contact_name  TEXT,
  emergency_contact_phone TEXT,
  is_active     BOOLEAN NOT NULL DEFAULT TRUE,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Trigger para actualizar updated_at automáticamente
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_profiles_updated_at
  BEFORE UPDATE ON public.profiles
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

#### Tabla: `courts`
Canchas/pistas de tenis disponibles en la escuela.

```sql
CREATE TABLE public.courts (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL,               -- Ej: "Cancha 1", "Cancha Central"
  surface     TEXT NOT NULL DEFAULT 'hard' 
                CHECK (surface IN ('clay', 'hard', 'grass', 'carpet')),
  is_indoor   BOOLEAN NOT NULL DEFAULT FALSE,
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  description TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Tabla: `class_categories`
Categorías de clases (niveles y tipos).

```sql
CREATE TABLE public.class_categories (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL UNIQUE,        -- Ej: "Iniciación", "Intermedio", "Avanzado"
  level       INTEGER NOT NULL DEFAULT 1,  -- 1=Principiante, 5=Elite
  color       TEXT NOT NULL DEFAULT '#3B82F6',  -- Color hex para UI
  description TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Tabla: `classes`
Clases de tenis disponibles para inscripción.

```sql
CREATE TABLE public.classes (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title           TEXT NOT NULL,
  description     TEXT,
  coach_id        UUID NOT NULL REFERENCES public.profiles(id),
  court_id        UUID NOT NULL REFERENCES public.courts(id),
  category_id     UUID REFERENCES public.class_categories(id),
  
  -- Horario
  start_datetime  TIMESTAMPTZ NOT NULL,    -- Fecha y hora inicio
  end_datetime    TIMESTAMPTZ NOT NULL,    -- Fecha y hora fin
  duration_minutes INTEGER GENERATED ALWAYS AS 
    (EXTRACT(EPOCH FROM (end_datetime - start_datetime)) / 60) STORED,
  
  -- Recurrencia
  is_recurring    BOOLEAN NOT NULL DEFAULT FALSE,
  recurrence_rule TEXT,                    -- RFC 5545: RRULE (ej: FREQ=WEEKLY;BYDAY=MO,WE,FR)
  recurrence_end_date DATE,
  parent_class_id UUID REFERENCES public.classes(id), -- Para clases de serie
  
  -- Cupos y precio
  max_students    INTEGER NOT NULL DEFAULT 6,
  price           DECIMAL(10,2) NOT NULL DEFAULT 0,
  currency        TEXT NOT NULL DEFAULT 'CLP',
  
  -- Estado
  status          TEXT NOT NULL DEFAULT 'scheduled'
                    CHECK (status IN ('scheduled', 'in_progress', 'completed', 'cancelled')),
  cancellation_reason TEXT,
  
  -- Metadata
  notes           TEXT,
  created_by      UUID REFERENCES public.profiles(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_classes_start_datetime ON public.classes(start_datetime);
CREATE INDEX idx_classes_coach_id ON public.classes(coach_id);
CREATE INDEX idx_classes_status ON public.classes(status);

CREATE TRIGGER update_classes_updated_at
  BEFORE UPDATE ON public.classes
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

#### Tabla: `enrollments`
Inscripciones de alumnos en clases.

```sql
CREATE TABLE public.enrollments (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id      UUID NOT NULL REFERENCES public.classes(id) ON DELETE CASCADE,
  student_id    UUID NOT NULL REFERENCES public.profiles(id),
  
  -- Estado de la inscripción
  status        TEXT NOT NULL DEFAULT 'confirmed'
                  CHECK (status IN ('pending', 'confirmed', 'cancelled', 'waitlist')),
  
  -- Asistencia
  attendance    TEXT DEFAULT NULL
                  CHECK (attendance IN ('present', 'absent', 'late', 'excused')),
  attended_at   TIMESTAMPTZ,
  
  -- Control
  enrolled_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  cancelled_at  TIMESTAMPTZ,
  cancel_reason TEXT,
  enrolled_by   UUID REFERENCES public.profiles(id), -- Quién inscribió (self o admin)
  
  -- Constraint: un alumno no puede inscribirse dos veces en la misma clase
  UNIQUE (class_id, student_id)
);

CREATE INDEX idx_enrollments_class_id ON public.enrollments(class_id);
CREATE INDEX idx_enrollments_student_id ON public.enrollments(student_id);
CREATE INDEX idx_enrollments_status ON public.enrollments(status);
```

#### Tabla: `payment_plans`
Planes/paquetes de pago disponibles.

```sql
CREATE TABLE public.payment_plans (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name            TEXT NOT NULL,           -- Ej: "Plan Mensual 8 clases"
  description     TEXT,
  classes_count   INTEGER NOT NULL,        -- Cantidad de clases incluidas
  price           DECIMAL(10,2) NOT NULL,
  currency        TEXT NOT NULL DEFAULT 'CLP',
  validity_days   INTEGER NOT NULL DEFAULT 30,  -- Días de validez del plan
  is_active       BOOLEAN NOT NULL DEFAULT TRUE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Tabla: `payments`
Registro de pagos de alumnos.

```sql
CREATE TABLE public.payments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id      UUID NOT NULL REFERENCES public.profiles(id),
  class_id        UUID REFERENCES public.classes(id), -- Si es pago por clase individual
  plan_id         UUID REFERENCES public.payment_plans(id), -- Si es pago de plan
  enrollment_id   UUID REFERENCES public.enrollments(id),
  
  -- Montos
  amount          DECIMAL(10,2) NOT NULL,
  currency        TEXT NOT NULL DEFAULT 'CLP',
  discount        DECIMAL(10,2) DEFAULT 0,
  final_amount    DECIMAL(10,2) GENERATED ALWAYS AS (amount - COALESCE(discount, 0)) STORED,
  
  -- Estado del pago
  status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'paid', 'overdue', 'refunded', 'cancelled')),
  
  -- Método y referencia
  payment_method  TEXT CHECK (payment_method IN ('cash', 'transfer', 'card', 'other')),
  payment_reference TEXT,                  -- Número de comprobante
  
  -- Fechas
  due_date        DATE NOT NULL,
  paid_at         TIMESTAMPTZ,
  
  -- Descripción y notas
  description     TEXT,
  notes           TEXT,
  
  -- Control
  registered_by   UUID REFERENCES public.profiles(id), -- Admin que registró el pago
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_student_id ON public.payments(student_id);
CREATE INDEX idx_payments_status ON public.payments(status);
CREATE INDEX idx_payments_due_date ON public.payments(due_date);

CREATE TRIGGER update_payments_updated_at
  BEFORE UPDATE ON public.payments
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

#### Tabla: `notifications`
Notificaciones del sistema.

```sql
CREATE TABLE public.notifications (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  title       TEXT NOT NULL,
  body        TEXT NOT NULL,
  type        TEXT NOT NULL 
                CHECK (type IN ('class_reminder', 'payment_due', 'class_cancelled', 
                                'enrollment_confirmed', 'class_updated', 'general')),
  data        JSONB DEFAULT '{}',          -- Payload extra (class_id, etc.)
  is_read     BOOLEAN NOT NULL DEFAULT FALSE,
  read_at     TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_id ON public.notifications(user_id);
CREATE INDEX idx_notifications_is_read ON public.notifications(is_read);
```

#### Vista: `classes_with_availability`
Vista que incluye cupos disponibles en tiempo real.

```sql
CREATE OR REPLACE VIEW public.classes_with_availability AS
SELECT 
  c.*,
  p.full_name AS coach_name,
  p.avatar_url AS coach_avatar,
  ct.name AS court_name,
  ct.surface AS court_surface,
  cc.name AS category_name,
  cc.color AS category_color,
  cc.level AS category_level,
  COUNT(e.id) FILTER (WHERE e.status = 'confirmed') AS enrolled_count,
  c.max_students - COUNT(e.id) FILTER (WHERE e.status = 'confirmed') AS available_spots,
  COUNT(e.id) FILTER (WHERE e.status = 'waitlist') AS waitlist_count
FROM public.classes c
JOIN public.profiles p ON c.coach_id = p.id
JOIN public.courts ct ON c.court_id = ct.id
LEFT JOIN public.class_categories cc ON c.category_id = cc.id
LEFT JOIN public.enrollments e ON c.id = e.class_id
GROUP BY c.id, p.full_name, p.avatar_url, ct.name, ct.surface, cc.name, cc.color, cc.level;
```

### 6.2 Funciones de Base de Datos

```sql
-- Función: obtener clases disponibles para un alumno en un rango de fechas
CREATE OR REPLACE FUNCTION get_available_classes(
  p_start_date DATE,
  p_end_date DATE,
  p_student_id UUID DEFAULT NULL
)
RETURNS TABLE (
  class_id UUID,
  title TEXT,
  coach_name TEXT,
  court_name TEXT,
  category_name TEXT,
  category_color TEXT,
  start_datetime TIMESTAMPTZ,
  end_datetime TIMESTAMPTZ,
  price DECIMAL,
  available_spots INTEGER,
  is_enrolled BOOLEAN
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    ca.id,
    ca.title,
    ca.coach_name,
    ca.court_name,
    ca.category_name,
    ca.category_color,
    ca.start_datetime,
    ca.end_datetime,
    ca.price,
    ca.available_spots::INTEGER,
    CASE 
      WHEN p_student_id IS NOT NULL THEN
        EXISTS(
          SELECT 1 FROM public.enrollments e 
          WHERE e.class_id = ca.id 
          AND e.student_id = p_student_id 
          AND e.status = 'confirmed'
        )
      ELSE FALSE
    END AS is_enrolled
  FROM public.classes_with_availability ca
  WHERE 
    ca.start_datetime::DATE >= p_start_date
    AND ca.start_datetime::DATE <= p_end_date
    AND ca.status = 'scheduled'
  ORDER BY ca.start_datetime ASC;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Función: resumen financiero de un alumno
CREATE OR REPLACE FUNCTION get_student_payment_summary(p_student_id UUID)
RETURNS JSON AS $$
DECLARE
  result JSON;
BEGIN
  SELECT json_build_object(
    'total_paid', COALESCE(SUM(final_amount) FILTER (WHERE status = 'paid'), 0),
    'total_pending', COALESCE(SUM(final_amount) FILTER (WHERE status = 'pending'), 0),
    'total_overdue', COALESCE(SUM(final_amount) FILTER (WHERE status = 'overdue'), 0),
    'payments_count', COUNT(*),
    'overdue_count', COUNT(*) FILTER (WHERE status = 'overdue')
  ) INTO result
  FROM public.payments
  WHERE student_id = p_student_id;
  
  RETURN result;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## 7. Autenticación y Autorización

### 7.1 Configuración de Supabase Auth

```typescript
// src/services/supabase.ts
import { createClient } from '@supabase/supabase-js';
import * as SecureStore from 'expo-secure-store';
import { Database } from '@/types/database.types';

const ExpoSecureStoreAdapter = {
  getItem: (key: string) => SecureStore.getItemAsync(key),
  setItem: (key: string, value: string) => SecureStore.setItemAsync(key, value),
  removeItem: (key: string) => SecureStore.deleteItemAsync(key),
};

export const supabase = createClient<Database>(
  process.env.EXPO_PUBLIC_SUPABASE_URL!,
  process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!,
  {
    auth: {
      storage: ExpoSecureStoreAdapter,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false,
    },
  }
);
```

### 7.2 Métodos de Autenticación

| Método | Implementación | Notas |
|--------|---------------|-------|
| Email + Contraseña | `supabase.auth.signUp()` / `signInWithPassword()` | Principal |
| Google OAuth | `supabase.auth.signInWithOAuth({ provider: 'google' })` | Opcional |
| Recuperar contraseña | `supabase.auth.resetPasswordForEmail()` | Email con link |
| Actualizar contraseña | `supabase.auth.updateUser()` | Desde perfil |
| Cerrar sesión | `supabase.auth.signOut()` | Limpia SecureStore |

### 7.3 Trigger de Creación de Perfil

```sql
-- Trigger que crea un perfil automáticamente al registrar usuario
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, email, full_name, role)
  VALUES (
    NEW.id,
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'full_name', split_part(NEW.email, '@', 1)),
    COALESCE(NEW.raw_user_meta_data->>'role', 'student')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

### 7.4 Row Level Security (RLS) Policies

```sql
-- Habilitar RLS en todas las tablas
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.classes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.enrollments ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.payments ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.notifications ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.courts ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.class_categories ENABLE ROW LEVEL SECURITY;

-- ========================
-- POLICIES: profiles
-- ========================
-- Todo usuario autenticado puede ver todos los perfiles (para mostrar coaches)
CREATE POLICY "Profiles are viewable by authenticated users"
  ON public.profiles FOR SELECT
  USING (auth.role() = 'authenticated');

-- Solo el propio usuario puede actualizar su perfil
CREATE POLICY "Users can update own profile"
  ON public.profiles FOR UPDATE
  USING (auth.uid() = id);

-- Solo admins pueden insertar y eliminar perfiles
CREATE POLICY "Admins can manage all profiles"
  ON public.profiles FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role = 'admin'
    )
  );

-- ========================
-- POLICIES: classes
-- ========================
-- Cualquier usuario autenticado puede ver clases
CREATE POLICY "Classes are viewable by authenticated users"
  ON public.classes FOR SELECT
  USING (auth.role() = 'authenticated');

-- Solo admins y coaches pueden crear clases
CREATE POLICY "Admins and coaches can create classes"
  ON public.classes FOR INSERT
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role IN ('admin', 'coach')
    )
  );

-- Admins pueden editar cualquier clase; coaches solo las suyas
CREATE POLICY "Admins can update any class, coaches their own"
  ON public.classes FOR UPDATE
  USING (
    EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role = 'admin'
    )
    OR coach_id = auth.uid()
  );

-- Solo admins pueden eliminar clases
CREATE POLICY "Only admins can delete classes"
  ON public.classes FOR DELETE
  USING (
    EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role = 'admin'
    )
  );

-- ========================
-- POLICIES: enrollments
-- ========================
-- Alumnos ven sus propias inscripciones; admins/coaches ven todas
CREATE POLICY "Students see own enrollments, admins see all"
  ON public.enrollments FOR SELECT
  USING (
    student_id = auth.uid()
    OR EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role IN ('admin', 'coach')
    )
  );

-- Alumnos pueden inscribirse; admins pueden inscribir a cualquiera
CREATE POLICY "Students can enroll themselves, admins can enroll anyone"
  ON public.enrollments FOR INSERT
  WITH CHECK (
    student_id = auth.uid()
    OR EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role = 'admin'
    )
  );

-- Alumnos pueden cancelar sus propias inscripciones; admins cualquiera
CREATE POLICY "Students can update own enrollments, admins all"
  ON public.enrollments FOR UPDATE
  USING (
    student_id = auth.uid()
    OR EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role = 'admin'
    )
  );

-- ========================
-- POLICIES: payments
-- ========================
-- Alumnos ven sus propios pagos; admins ven todos
CREATE POLICY "Students see own payments, admins see all"
  ON public.payments FOR SELECT
  USING (
    student_id = auth.uid()
    OR EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role = 'admin'
    )
  );

-- Solo admins pueden gestionar pagos
CREATE POLICY "Only admins can manage payments"
  ON public.payments FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM public.profiles
      WHERE id = auth.uid() AND role = 'admin'
    )
  );

-- ========================
-- POLICIES: notifications
-- ========================
-- Los usuarios ven solo sus propias notificaciones
CREATE POLICY "Users see own notifications"
  ON public.notifications FOR ALL
  USING (user_id = auth.uid());

-- ========================
-- POLICIES: courts y categories (solo lectura para todos)
-- ========================
CREATE POLICY "Courts are viewable by authenticated users"
  ON public.courts FOR SELECT USING (auth.role() = 'authenticated');

CREATE POLICY "Admins manage courts"
  ON public.courts FOR ALL
  USING (EXISTS (SELECT 1 FROM public.profiles WHERE id = auth.uid() AND role = 'admin'));

CREATE POLICY "Categories are viewable by authenticated users"
  ON public.class_categories FOR SELECT USING (auth.role() = 'authenticated');
```

---

## 8. Módulos y Funcionalidades

### 8.1 Módulo de Autenticación

**Pantalla de Login (`app/(auth)/login.tsx`)**
- Formulario con campos: Email y Contraseña
- Validación con Zod: email válido, password mínimo 8 caracteres
- Botón "Olvidé mi contraseña" → navega a forgot-password
- Botón "Crear cuenta" → navega a register
- Manejo de errores con mensajes claros en español
- Loading state durante autenticación
- Redirección automática si ya hay sesión activa

**Pantalla de Registro (`app/(auth)/register.tsx`)**
- Campos: Nombre completo, Email, Teléfono (opcional), Contraseña, Confirmar contraseña
- Validación completa con Zod
- Creación automática de perfil vía trigger de Supabase
- Mensaje de bienvenida y redirección al home

**Pantalla de Recuperar Contraseña (`app/(auth)/forgot-password.tsx`)**
- Campo: Email
- Envía email con link de recuperación via Supabase Auth
- Confirmación visual al enviarse

### 8.2 Módulo de Clases

**Lista de Clases disponibles**
- Listado de todas las clases con status `scheduled`
- Filtros: por categoría, por coach, por cancha, por día
- Buscador por nombre de clase
- Cada tarjeta muestra: título, coach, horario, cancha, cupos disponibles, precio
- Indicador visual de cupos (verde: disponible, amarillo: últimos cupos, rojo: lleno)
- Acceso rápido a inscripción

**Detalle de Clase (`app/class/[id].tsx`)**
- Información completa de la clase
- Lista de alumnos inscritos (visible para coach/admin)
- Botón "Inscribirme" (si el alumno no está inscrito y hay cupos)
- Botón "Cancelar inscripción" (si ya está inscrito)
- Indicador de lista de espera si está llena
- Información del coach con avatar

**Formulario de Clase (Admin/Coach) (`app/(admin)/classes/create.tsx`)**
- Campos: título, descripción, coach (selector), cancha (selector), categoría, fecha inicio, hora inicio, duración, máximo alumnos, precio
- Opción de clase recurrente con selector de días de la semana y fecha de fin
- Validaciones completas

### 8.3 Módulo de Calendario

**Vista Diaria (`src/components/calendar/DayView.tsx`)**
- Timeline vertical de 08:00 a 22:00
- Clases representadas como bloques en la línea de tiempo
- Color por categoría/nivel
- Tap en clase → detalle

**Vista Semanal (`src/components/calendar/WeekView.tsx`)**
- Grilla de 7 columnas (lun-dom)
- Clases como bloques en cada día
- Scroll horizontal para navegar semanas

**Vista Mensual (`src/components/calendar/MonthView.tsx`)**
- Calendario tipo grid mes completo
- Puntos de color en días con clases
- Tap en día → vista diaria de ese día
- Navegación entre meses con flechas

### 8.4 Módulo de Inscripciones

**Mis Clases (`app/(tabs)/my-classes.tsx`)**
- Lista de inscripciones del alumno logueado
- Filtros: próximas / pasadas / canceladas
- Estado de asistencia (si aplica)
- Opción de cancelar inscripción (con confirmación)
- Historial completo ordenado por fecha

**Lógica de Inscripción**
```typescript
// src/services/enrollments.service.ts
export const enrollInClass = async (classId: string, studentId: string) => {
  // 1. Verificar que la clase existe y está scheduled
  // 2. Verificar que el alumno no está ya inscrito
  // 3. Verificar que hay cupos disponibles
  // 4. Si no hay cupos, ofrecer lista de espera
  // 5. Crear el enrollment con status 'confirmed'
  // 6. Crear notificación de confirmación
  // 7. Crear registro de pago pendiente si la clase tiene precio
};
```

### 8.5 Módulo de Pagos

**Estado de Cuenta del Alumno (`app/(tabs)/payments.tsx`)**
- Resumen visual: total pagado, pendiente, vencido
- Lista de todos sus pagos ordenados por fecha
- Badge de estado: Pagado (verde), Pendiente (amarillo), Vencido (rojo)
- Detalle de cada pago: descripción, monto, fecha vencimiento, método de pago

**Gestión de Pagos (Admin) (`app/(admin)/payments/index.tsx`)**
- Lista de todos los pagos del sistema
- Filtros: por estado, por alumno, por rango de fechas
- Acción: marcar como pagado (con método y referencia)
- Acción: registrar pago manual
- Exportar reporte (futuro)

### 8.6 Módulo de Perfil

**Perfil del Usuario (`app/(tabs)/profile.tsx`)**
- Foto de perfil (con opción de cambiar desde galería o cámara)
- Información personal editable: nombre, teléfono, dirección
- Información de contacto de emergencia
- Cambiar contraseña
- Cerrar sesión
- Versión de la app

### 8.7 Panel Administrativo

**Dashboard Admin (`app/(admin)/dashboard.tsx`)**
- Métricas del día: clases hoy, alumnos activos, pagos pendientes
- Clases próximas (próximas 24h)
- Alertas: pagos vencidos, clases sin coach asignado
- Accesos rápidos: crear clase, ver pagos, gestionar alumnos

**Gestión de Alumnos (`app/(admin)/students/`)**
- Lista de todos los alumnos activos
- Buscador por nombre o email
- Ver detalle: inscripciones, pagos, historial
- Editar información del alumno
- Activar/desactivar alumno

**Gestión de Coaches (`app/(admin)/coaches/`)**
- Lista de coaches
- Ver horario del coach (clases asignadas)
- Editar información
- Asignar/desasignar clases

---


### 8.8 Reglas de Negocio Operacionales (v1)

> Objetivo: que el producto se comporte como “escuela de tenis” (operación real) y no como “agenda bonita”.

#### 8.8.1 Capacidad, cupos e inscripción
- Cada clase define `max_students` (editable por ADMIN).
- El sistema calcula disponibilidad con `enrolled_count` y `available_spots` (vista `classes_with_availability`).
- Inscripción permitida solo si:
  - `classes.status = 'scheduled'`
  - `available_spots > 0`
- Si no hay cupos:
  - El alumno puede quedar en **lista de espera** (`enrollments.status = 'waitlist'`).
- Si el ADMIN reduce `max_students` por debajo de `enrolled_count`:
  - **No se expulsa automáticamente** a alumnos (evitamos daños colaterales).
  - La clase queda marcada como “sobre-capacidad” en UI (badge) y requiere acción del ADMIN (mover/reagendar/abrir nueva clase).

#### 8.8.2 Horario operativo y validaciones de agenda
- Operación estándar: **08:00–22:00**.
- Validación de creación/edición de clases:
  - `start_datetime` y `end_datetime` deben caer dentro del rango operativo (a menos que `admin_override=true` en el request).
  - `end_datetime` debe ser estrictamente mayor que `start_datetime`.
- **No solape por cancha**:
  - No se permite traslape horario en la misma cancha para clases con `status in ('scheduled','in_progress')`.
  - Recomendación técnica: constraint de exclusión con `tstzrange()`.

SQL recomendado:

```sql
ALTER TABLE public.classes
ADD CONSTRAINT classes_no_overlap_per_court
EXCLUDE USING gist (
  court_id WITH =,
  tstzrange(start_datetime, end_datetime, '[)') WITH &&
)
WHERE (status IN ('scheduled', 'in_progress'));
```

#### 8.8.3 Reagendamiento por lluvia / fuerza mayor
**Principio:** ante lluvia u otro evento operativo, la clase original se cancela y se crea una clase reemplazo trazable.

- En la clase original:
  - `status='cancelled'`
  - `cancellation_reason` ∈ {`weather`, `maintenance`, `coach_absence`, `force_majeure`, `other`}
- Se crea una clase reemplazo (`replacement`) con nueva fecha/hora/cancha/coach.
- Se registra la relación en una tabla de vínculo.

Tabla nueva: `class_reschedules`

```sql
CREATE TABLE public.class_reschedules (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  original_class_id     UUID NOT NULL REFERENCES public.classes(id) ON DELETE CASCADE,
  replacement_class_id  UUID NOT NULL REFERENCES public.classes(id) ON DELETE CASCADE,
  reason                TEXT NOT NULL,
  rescheduled_by        UUID NOT NULL REFERENCES public.profiles(id),
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (original_class_id),
  UNIQUE (replacement_class_id)
);

CREATE INDEX idx_class_reschedules_original ON public.class_reschedules(original_class_id);
```

**Migración de alumnos**
- Para cada inscripción `confirmed` de la original:
  - Si hay cupo en reemplazo → crear inscripción `confirmed`.
  - Si no hay cupo → crear inscripción `waitlist`.
- Se notifican alumnos (push + in-app): `class_updated` o `class_cancelled` + link a clase reemplazo.

#### 8.8.4 Pagos y “créditos” (para no cobrar doble ni perder plata)
Estados existentes de pago se mantienen (`pending`, `paid`, `overdue`, `cancelled`, `refunded`).

**Reglas mínimas**
- Al inscribirse:
  - Se crea `payment` en `pending` con `due_date = start_datetime::date` (vencimiento el día de la clase).
- Al marcar pago como `paid`:
  - El alumno queda habilitado para participar (y el ADMIN puede ver estado en la ficha de clase/alumno).

**Reagendamiento**
- Si el pago de la clase original está `paid`:
  - Se considera **trasladado** a la clase reemplazo (no se genera un segundo cobro).
  - Se agrega nota/bitácora (campo `notes`) y se mantiene trazabilidad vía `class_reschedules`.
- Si el pago está `pending`:
  - Se mantiene `pending` y se actualiza referencia (enrollment/clase reemplazo) según implementación.

**Créditos (recomendado)**
Para casos de cancelación sin clase reemplazo o compensaciones (lluvia, descuentos, promo), se recomienda una tabla de saldo a favor:

```sql
CREATE TABLE public.student_credits (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id        UUID NOT NULL REFERENCES public.profiles(id) ON DELETE CASCADE,
  amount            DECIMAL(10,2) NOT NULL,
  currency          TEXT NOT NULL DEFAULT 'CLP',
  reason            TEXT NOT NULL,
  related_payment_id UUID REFERENCES public.payments(id),
  related_class_id   UUID REFERENCES public.classes(id),
  expires_at         DATE,
  created_by         UUID REFERENCES public.profiles(id),
  created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_student_credits_student ON public.student_credits(student_id);
```

#### 8.8.5 Administración “asistida” de alumnos (ADMIN crea usuarios)
El ADMIN puede ingresar alumnos “a mano” para operación (cliente llega, paga, y listo):
- UI Admin/Alumnos → “Crear Alumno”
- Backend:
  - Edge Function con Service Role crea usuario en Supabase Auth (email) y dispara el trigger existente de creación de `profiles`.
  - Se envía link de onboarding (“set password” / invitación) al email del alumno.

#### 8.8.6 Scheduler operativo (mantenimiento diario)
Se requiere un proceso programado (Edge scheduled / cron externo) para:
- Marcar pagos vencidos: `pending` con `due_date < today` → `overdue`
- Disparar notificaciones:
  - Recordatorio 24h y 2h antes de la clase
  - `payment_due` y `payment_overdue`
- Garantizar idempotencia (evitar doble envío)

Edge Functions sugeridas:
- `reschedule-class`
- `daily-maintenance`

#### 8.8.7 Bloqueos de cancha (v1.1 recomendado)
Para mantenciones/torneos/eventos, agregar `facility_blocks` (v1.1):
- `court_id`, `start_datetime`, `end_datetime`, `reason`, `created_by`
- Se visualiza en calendario como bloque no reservable
- Evita crear clases en esos intervalos

## 9. Diseño de Interfaces (UI/UX)

### 9.1 Sistema de Diseño

#### Paleta de Colores
```typescript
// src/theme/colors.ts
export const colors = {
  // Primarios
  primary: {
    50:  '#EEF2FF',
    100: '#E0E7FF',
    300: '#A5B4FC',
    500: '#6366F1',   // Color principal (Indigo)
    600: '#4F46E5',
    700: '#4338CA',
    900: '#312E81',
  },
  
  // Secundarios (verde tenis)
  secondary: {
    50:  '#F0FDF4',
    100: '#DCFCE7',
    300: '#86EFAC',
    500: '#22C55E',   // Verde
    600: '#16A34A',
    700: '#15803D',
  },
  
  // Semánticos
  success: '#22C55E',
  warning: '#F59E0B',
  error:   '#EF4444',
  info:    '#3B82F6',
  
  // Neutros
  gray: {
    50:  '#F9FAFB',
    100: '#F3F4F6',
    200: '#E5E7EB',
    300: '#D1D5DB',
    400: '#9CA3AF',
    500: '#6B7280',
    600: '#4B5563',
    700: '#374151',
    800: '#1F2937',
    900: '#111827',
  },
  
  // Superficie
  white: '#FFFFFF',
  background: '#F9FAFB',
  surface: '#FFFFFF',
  
  // Estados de clase (por nivel/categoría)
  level: {
    beginner:     '#22C55E',  // Verde
    intermediate: '#3B82F6',  // Azul
    advanced:     '#F59E0B',  // Amarillo
    elite:        '#EF4444',  // Rojo
  },
  
  // Estados de pago
  payment: {
    paid:    '#22C55E',
    pending: '#F59E0B',
    overdue: '#EF4444',
  },
};
```

#### Tipografía
```typescript
// src/theme/typography.ts
export const typography = {
  fontFamily: {
    regular:  'Inter-Regular',
    medium:   'Inter-Medium',
    semibold: 'Inter-SemiBold',
    bold:     'Inter-Bold',
  },
  fontSize: {
    xs:   10,
    sm:   12,
    base: 14,
    md:   16,
    lg:   18,
    xl:   20,
    '2xl': 24,
    '3xl': 30,
    '4xl': 36,
  },
  lineHeight: {
    tight:  1.25,
    normal: 1.5,
    loose:  1.75,
  },
};
```

#### Espaciado
```typescript
// src/theme/spacing.ts
export const spacing = {
  xs:  4,
  sm:  8,
  md:  12,
  lg:  16,
  xl:  20,
  '2xl': 24,
  '3xl': 32,
  '4xl': 40,
  '5xl': 48,
  '6xl': 64,
};
```

### 9.2 Componentes UI Principales

#### ClassCard
```tsx
// src/components/class/ClassCard.tsx
// Props:
interface ClassCardProps {
  class: ClassWithAvailability;
  onPress: () => void;
  showEnrollButton?: boolean;
  isEnrolled?: boolean;
}
// Muestra: categoría (badge color), título, coach, horario, cancha, cupos
// Diseño: card con sombra suave, borde izquierdo del color de la categoría
```

#### PaymentBadge
```tsx
// src/components/payment/PaymentBadge.tsx
// Badge que muestra el estado del pago con color semántico
// paid → verde, pending → amarillo, overdue → rojo
```

#### CalendarStrip (Selector de fecha horizontal)
```tsx
// Tira horizontal de fechas en la parte superior del calendario
// Muestra 7 días, el seleccionado resaltado
// Navegar con swipe o flechas
```

### 9.3 Navegación

```
RootLayout
├── (auth)/          ← Stack sin tab bar
│   ├── login
│   ├── register
│   └── forgot-password
└── (tabs)/          ← Tab bar con 5 tabs
    ├── index (🏠 Inicio)
    ├── schedule (📅 Horarios)
    ├── my-classes (🎾 Mis Clases)
    ├── payments (💳 Pagos)
    └── profile (👤 Perfil)

Stack sobre tabs:
└── class/[id]       ← Pantalla detalle clase

Stack admin (accesible desde perfil o botón flotante):
└── (admin)/
    ├── dashboard
    ├── classes/*
    ├── students/*
    ├── coaches/*
    └── payments/*
```

#### Tab Bar
- 5 tabs principales con iconos y labels
- Tab activo: color primario
- Tab inactivo: gris
- Badge en "Mis Clases" con cantidad de próximas clases
- Badge en "Pagos" con cantidad de pagos pendientes/vencidos

---

## 10. API y Servicios

### 10.1 Auth Service

```typescript
// src/services/auth.service.ts
export const authService = {
  signIn: (email: string, password: string) =>
    supabase.auth.signInWithPassword({ email, password }),

  signUp: (email: string, password: string, fullName: string) =>
    supabase.auth.signUp({
      email,
      password,
      options: { data: { full_name: fullName } },
    }),

  signOut: () => supabase.auth.signOut(),

  resetPassword: (email: string) =>
    supabase.auth.resetPasswordForEmail(email),

  getSession: () => supabase.auth.getSession(),

  onAuthStateChange: (callback: (event: string, session: Session | null) => void) =>
    supabase.auth.onAuthStateChange(callback),
};
```

### 10.2 Classes Service

```typescript
// src/services/classes.service.ts
export const classesService = {
  // Obtener clases con disponibilidad (usa la vista)
  getClassesInRange: (startDate: string, endDate: string, studentId?: string) =>
    supabase.rpc('get_available_classes', {
      p_start_date: startDate,
      p_end_date: endDate,
      p_student_id: studentId,
    }),

  // Obtener clase por ID con detalles
  getClassById: (id: string) =>
    supabase.from('classes_with_availability').select('*').eq('id', id).single(),

  // Crear clase (solo admin/coach)
  createClass: (data: CreateClassDTO) =>
    supabase.from('classes').insert(data).select().single(),

  // Actualizar clase
  updateClass: (id: string, data: UpdateClassDTO) =>
    supabase.from('classes').update(data).eq('id', id).select().single(),

  // Cancelar clase
  cancelClass: (id: string, reason: string) =>
    supabase.from('classes').update({
      status: 'cancelled',
      cancellation_reason: reason,
    }).eq('id', id),

  // Obtener clases de un coach
  getCoachClasses: (coachId: string, from: string, to: string) =>
    supabase.from('classes_with_availability')
      .select('*')
      .eq('coach_id', coachId)
      .gte('start_datetime', from)
      .lte('start_datetime', to)
      .order('start_datetime'),
};
```

### 10.3 Enrollments Service

```typescript
// src/services/enrollments.service.ts
export const enrollmentsService = {
  // Inscribir alumno en clase
  enroll: async (classId: string, studentId: string): Promise<EnrollResult> => {
    // Verificar disponibilidad
    const { data: classData } = await supabase
      .from('classes_with_availability')
      .select('available_spots, price, status')
      .eq('id', classId)
      .single();

    if (classData?.status !== 'scheduled') throw new Error('Clase no disponible');
    if (classData.available_spots <= 0) throw new Error('No hay cupos disponibles');

    // Crear inscripción
    const { data: enrollment, error } = await supabase
      .from('enrollments')
      .insert({ class_id: classId, student_id: studentId, enrolled_by: studentId })
      .select()
      .single();

    if (error) throw error;

    // Crear pago pendiente si la clase tiene precio
    if (classData.price > 0) {
      await supabase.from('payments').insert({
        student_id: studentId,
        class_id: classId,
        enrollment_id: enrollment.id,
        amount: classData.price,
        status: 'pending',
        due_date: new Date().toISOString().split('T')[0], // Hoy por defecto
        description: `Pago por clase`,
      });
    }

    return enrollment;
  },

  // Cancelar inscripción
  cancel: (enrollmentId: string, reason?: string) =>
    supabase.from('enrollments').update({
      status: 'cancelled',
      cancelled_at: new Date().toISOString(),
      cancel_reason: reason,
    }).eq('id', enrollmentId),

  // Inscripciones del alumno
  getStudentEnrollments: (studentId: string) =>
    supabase.from('enrollments')
      .select(`*, classes(*, courts(name), profiles!coach_id(full_name, avatar_url))`)
      .eq('student_id', studentId)
      .order('enrolled_at', { ascending: false }),

  // Alumnos inscritos en una clase
  getClassEnrollments: (classId: string) =>
    supabase.from('enrollments')
      .select(`*, profiles!student_id(full_name, email, avatar_url, phone)`)
      .eq('class_id', classId)
      .eq('status', 'confirmed'),

  // Marcar asistencia (coach/admin)
  markAttendance: (enrollmentId: string, attendance: AttendanceStatus) =>
    supabase.from('enrollments').update({
      attendance,
      attended_at: attendance === 'present' ? new Date().toISOString() : null,
    }).eq('id', enrollmentId),
};
```

### 10.4 Payments Service

```typescript
// src/services/payments.service.ts
export const paymentsService = {
  // Resumen de pagos del alumno
  getStudentSummary: (studentId: string) =>
    supabase.rpc('get_student_payment_summary', { p_student_id: studentId }),

  // Lista de pagos del alumno
  getStudentPayments: (studentId: string) =>
    supabase.from('payments')
      .select(`*, classes(title, start_datetime)`)
      .eq('student_id', studentId)
      .order('created_at', { ascending: false }),

  // Todos los pagos (admin)
  getAllPayments: (filters?: PaymentFilters) => {
    let query = supabase.from('payments')
      .select(`*, profiles!student_id(full_name, email), classes(title)`);
    if (filters?.status) query = query.eq('status', filters.status);
    if (filters?.studentId) query = query.eq('student_id', filters.studentId);
    return query.order('created_at', { ascending: false });
  },

  // Marcar como pagado (admin)
  markAsPaid: (paymentId: string, method: PaymentMethod, reference?: string) =>
    supabase.from('payments').update({
      status: 'paid',
      payment_method: method,
      payment_reference: reference,
      paid_at: new Date().toISOString(),
    }).eq('id', paymentId),

  // Registrar pago manual (admin)
  createManualPayment: (data: CreatePaymentDTO) =>
    supabase.from('payments').insert(data).select().single(),
};
```

### 10.5 Custom Hooks (React Query)

```typescript
// src/hooks/useClasses.ts
export const useClasses = (startDate: string, endDate: string) => {
  const { user } = useAuth();
  
  return useQuery({
    queryKey: ['classes', startDate, endDate, user?.id],
    queryFn: () => classesService.getClassesInRange(startDate, endDate, user?.id),
    staleTime: 5 * 60 * 1000, // 5 minutos
  });
};

export const useClass = (id: string) => {
  return useQuery({
    queryKey: ['class', id],
    queryFn: () => classesService.getClassById(id),
  });
};

export const useEnrollMutation = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ classId, studentId }: { classId: string; studentId: string }) =>
      enrollmentsService.enroll(classId, studentId),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['classes'] });
      queryClient.invalidateQueries({ queryKey: ['enrollments'] });
      queryClient.invalidateQueries({ queryKey: ['payments'] });
    },
  });
};

// src/hooks/useAuth.ts
export const useAuth = () => {
  const { session, profile, isLoading } = useAuthStore();
  return {
    user: session?.user,
    profile,
    isLoading,
    isAdmin: profile?.role === 'admin',
    isCoach: profile?.role === 'coach',
    isStudent: profile?.role === 'student',
  };
};
```

---

## 11. Requerimientos No Funcionales

### 11.1 Rendimiento
- Tiempo de carga inicial < 3 segundos en conexión 4G
- Tiempo de respuesta de operaciones UI < 200ms
- Las listas de clases se paginarán (20 items por página)
- Uso de `FlatList` con `keyExtractor` y `getItemLayout` para listas largas
- Imágenes cargadas con lazy loading y caché (Expo Image)

### 11.2 Disponibilidad
- La app consume Supabase con SLA del 99.9%
- Manejo de errores de red con reintentos automáticos (React Query)
- Mensajes de error claros y comprensibles para el usuario

### 11.3 Seguridad
- Tokens de sesión almacenados en SecureStore (encriptado)
- Todas las operaciones de datos validadas por RLS en PostgreSQL
- No se almacenan datos sensibles en AsyncStorage sin encriptación
- HTTPS obligatorio para todas las comunicaciones
- Rate limiting en Supabase Edge Functions

### 11.4 Accesibilidad
- Labels de accesibilidad (`accessibilityLabel`) en todos los botones e inputs
- Contraste de colores mínimo WCAG AA
- Soporte de Dynamic Type (tamaños de fuente del sistema)
- Soporte de modo oscuro (Dark Mode) mediante `useColorScheme`

### 11.5 Compatibilidad
- iOS 13+
- Android 8+ (API Level 26)
- Tabletas: layout adaptativo (no optimizado para tablet pero funcional)
- RTL: no requerido en v1.0

---

## 12. Flujos de Usuario

### 12.1 Flujo de Registro e Inicio de Sesión

```
[App abre] → ¿Hay sesión guardada?
    ├── SÍ → Cargar perfil → Ir a Home
    └── NO → Pantalla Login
              ├── [Login exitoso] → Cargar perfil → Ir a Home
              └── [¿No tienes cuenta?] → Pantalla Register
                    └── [Registro exitoso] → Crear perfil → Ir a Home
```

### 12.2 Flujo de Inscripción en Clase

```
Home / Calendario → [Tap en clase disponible]
    → Pantalla Detalle Clase
        ├── ¿Ya está inscrito? → Mostrar "Ya inscrito" + botón Cancelar
        ├── ¿Cupos disponibles?
        │   ├── SÍ → Botón "Inscribirme"
        │   │         → Modal Confirmación (fecha, hora, precio)
        │   │         → [Confirmar] → Inscripción creada → Toast éxito
        │   │                          → Actualizar lista de cupos
        │   └── NO → Botón "Unirme a lista de espera"
        └── ¿Clase pasada? → Solo mostrar info, sin acciones
```

### 12.3 Flujo de Pago (Admin)

```
Panel Admin → Pagos → [Filtrar pendientes]
    → [Tap en pago] → Detalle de pago
        → Botón "Registrar Pago"
            → Modal: seleccionar método + número referencia (opcional)
            → [Confirmar] → Pago marcado como pagado
                             → Notificación al alumno
```

### 12.4 Flujo de Creación de Clase (Admin/Coach)

```
Panel Admin → Clases → [+ Nueva Clase]
    → Formulario Clase
        → Seleccionar: Coach, Cancha, Categoría
        → Definir: Título, Fecha, Hora inicio, Duración, Cupos, Precio
        → ¿Es recurrente?
            ├── SÍ → Seleccionar días de la semana + fecha fin
            └── NO → Solo esa fecha
        → [Guardar] → Clases creadas → Lista actualizada
```

---

## 13. Notificaciones

### 13.1 Tipos de Notificaciones Push

| Tipo | Trigger | Destinatario |
|------|---------|-------------|
| `enrollment_confirmed` | Al inscribirse en clase | Alumno |
| `class_reminder` | 24h y 2h antes de la clase | Alumno inscrito |
| `class_cancelled` | Al cancelar una clase | Todos los inscritos |
| `class_updated` | Al modificar horario/cancha | Todos los inscritos |
| `payment_due` | 3 días antes del vencimiento | Alumno con pago pendiente |
| `payment_overdue` | El día del vencimiento | Alumno con pago vencido |
| `enrollment_waitlist_promoted` | Al liberarse cupo y promovido de espera | Alumno en espera |

### 13.2 Configuración de Expo Notifications

```typescript
// src/services/notifications.service.ts
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

export const registerForPushNotifications = async (userId: string) => {
  if (!Device.isDevice) return null;

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== 'granted') return null;

  const token = (await Notifications.getExpoPushTokenAsync()).data;

  // Guardar token en perfil del usuario
  await supabase.from('profiles').update({ push_token: token }).eq('id', userId);

  return token;
};
```

### 13.3 Edge Function para Envío de Notificaciones

```typescript
// supabase/functions/send-notification/index.ts
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts';

serve(async (req) => {
  const { userId, title, body, data, type } = await req.json();

  // 1. Obtener push token del usuario
  const { data: profile } = await supabase
    .from('profiles').select('push_token').eq('id', userId).single();

  // 2. Guardar notificación en BD
  await supabase.from('notifications').insert({ userId, title, body, type, data });

  // 3. Enviar push notification via Expo Push API
  if (profile?.push_token) {
    await fetch('https://exp.host/--/api/v2/push/send', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ to: profile.push_token, title, body, data }),
    });
  }

  return new Response(JSON.stringify({ success: true }), { status: 200 });
});
```

---

## 14. Seguridad

### 14.1 Checklist de Seguridad

- [x] RLS habilitado en todas las tablas de Supabase
- [x] Tokens de auth almacenados en SecureStore (iOS Keychain / Android Keystore)
- [x] Variables de entorno nunca commiteadas al repositorio
- [x] Validación de datos en cliente (Zod) Y en base de datos (constraints SQL)
- [x] No exponer datos sensibles en logs
- [x] Uso del Anon Key de Supabase en el cliente (nunca el Service Role Key)
- [x] Políticas RLS que siguen el principio de mínimo privilegio

### 14.2 Variables Sensibles

El archivo `.env.local` NUNCA debe commitearse. Usar `.gitignore`:

```
.env.local
.env.*.local
```

El `Service Role Key` de Supabase solo se usa en Edge Functions del lado servidor, nunca en el cliente móvil.

---

## 15. Variables de Entorno

### 15.1 Archivo `.env.example`

```bash
# Supabase
EXPO_PUBLIC_SUPABASE_URL=https://tu-proyecto.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=tu-anon-key-publica

# Solo para Edge Functions (NO en cliente móvil)
SUPABASE_SERVICE_ROLE_KEY=tu-service-role-key-secreta

# Expo (para notificaciones)
EXPO_PUBLIC_PROJECT_ID=tu-expo-project-id

# Configuración de la escuela
EXPO_PUBLIC_SCHOOL_NAME=Academia de Tenis XYZ
EXPO_PUBLIC_SCHOOL_PHONE=+56 9 1234 5678
EXPO_PUBLIC_CURRENCY=CLP
EXPO_PUBLIC_TIMEZONE=America/Santiago
# Operación
EXPO_PUBLIC_OPERATING_HOURS_START=08:00
EXPO_PUBLIC_OPERATING_HOURS_END=22:00
```

### 15.2 `app.json` Configuration

```json
{
  "expo": {
    "name": "Escuela de Tenis",
    "slug": "escuela-tenis",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/images/icon.png",
    "userInterfaceStyle": "automatic",
    "splash": {
      "image": "./assets/images/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#4F46E5"
    },
    "ios": {
      "supportsTablet": false,
      "bundleIdentifier": "com.tuempresa.escuelatenis",
      "infoPlist": {
        "NSCameraUsageDescription": "Para actualizar tu foto de perfil",
        "NSPhotoLibraryUsageDescription": "Para seleccionar tu foto de perfil"
      }
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/images/adaptive-icon.png",
        "backgroundColor": "#4F46E5"
      },
      "package": "com.tuempresa.escuelatenis",
      "permissions": ["CAMERA", "READ_EXTERNAL_STORAGE"]
    },
    "plugins": [
      "expo-router",
      "expo-secure-store",
      [
        "expo-notifications",
        {
          "icon": "./assets/images/notification-icon.png",
          "color": "#4F46E5"
        }
      ],
      [
        "expo-image-picker",
        {
          "photosPermission": "La app necesita acceso a tus fotos para cambiar tu avatar."
        }
      ]
    ],
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

---

## 16. Guía de Implementación Paso a Paso

> Esta sección guía al IDE con IA para construir el sistema en el orden correcto.

### FASE 1: Configuración del Proyecto (Día 1)

```bash
# 1. Crear proyecto Expo con TypeScript
npx create-expo-app@latest escuela-tenis --template blank-typescript
cd escuela-tenis

# 2. Instalar Expo Router
npx expo install expo-router react-native-safe-area-context react-native-screens \
  expo-linking expo-constants expo-status-bar

# 3. Instalar dependencias principales
npx expo install @supabase/supabase-js expo-secure-store

# 4. Instalar UI y navegación
npm install react-native-paper react-native-vector-icons
npx expo install @expo/vector-icons expo-image expo-image-picker

# 5. Instalar estado y datos
npm install zustand @tanstack/react-query

# 6. Instalar formularios y validación
npm install react-hook-form zod @hookform/resolvers

# 7. Instalar calendario y fechas
npm install react-native-calendars date-fns

# 8. Instalar animaciones y gestos
npx expo install react-native-reanimated react-native-gesture-handler

# 9. Instalar notificaciones
npx expo install expo-notifications expo-device

# 10. TypeScript paths (tsconfig.json)
# Configurar path alias "@/*" → "./src/*"
```

### FASE 2: Base de Datos y Auth en Supabase (Día 1-2)

1. Crear proyecto en [supabase.com](https://supabase.com)
2. Ejecutar migraciones SQL en el orden:
   - `001_initial_schema.sql` → Crea todas las tablas
   - `002_rls_policies.sql` → Aplica políticas RLS
   - `003_functions.sql` → Crea funciones y triggers
   - `004_seed_data.sql` → Datos de prueba (canchas, categorías)
3. Configurar Auth: habilitar Email/Password, configurar redirect URLs
4. Generar tipos TypeScript: `npx supabase gen types typescript --project-id <id> > src/types/database.types.ts`

### FASE 3: Autenticación (Día 2-3)

Implementar en este orden:
1. `src/services/supabase.ts` → Cliente Supabase con SecureStore
2. `src/store/auth.store.ts` → Zustand store con session y profile
3. `src/hooks/useAuth.ts` → Hook de autenticación
4. `app/(auth)/_layout.tsx` → Layout sin tabs
5. `app/(auth)/login.tsx` → Pantalla de login
6. `app/(auth)/register.tsx` → Pantalla de registro
7. `app/(auth)/forgot-password.tsx` → Recuperar contraseña
8. `app/_layout.tsx` → Root layout con lógica de redirección auth

### FASE 4: Navegación Principal (Día 3)

1. `app/(tabs)/_layout.tsx` → Tab navigator con 5 tabs
2. `src/components/shared/ProtectedRoute.tsx` → HOC de protección por rol
3. Pantallas placeholder para cada tab

### FASE 5: Módulo de Clases y Calendario (Día 4-6)

1. `src/services/classes.service.ts`
2. `src/hooks/useClasses.ts`
3. `src/components/class/ClassCard.tsx`
4. `src/components/calendar/CalendarView.tsx` (con vistas día/semana/mes)
5. `app/(tabs)/schedule.tsx`
6. `app/class/[id].tsx`

### FASE 6: Inscripciones (Día 6-7)

1. `src/services/enrollments.service.ts`
2. `src/hooks/useEnrollments.ts`
3. Botón inscribir en `app/class/[id].tsx`
4. `app/(tabs)/my-classes.tsx`

### FASE 7: Pagos (Día 7-8)

1. `src/services/payments.service.ts`
2. `src/hooks/usePayments.ts`
3. `src/components/payment/PaymentCard.tsx`
4. `app/(tabs)/payments.tsx`

### FASE 8: Perfil (Día 8)

1. `src/services/profiles.service.ts`
2. `app/(tabs)/profile.tsx`
3. Subida de avatar a Supabase Storage

### FASE 9: Panel Admin (Día 9-11)

1. `app/(admin)/_layout.tsx`
2. `app/(admin)/dashboard.tsx`
3. CRUD de clases
4. Gestión de alumnos
5. Gestión de pagos

### FASE 10: Notificaciones (Día 11-12)

1. `src/services/notifications.service.ts`
2. Registro de token push al iniciar sesión
3. Edge Function `send-notification`
4. `app/(tabs)/profile.tsx` → Sección notificaciones

### FASE 11: Pulido y Testing (Día 12-14)

1. Manejo de errores global (Error Boundary)
2. Loading states y Skeleton screens
3. Empty states para listas vacías
4. Dark Mode
5. Testing de flujos críticos
6. Build de producción con EAS

---

## Apéndice A: Seed Data (Datos de Prueba)

```sql
-- 004_seed_data.sql

-- Categorías de clases
INSERT INTO public.class_categories (name, level, color, description) VALUES
  ('Iniciación', 1, '#22C55E', 'Para personas que nunca han jugado tenis'),
  ('Básico', 2, '#3B82F6', 'Fundamentos del tenis'),
  ('Intermedio', 3, '#F59E0B', 'Mejora técnica y táctica'),
  ('Avanzado', 4, '#EF4444', 'Alto rendimiento competitivo'),
  ('Competición', 5, '#8B5CF6', 'Preparación para torneos');

-- Canchas
INSERT INTO public.courts (name, surface, is_indoor, description) VALUES
  ('Cancha 1', 'clay', FALSE, 'Cancha principal del recinto'),
  ('Cancha 2', 'clay', FALSE, 'Cancha secundaria del recinto');

-- Planes de pago
INSERT INTO public.payment_plans (name, description, classes_count, price, validity_days) VALUES
  ('Plan 4 clases', '4 clases al mes', 4, 60000, 30),
  ('Plan 8 clases', '8 clases al mes - el más popular', 8, 110000, 30),
  ('Plan 12 clases', '12 clases al mes - máximo progreso', 12, 150000, 30),
  ('Clase suelta', 'Una clase individual sin plan', 1, 18000, 7);
```

---

## Apéndice B: Tipos TypeScript Principales

```typescript
// src/types/class.types.ts
export interface Class {
  id: string;
  title: string;
  description?: string;
  coach_id: string;
  court_id: string;
  category_id?: string;
  start_datetime: string;
  end_datetime: string;
  duration_minutes: number;
  is_recurring: boolean;
  recurrence_rule?: string;
  max_students: number;
  price: number;
  currency: string;
  status: 'scheduled' | 'in_progress' | 'completed' | 'cancelled';
  created_at: string;
  updated_at: string;
}

export interface ClassWithAvailability extends Class {
  coach_name: string;
  coach_avatar?: string;
  court_name: string;
  court_surface: string;
  category_name?: string;
  category_color?: string;
  category_level?: number;
  enrolled_count: number;
  available_spots: number;
  waitlist_count: number;
  is_enrolled?: boolean;
}

// src/types/enrollment.types.ts
export interface Enrollment {
  id: string;
  class_id: string;
  student_id: string;
  status: 'pending' | 'confirmed' | 'cancelled' | 'waitlist';
  attendance?: 'present' | 'absent' | 'late' | 'excused';
  attended_at?: string;
  enrolled_at: string;
  cancelled_at?: string;
  cancel_reason?: string;
}

export type AttendanceStatus = 'present' | 'absent' | 'late' | 'excused';

// src/types/payment.types.ts
export interface Payment {
  id: string;
  student_id: string;
  class_id?: string;
  plan_id?: string;
  enrollment_id?: string;
  amount: number;
  currency: string;
  discount: number;
  final_amount: number;
  status: 'pending' | 'paid' | 'overdue' | 'refunded' | 'cancelled';
  payment_method?: 'cash' | 'transfer' | 'card' | 'other';
  payment_reference?: string;
  due_date: string;
  paid_at?: string;
  description?: string;
  created_at: string;
}

export type PaymentMethod = 'cash' | 'transfer' | 'card' | 'other';
```

---

## Apéndice C: Comandos Útiles

```bash
# Desarrollo
npx expo start                        # Iniciar dev server
npx expo start --ios                  # iOS simulator
npx expo start --android              # Android emulator

# Supabase CLI
npx supabase login
npx supabase init
npx supabase db push                  # Aplicar migraciones
npx supabase gen types typescript \
  --project-id <PROJECT_ID> \
  > src/types/database.types.ts      # Generar tipos

# EAS (Build y Deploy)
npm install -g eas-cli
eas login
eas build --platform ios             # Build iOS
eas build --platform android         # Build Android
eas submit --platform ios            # Submit a App Store
eas submit --platform android        # Submit a Play Store

# Linting
npm run lint
npm run type-check

# Testing
npm test
npm run test:coverage
```

---

*Fin del documento SRS v1.0.0 — Escuela de Tenis*  
*Este documento está diseñado para ser consumido por un IDE con IA como contexto completo de construcción del sistema.*
