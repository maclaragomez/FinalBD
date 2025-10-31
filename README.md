# Sistema de Gestion de Salas de Estudio - UCU

Base de Datos 1 - Segundo Semestre 2025
Primera Entrega - Avance

# Descripcion

Sistema para gestionar reservas de salas de estudio en la UCU. Esta es la primera entrega del trabajo.

# Estructura de Carpetas

obligatorioBD/
├── sql/
│   └── database_setup.sql
├── backend/
│   └── sistema_salas_completo.py
└── README.md

## Requisitos

- Python 3.x
- MySQL
- PyMySQL (se instala con pip)

# Como usar

1. Primero crear la base de datos:
   mysql -u root -p < sql/database_setup.sql

2. Instalar PyMySQL:
   pip install pymysql

3. Ejecutar el sistema:
   python backend/sistema_salas_completo.py

# Lo que hace el sistema

- Agregar, eliminar y modificar participantes
- Listar participantes y salas
- Ver reportes basicos
- Conexion con base de datos MySQL

# Base de Datos

El sistema usa tablas para:
- participantes (estudiantes y docentes)
- salas de estudio
- facultades y programas
- reservas
- turnos

# Tecnologias

- Python
- MySQL
- PyMySQL para conectar Python con MySQL

# Proximos pasos

En la siguiente entrega se agregara la funcionalidad completa de reservas y front basico.

# Autor
Maria Gomez, Santiago Tarigo, Francisco Von Sanden
