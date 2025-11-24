# Sistema de Gestión de Salas de Estudio - UCU

## Descripción

Sistema integral para la gestión de reservas, control de asistencia y generación de reportes de las salas de estudio de la Universidad Católica del Uruguay. Desarrollado como trabajo obligatorio para la materia Base de Datos 1.

## Características Principales

### Sistema de Autenticación
- Tres roles de usuario: Administrador, Docente, Alumno
- Login seguro con validación por cédula y contraseña
- Permisos diferenciados por rol

### Gestión Académica
- ABM completo de participantes, salas, reservas y sanciones
- Control de programas académicos y facultades
- Validación de tipos de sala (libre, posgrado, docente)

### Sistema de Reservas Inteligente
- Reserva por bloques horarios (8:00 AM - 11:00 PM)
- Validaciones automáticas:
  - Límite de 2 horas diarias para alumnos de grado
  - Máximo 3 reservas activas por semana
  - Verificación de sanciones activas
  - Control de tipo de sala vs usuario

### Funcionalidades Avanzadas
- Registro de asistencia en tiempo real
- Sistema automático de sanciones por no asistencia
- Cancelación de reservas
- Búsqueda de disponibilidad en tiempo real

### Sistema de Reportes BI
- 10 reportes analíticos para toma de decisiones
- Métricas de uso y ocupación
- Análisis por facultad, carrera y tipo de usuario
- Detección de patrones de uso

## Tecnologías Utilizadas

- Backend: Python 3.x
- Base de Datos: MySQL
- Frontend: Streamlit
- Conectividad: PyMySQL
- Seguridad: Hash SHA-256 para contraseñas

## Instalación y Configuración

### Prerrequisitos
- Python 3.8+
- MySQL Server 8.0+
- pip (gestor de paquetes de Python)

### 1. Clonar o Descargar el Proyecto
```bash
git clone <https://github.com/maclaragomez/FinalBD>
cd gestion-salas-ucu


### 2. Instalar Dependencias
pip install streamlit pymysql


### 3. Configurar Base de Datos

#### Ejecutar Script SQL Manualmente


### 4. Configurar Conexión a BD
Editar las credenciales en el archivo principal (en la clase SistemaSalas, método __init__):
self.host = 'localhost'
self.user = 'root'
self.password = 'tu_password'
self.database = 'gestion_salas_estudio'


### 5. Ejecutar la Aplicación
streamlit run app.py


## Roles y Permisos

### Administrador
- Gestión completa de participantes, salas y sanciones
- Acceso a todos los reportes
- Aplicación manual de sanciones
- Verificación de reservas sin asistencia

### Docente
- Gestión de participantes
- Reservas en salas libres y de docentes
- Registro de asistencia
- Visualización de reservas propias

### Alumno
- Reservas en salas permitidas según su tipo (grado/posgrado)
- Registro de asistencia a sus reservas
- Visualización de reservas propias
- Cancelación de reservas activas

## Reglas de Negocio Implementadas

### Horarios y Reservas
- Horario: 8:00 AM - 11:00 PM
- Bloques: Reservas por hora completa
- Fechas: No se permiten reservas en fechas pasadas

### Límites por Usuario
- Alumnos de grado: Máximo 2 horas diarias y 3 reservas semanales en salas libres
- Alumnos de posgrado y docentes: Sin límites en salas exclusivas
- Sanciones: 2 meses sin reservas por no asistencia

### Tipos de Sala
- Libre: Acceso para todos los usuarios
- Posgrado: Solo docentes y alumnos de posgrado
- Docente: Exclusivo para docentes

## Credenciales de Prueba

### Administrador
- Usuario: admin
- Contraseña: admin123

### Usuarios de Prueba
- 11111111 (contraseña: 11111111) - Alumno grado
- 22222222 (contraseña: 22222222) - Alumno posgrado  
- 44444444 (contraseña: 44444444) - Docente

## Estructura de la Base de Datos

### Tablas Principales
- participante: Datos personales de usuarios
- sala: Información de salas disponibles
- reserva: Registro de reservas
- turno: Bloques horarios disponibles
- programa_academico: Carreras y facultades
- sancion_participante: Historial de sanciones

### Relaciones
- Participantes asociados a programas académicos
- Reservas vinculadas a salas y turnos
- Control de asistencia por participante
- Sanciones aplicadas por no asistencia

## Funcionalidades por Módulo

### Gestión de Participantes
- Alta, baja y modificación de participantes
- Asignación a programas académicos
- Definición de roles (alumno/docente)

### Gestión de Salas
- Creación y eliminación de salas
- Definición de capacidad y tipo
- Listado completo de salas disponibles

### Gestión de Reservas
- Creación de nuevas reservas con validaciones
- Cancelación de reservas activas
- Registro de asistencia
- Consulta de disponibilidad

### Gestión de Sanciones
- Aplicación manual de sanciones
- Sistema automático por no asistencia
- Verificación de sanciones activas

### Reportes y Consultas
- Salas más reservadas
- Turnos más demandados
- Promedio de participantes por sala
- Reservas por carrera y facultad
- Ocupación por edificio
- Reservas y asistencias por tipo de usuario
- Sanciones por tipo de usuario
- Uso de reservas (utilizadas vs canceladas)
- Participantes más activos
- Salas sin uso
- Horarios pico

## Solución de Problemas

### Error de Conexión a BD
- Verificar que MySQL esté ejecutándose
- Confirmar credenciales en el código
- Asegurar que la base de datos existe

### Dependencias Faltantes
pip install --upgrade streamlit pymysql


### Problemas de Permisos
- Ejecutar MySQL con privilegios de root
- Verificar permisos de usuario en la base de datos

## Estructura del Proyecto
gestion-salas-ucu/
│
├── sistema_salas.py # Aplicación principal
├── dbControl_Salas # Script de base de datos
└── README.md # Este archivo

## Desarrollo
- Backend: Python puro sin ORM
- Base de datos: MySQL con constraints nativas
- Frontend: Streamlit para interfaz web
- Validaciones: En frontend, backend y base de datos

## Licencia
Universidad Católica del Uruguay - Base de Datos 1 - 2025

## Autores
Gomez, Tarigo, Von Sanden
