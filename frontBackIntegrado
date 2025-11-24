import streamlit as st
import pymysql
from pymysql import Error
import datetime
import re
import sys
import io
from contextlib import redirect_stdout
import hashlib


class SistemaSalas:
    def __init__(self):
        self.host = 'localhost'
        self.user = 'root'
        self.password = 'root'
        self.database = 'gestion_salas_estudio'
        self.usuario_actual = None
        self.rol_actual = None
        self.nombre_usuario = None

    def get_connection(self):
        try:
            connection = pymysql.connect(
                host=self.host,
                user=self.user,
                password=self.password,
                database=self.database,
                charset='utf8mb4',
                cursorclass=pymysql.cursors.DictCursor
            )
            return connection
        except Error as e:
            st.error(f"Error de conexion: {e}")
            return None

    def hash_password(self, password):
        """Hash básico para contraseñas"""
        return hashlib.sha256(password.encode()).hexdigest()

    def login(self, ci, password):
        """Verificar credenciales y obtener rol del usuario"""
        conn = self.get_connection()
        if not conn:
            return False

        try:
            with conn.cursor() as cursor:
                # Verificar si es administrador
                if ci == "admin":
                    cursor.execute("SELECT * FROM administrador WHERE usuario = 'admin'")
                    admin = cursor.fetchone()
                    if admin and admin['contrasena'] == password:  # Contraseña en texto plano para admin
                        self.usuario_actual = "admin"
                        self.nombre_usuario = admin['nombre']
                        self.rol_actual = 'admin'
                        return True
                    else:
                        st.error("Credenciales de administrador incorrectas")
                        return False

                # Verificar credenciales del participante
                cursor.execute("""
                    SELECT p.ci, p.nombre, p.apellido, ppa.rol, l.contrasena
                    FROM participante p
                    LEFT JOIN participante_programa_academico ppa ON p.ci = ppa.ci_participante
                    LEFT JOIN login l ON p.ci = l.ci_participante
                    WHERE p.ci = %s
                """, (ci,))

                usuario = cursor.fetchone()

                if usuario:
                    # Verificar contraseña
                    if usuario['contrasena'] and usuario['contrasena'] == self.hash_password(password):
                        self.usuario_actual = usuario['ci']
                        self.nombre_usuario = f"{usuario['nombre']} {usuario['apellido']}"
                        self.rol_actual = usuario['rol'] if usuario['rol'] else 'alumno'
                        return True
                    else:
                        st.error("Contraseña incorrecta")
                        return False
                else:
                    st.error("Usuario no encontrado")
                    return False

        except Error as e:
            st.error(f"Error en login: {e}")
            return False
        finally:
            conn.close()

    def cambiar_contrasena(self, ci, nueva_contrasena):
        """Cambiar contraseña de usuario"""
        conn = self.get_connection()
        if not conn:
            return False

        try:
            with conn.cursor() as cursor:
                hashed_password = self.hash_password(nueva_contrasena)
                sql = "UPDATE login SET contrasena = %s WHERE ci_participante = %s"
                cursor.execute(sql, (hashed_password, ci))
                conn.commit()
                st.success("Contraseña actualizada exitosamente!")
                return True

        except Error as e:
            st.error(f"Error al cambiar contraseña: {e}")
            return False
        finally:
            conn.close()

    def logout(self):
        """Cerrar sesión"""
        self.usuario_actual = None
        self.rol_actual = None
        self.nombre_usuario = None

    def validar_email(self, email):
        """Validacion basica de email"""
        if '@' not in email or '.' not in email:
            return False
        if len(email) < 5:
            return False
        return True

    def verificar_permiso(self, roles_permitidos):
        """Verificacion de permisos por rol"""
        if not self.usuario_actual:
            st.warning("Debe iniciar sesion para realizar esta accion")
            return False

        # Admin tiene acceso completo
        if self.rol_actual == 'admin':
            return True

        if self.rol_actual in roles_permitidos:
            return True
        else:
            st.warning(f"No tiene permisos para esta accion. Se requiere: {', '.join(roles_permitidos)}")
            return False

    # ========== PERMISOS ESPECIFICOS POR ROL ==========

    def puede_gestionar_participantes(self):
        """Solo admin y docentes pueden gestionar participantes"""
        return self.verificar_permiso(['admin', 'docente'])

    def puede_gestionar_salas(self):
        """Solo admin puede gestionar salas"""
        return self.verificar_permiso(['admin'])

    def puede_gestionar_reservas(self):
        """Todos los roles pueden gestionar reservas"""
        return self.verificar_permiso(['admin', 'docente', 'alumno'])

    def puede_gestionar_sanciones(self):
        """Solo admin puede gestionar sanciones"""
        return self.verificar_permiso(['admin'])

    def puede_ver_reportes(self):
        """Solo admin puede ver reportes completos"""
        return self.verificar_permiso(['admin'])

    # ========== VALIDACIONES DE REGLAS DE NEGOCIO ==========
    def verificar_limite_horas(self, ci_participante, fecha, id_turno):
        """Verificar limite de 2 horas diarias"""
        conn = self.get_connection()
        if not conn:
            return False

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT COUNT(*) as horas_reservadas
                    FROM reserva r
                    JOIN reserva_participante rp ON r.id_reserva = rp.id_reserva
                    WHERE rp.ci_participante = %s AND r.fecha = %s AND r.estado = 'activa'
                """, (ci_participante, fecha))
                resultado = cursor.fetchone()
                return resultado['horas_reservadas'] < 2
        except Error as e:
            st.error(f"Error: {e}")
            return False
        finally:
            conn.close()

    def verificar_limite_semanal(self, ci_participante, fecha):
        """Verificar limite de 3 reservas activas por semana"""
        conn = self.get_connection()
        if not conn:
            return False

        try:
            with conn.cursor() as cursor:
                # Calcular inicio de semana (lunes)
                cursor.execute("SELECT DATE_SUB(%s, INTERVAL WEEKDAY(%s) DAY) as inicio_semana", (fecha, fecha))
                inicio_semana = cursor.fetchone()['inicio_semana']

                cursor.execute("""
                    SELECT COUNT(*) as reservas_semana
                    FROM reserva r
                    JOIN reserva_participante rp ON r.id_reserva = rp.id_reserva
                    WHERE rp.ci_participante = %s AND r.fecha >= %s AND r.estado = 'activa'
                """, (ci_participante, inicio_semana))
                resultado = cursor.fetchone()
                return resultado['reservas_semana'] < 3
        except Error as e:
            st.error(f"Error: {e}")
            return False
        finally:
            conn.close()

    def verificar_tipo_sala(self, ci_participante, tipo_sala):
        """Verificar si el usuario puede usar este tipo de sala"""
        conn = self.get_connection()
        if not conn:
            return False

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT ppa.rol, pa.tipo 
                    FROM participante_programa_academico ppa
                    JOIN programa_academico pa ON ppa.nombre_programa = pa.nombre_programa
                    WHERE ppa.ci_participante = %s
                """, (ci_participante,))
                usuario = cursor.fetchone()

                if not usuario:
                    return False

                rol = usuario['rol']
                tipo_usuario = usuario['tipo']

                # Reglas de acceso a salas
                if tipo_sala == 'libre':
                    return True
                elif tipo_sala == 'posgrado':
                    return rol == 'docente' or tipo_usuario == 'posgrado'
                elif tipo_sala == 'docente':
                    return rol == 'docente'
                else:
                    return False

        except Error as e:
            st.error(f"Error: {e}")
            return False
        finally:
            conn.close()

    def verificar_sanciones(self, ci_participante):
        """Verificar si el usuario tiene sanciones activas"""
        conn = self.get_connection()
        if not conn:
            return False

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT * FROM sancion_participante 
                    WHERE ci_participante = %s AND fecha_fin >= CURDATE()
                """, (ci_participante,))
                sanciones = cursor.fetchall()
                return len(sanciones) == 0
        except Error as e:
            st.error(f"Error: {e}")
            return False
        finally:
            conn.close()

    def obtener_programas_academicos(self):
        """Obtener lista de programas académicos disponibles"""
        conn = self.get_connection()
        if not conn:
            return []

        try:
            with conn.cursor() as cursor:
                cursor.execute("SELECT nombre_programa FROM programa_academico ORDER BY nombre_programa")
                programas = cursor.fetchall()
                return [programa['nombre_programa'] for programa in programas]
        except Error as e:
            st.error(f"Error: {e}")
            return []
        finally:
            conn.close()

    def obtener_salas_disponibles(self, fecha, id_turno):
        """Obtener salas disponibles para una fecha y turno específicos"""
        conn = self.get_connection()
        if not conn:
            return []

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT s.nombre_sala, s.edificio, s.capacidad, s.tipo_sala
                    FROM sala s
                    WHERE s.nombre_sala NOT IN (
                        SELECT r.nombre_sala 
                        FROM reserva r 
                        WHERE r.fecha = %s AND r.id_turno = %s AND r.estado = 'activa'
                    )
                    ORDER BY s.edificio, s.nombre_sala
                """, (fecha, id_turno))
                return cursor.fetchall()
        except Error as e:
            st.error(f"Error: {e}")
            return []
        finally:
            conn.close()

    def obtener_turnos_disponibles(self, fecha, nombre_sala):
        """Obtener turnos disponibles para una sala y fecha específicas"""
        conn = self.get_connection()
        if not conn:
            return []

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT t.id_turno, t.hora_inicio, t.hora_fin
                    FROM turno t
                    WHERE t.id_turno NOT IN (
                        SELECT r.id_turno 
                        FROM reserva r 
                        WHERE r.nombre_sala = %s AND r.fecha = %s AND r.estado = 'activa'
                    )
                    ORDER BY t.hora_inicio
                """, (nombre_sala, fecha))
                return cursor.fetchall()
        except Error as e:
            st.error(f"Error: {e}")
            return []
        finally:
            conn.close()
    # ========== ABM PARTICIPANTES ==========
    def alta_participante(self, ci, nombre, apellido, email, programa_academico, rol='alumno', password=None):
        """Agregar nuevo participante con programa académico, rol y contraseña"""
        if not self.puede_gestionar_participantes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                # Verificar si ya existe el participante
                cursor.execute("SELECT ci FROM participante WHERE ci = %s OR email = %s", (ci, email))
                if cursor.fetchone():
                    st.error("Error: Ya existe un participante con esa cedula o email")
                    return

                # Verificar que el programa académico existe
                cursor.execute("SELECT nombre_programa FROM programa_academico WHERE nombre_programa = %s",
                               (programa_academico,))
                if not cursor.fetchone():
                    st.error("Error: El programa académico seleccionado no existe")
                    return

                # Insertar participante
                sql_participante = "INSERT INTO participante (ci, nombre, apellido, email) VALUES (%s, %s, %s, %s)"
                cursor.execute(sql_participante, (ci, nombre, apellido, email))

                # Insertar en login con contraseña por defecto si no se proporciona
                if not password:
                    password = ci  # Por defecto, la cédula como contraseña

                hashed_password = self.hash_password(password)
                sql_login = "INSERT INTO login (correo, contrasena, ci_participante) VALUES (%s, %s, %s)"
                cursor.execute(sql_login, (email, hashed_password, ci))

                # Asignar programa académico con el rol elegido
                sql_programa = """
                INSERT INTO participante_programa_academico (ci_participante, nombre_programa, rol) 
                VALUES (%s, %s, %s)
                """
                cursor.execute(sql_programa, (ci, programa_academico, rol))

                conn.commit()
                st.success("Participante agregado exitosamente!")
                st.info(f"Contraseña inicial: {password}")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def baja_participante(self, ci):
        """Eliminar participante"""
        if not self.puede_gestionar_participantes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("SELECT * FROM participante WHERE ci = %s", (ci,))
                if not cursor.fetchone():
                    st.error("No se encontro participante con esa cedula")
                    return

                sql = "DELETE FROM participante WHERE ci = %s"
                cursor.execute(sql, (ci,))
                conn.commit()
                st.success("Participante eliminado exitosamente!")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def modificacion_participante(self, ci, nuevo_nombre, nuevo_apellido, nuevo_email):
        """Modificar datos de participante"""
        if not self.puede_gestionar_participantes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("SELECT * FROM participante WHERE ci = %s", (ci,))
                participante = cursor.fetchone()

                if not participante:
                    st.error("No se encontro participante con esa cedula")
                    return

                if nuevo_email != participante['email'] and not self.validar_email(nuevo_email):
                    st.error("Error: El email debe contener @ y . (ej: usuario@ucu.edu.uy)")
                    return

                sql = """UPDATE participante 
                         SET nombre = %s, apellido = %s, email = %s 
                         WHERE ci = %s"""
                cursor.execute(sql, (nuevo_nombre, nuevo_apellido, nuevo_email, ci))
                conn.commit()
                st.success("Participante actualizado exitosamente!")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    # ========== ABM SALAS ==========
    def alta_sala(self, nombre_sala, edificio, capacidad, tipo_sala):
        """Agregar nueva sala"""
        if not self.puede_gestionar_salas():
            return

        # VALIDAR CAPACIDAD
        if capacidad <= 0:
            st.error("Error: La capacidad debe ser mayor a 0")
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                sql = "INSERT INTO sala (nombre_sala, edificio, capacidad, tipo_sala) VALUES (%s, %s, %s, %s)"
                cursor.execute(sql, (nombre_sala, edificio, capacidad, tipo_sala))
                conn.commit()
                st.success("Sala agregada exitosamente!")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def baja_sala(self, nombre_sala):
        """Eliminar sala"""
        if not self.puede_gestionar_salas():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                sql = "DELETE FROM sala WHERE nombre_sala = %s"
                cursor.execute(sql, (nombre_sala,))
                conn.commit()
                st.success("Sala eliminada exitosamente!")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    # ========== RESERVAS CON VALIDACIONES COMPLETAS ==========
    def hacer_reserva(self, nombre_sala, fecha, id_turno, ci_participante):
        """Hacer una nueva reserva con todas las validaciones"""
        if not self.puede_gestionar_reservas():
            return

        # Si fecha es date object, convertir a string
        if not isinstance(fecha, str):
            fecha_str = fecha.strftime('%Y-%m-%d')
        else:
            fecha_str = fecha

        # Validar que la fecha no sea en el pasado
        hoy = datetime.datetime.now().date()
        if isinstance(fecha, str):
            fecha_date = datetime.datetime.strptime(fecha_str, '%Y-%m-%d').date()
        else:
            fecha_date = fecha

        if fecha_date < hoy:
            st.error("Error: No se pueden hacer reservas en fechas pasadas")
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                # 1. Verificar disponibilidad de la sala
                cursor.execute("""
                    SELECT * FROM reserva 
                    WHERE nombre_sala = %s AND fecha = %s AND id_turno = %s AND estado = 'activa'
                """, (nombre_sala, fecha_str, id_turno))

                if cursor.fetchone():
                    st.error("La sala ya esta reservada en ese horario")
                    return

                # 2. Obtener informacion de la sala
                cursor.execute("SELECT capacidad, tipo_sala, edificio FROM sala WHERE nombre_sala = %s", (nombre_sala,))
                sala_info = cursor.fetchone()
                if not sala_info:
                    st.error("Sala no encontrada")
                    return

                # 3. Verificar tipo de sala vs usuario
                if not self.verificar_tipo_sala(ci_participante, sala_info['tipo_sala']):
                    st.error("Este usuario no tiene permiso para reservar este tipo de sala")
                    return

                # 4. Verificar sanciones activas
                if not self.verificar_sanciones(ci_participante):
                    st.error("El usuario tiene sanciones activas y no puede hacer reservas")
                    return

                # 5. Verificar limites (solo para alumnos de grado en salas libres)
                cursor.execute("""
                    SELECT ppa.rol, pa.tipo 
                    FROM participante_programa_academico ppa
                    JOIN programa_academico pa ON ppa.nombre_programa = pa.nombre_programa
                    WHERE ppa.ci_participante = %s
                """, (ci_participante,))
                usuario_info = cursor.fetchone()

                if usuario_info and usuario_info['rol'] == 'alumno' and usuario_info['tipo'] == 'grado' and sala_info[
                    'tipo_sala'] == 'libre':
                    if not self.verificar_limite_horas(ci_participante, fecha_str, id_turno):
                        st.error("El usuario ya tiene 2 horas reservadas hoy")
                        return

                    if not self.verificar_limite_semanal(ci_participante, fecha_str):
                        st.error("El usuario ya tiene 3 reservas activas esta semana")
                        return

                # 6. Obtener edificio de la sala
                edificio_info = sala_info['edificio']

                # Crear reserva
                sql = """
                INSERT INTO reserva (nombre_sala, edificio, fecha, id_turno, estado) 
                VALUES (%s, %s, %s, %s, 'activa')
                """
                cursor.execute(sql, (nombre_sala, edificio_info, fecha_str, id_turno))
                id_reserva = cursor.lastrowid

                # Agregar participante a la reserva (SOLO EL QUE HACE LA RESERVA)
                sql_participante = """
                INSERT INTO reserva_participante (ci_participante, id_reserva) 
                VALUES (%s, %s)
                """
                cursor.execute(sql_participante, (ci_participante, id_reserva))

                conn.commit()
                st.success("Reserva creada exitosamente!")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def cancelar_reserva(self, id_reserva):
        """Cancelar una reserva activa"""
        if not self.puede_gestionar_reservas():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                sql = "UPDATE reserva SET estado = 'cancelada' WHERE id_reserva = %s"
                cursor.execute(sql, (id_reserva,))
                conn.commit()
                st.success("Reserva cancelada exitosamente!")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    # ========== NUEVAS FUNCIONALIDADES ==========
    def registrar_asistencia(self, id_reserva, ci_participante):
        """Registrar asistencia a una reserva (solo para el que hizo la reserva)"""
        if not self.puede_gestionar_reservas():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                # Verificar que la reserva existe y es del día actual
                cursor.execute("""
                    SELECT r.*, rp.ci_participante 
                    FROM reserva r
                    JOIN reserva_participante rp ON r.id_reserva = rp.id_reserva
                    WHERE r.id_reserva = %s AND rp.ci_participante = %s 
                    AND r.estado = 'activa'
                """, (id_reserva, ci_participante))

                reserva = cursor.fetchone()

                if not reserva:
                    st.error("Reserva no encontrada o no tiene permisos")
                    return

                # Verificar que la fecha de la reserva es hoy
                hoy = datetime.datetime.now().date()
                if reserva['fecha'] != hoy:
                    st.error("Solo se puede registrar asistencia para reservas del día de hoy")
                    return

                # Actualizar asistencia
                sql = """
                UPDATE reserva_participante 
                SET asistencia = TRUE 
                WHERE id_reserva = %s AND ci_participante = %s
                """
                cursor.execute(sql, (id_reserva, ci_participante))

                # Cambiar estado de la reserva a finalizada
                cursor.execute("UPDATE reserva SET estado = 'finalizada' WHERE id_reserva = %s", (id_reserva,))

                conn.commit()
                st.success("Asistencia registrada exitosamente!")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def verificar_reservas_sin_asistencia(self):
        """Verificar reservas sin asistencia y aplicar sanciones automáticamente"""
        if not self.puede_gestionar_sanciones():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                # Buscar reservas pasadas sin asistencia registrada
                cursor.execute("""
                    SELECT DISTINCT r.id_reserva, r.fecha, rp.ci_participante
                    FROM reserva r
                    JOIN reserva_participante rp ON r.id_reserva = rp.id_reserva
                    WHERE r.fecha < CURDATE() 
                    AND r.estado = 'activa'
                    AND (rp.asistencia IS NULL OR rp.asistencia = FALSE)
                """)

                reservas_sin_asistencia = cursor.fetchall()
                sanciones_aplicadas = 0

                for reserva in reservas_sin_asistencia:
                    # Verificar si ya tiene sanción activa
                    cursor.execute("""
                        SELECT * FROM sancion_participante 
                        WHERE ci_participante = %s AND fecha_fin >= CURDATE()
                    """, (reserva['ci_participante'],))

                    if not cursor.fetchone():
                        # Aplicar sanción de 2 meses
                        fecha_inicio = datetime.datetime.now().date()
                        fecha_fin = fecha_inicio + datetime.timedelta(days=60)

                        cursor.execute("""
                            INSERT INTO sancion_participante (ci_participante, fecha_inicio, fecha_fin) 
                            VALUES (%s, %s, %s)
                        """, (reserva['ci_participante'], fecha_inicio, fecha_fin))
                        sanciones_aplicadas += 1

                    # Cambiar estado de la reserva
                    cursor.execute("UPDATE reserva SET estado = 'sin_asistencia' WHERE id_reserva = %s",
                                   (reserva['id_reserva'],))

                conn.commit()

                if sanciones_aplicadas > 0:
                    st.info(f"Se aplicaron {sanciones_aplicadas} sanciones por reservas sin asistencia")
                else:
                    st.info("No hay reservas pendientes de sanción")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def listar_mis_reservas(self):
        """Listar reservas del usuario actual"""
        if not self.usuario_actual or self.rol_actual == 'admin':
            st.info("Esta función es para usuarios regulares")
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT r.id_reserva, r.nombre_sala, r.edificio, r.fecha, 
                           t.hora_inicio, t.hora_fin, r.estado,
                           (SELECT COUNT(*) FROM reserva_participante rp2 WHERE rp2.id_reserva = r.id_reserva) as total_participantes,
                           rp.asistencia
                    FROM reserva r
                    JOIN turno t ON r.id_turno = t.id_turno
                    JOIN reserva_participante rp ON r.id_reserva = rp.id_reserva
                    WHERE rp.ci_participante = %s
                    ORDER BY r.fecha DESC, t.hora_inicio
                """, (self.usuario_actual,))

                reservas = cursor.fetchall()

                st.subheader("MIS RESERVAS")
                if not reservas:
                    st.info("No tienes reservas activas")
                    return

                for reserva in reservas:
                    estado_color = "🟢" if reserva['estado'] == 'activa' else "🔴" if reserva[
                                                                                        'estado'] == 'cancelada' else "🟡"
                    asistencia_text = "✅ Asistió" if reserva['asistencia'] else "❌ No asistió" if reserva[
                                                                                                      'asistencia'] is False else "⏳ Pendiente"

                    st.write(f"{estado_color} **Reserva #{reserva['id_reserva']}**")
                    st.write(f"**Sala:** {reserva['nombre_sala']} ({reserva['edificio']})")
                    st.write(f"**Fecha:** {reserva['fecha']} {reserva['hora_inicio']}-{reserva['hora_fin']}")
                    st.write(f"**Estado:** {reserva['estado']} | **Asistencia:** {asistencia_text}")
                    st.write(f"**Total participantes:** {reserva['total_participantes']}")
                    st.write("---")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    # ========== SANCIONES ==========
    def aplicar_sancion(self, ci_participante, fecha_inicio):
        """Aplicar sancion a un participante"""
        if not self.puede_gestionar_sanciones():
            return

        # VALIDAR FECHA (acepta tanto string como date object)
        if isinstance(fecha_inicio, str):
            try:
                fecha_inicio_obj = datetime.datetime.strptime(fecha_inicio, '%Y-%m-%d').date()
            except ValueError:
                st.error("Error: Formato de fecha invalido")
                return
        else:
            fecha_inicio_obj = fecha_inicio

        hoy = datetime.datetime.now().date()

        if fecha_inicio_obj < hoy:
            st.error("Error: La fecha de inicio no puede ser en el pasado")
            return

        # Calcular fecha fin (2 meses despues)
        fecha_fin_obj = fecha_inicio_obj + datetime.timedelta(days=60)
        fecha_fin = fecha_fin_obj.strftime('%Y-%m-%d')
        fecha_inicio_str = fecha_inicio_obj.strftime('%Y-%m-%d')

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                sql = "INSERT INTO sancion_participante (ci_participante, fecha_inicio, fecha_fin) VALUES (%s, %s, %s)"
                cursor.execute(sql, (ci_participante, fecha_inicio_str, fecha_fin))
                conn.commit()
                st.success("Sancion aplicada exitosamente!")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    # ========== CONSULTAS Y REPORTES COMPLETOS ==========
    def listar_participantes(self):
        """Listar todos los participantes"""
        if not self.puede_gestionar_participantes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT p.ci, p.nombre, p.apellido, p.email,
                           GROUP_CONCAT(CONCAT(ppa.nombre_programa, ' (', ppa.rol, ')') SEPARATOR ', ') as programas
                    FROM participante p
                    LEFT JOIN participante_programa_academico ppa ON p.ci = ppa.ci_participante
                    GROUP BY p.ci
                    ORDER BY p.apellido, p.nombre
                """)
                participantes = cursor.fetchall()

                st.subheader("LISTA DE PARTICIPANTES")
                for part in participantes:
                    st.write(f"**Cedula:** {part['ci']}")
                    st.write(f"**Nombre:** {part['nombre']} {part['apellido']}")
                    st.write(f"**Email:** {part['email']}")
                    st.write(f"**Programas:** {part['programas'] or 'Sin programa asignado'}")
                    st.write("---")

                st.write(f"**Total:** {len(participantes)} participantes")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def listar_salas(self):
        """Listar todas las salas disponibles"""
        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT s.nombre_sala, s.edificio, s.capacidad, s.tipo_sala, e.direccion
                    FROM sala s
                    JOIN edificio e ON s.edificio = e.nombre_edificio
                    ORDER BY s.edificio, s.nombre_sala
                """)
                salas = cursor.fetchall()

                st.subheader("SALAS DISPONIBLES")
                for sala in salas:
                    st.write(f"**Sala:** {sala['nombre_sala']}")
                    st.write(f"**Edificio:** {sala['edificio']}")
                    st.write(f"**Capacidad:** {sala['capacidad']} personas")
                    st.write(f"**Tipo:** {sala['tipo_sala']}")
                    st.write(f"**Direccion:** {sala['direccion']}")
                    st.write("---")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def listar_turnos(self):
        """Listar todos los turnos disponibles"""
        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("SELECT * FROM turno ORDER BY hora_inicio")
                turnos = cursor.fetchall()

                st.subheader("TURNOS DISPONIBLES")
                for turno in turnos:
                    st.write(f"**ID:** {turno['id_turno']} - {turno['hora_inicio']} a {turno['hora_fin']}")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    # ========== REPORTES COMPLETOS ==========
    def reporte_salas_populares(self):
        """Salas mas reservadas"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT r.nombre_sala, r.edificio, COUNT(*) as total_reservas
                    FROM reserva r
                    WHERE r.estado = 'activa'
                    GROUP BY r.nombre_sala, r.edificio
                    ORDER BY total_reservas DESC
                    LIMIT 10
                """)
                salas = cursor.fetchall()

                st.subheader("SALAS MAS RESERVADAS")
                for i, sala in enumerate(salas, 1):
                    st.write(f"{i}. {sala['nombre_sala']} ({sala['edificio']}): {sala['total_reservas']} reservas")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_turnos_demandados(self):
        """Turnos mas demandados"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT t.id_turno, t.hora_inicio, t.hora_fin, COUNT(r.id_reserva) as total_reservas
                    FROM turno t
                    LEFT JOIN reserva r ON t.id_turno = r.id_turno
                    GROUP BY t.id_turno, t.hora_inicio, t.hora_fin
                    ORDER BY total_reservas DESC
                """)
                turnos = cursor.fetchall()

                st.subheader("TURNOS MAS DEMANDADOS")
                for i, turno in enumerate(turnos, 1):
                    st.write(f"{i}. {turno['hora_inicio']} - {turno['hora_fin']}: {turno['total_reservas']} reservas")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_promedio_participantes(self):
        """Promedio de participantes por sala"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT s.nombre_sala, s.edificio, 
                           COALESCE(AVG(participantes_count), 0) as promedio_participantes
                    FROM sala s
                    LEFT JOIN (
                        SELECT r.id_reserva, r.nombre_sala, r.edificio, COUNT(rp.ci_participante) as participantes_count
                        FROM reserva r
                        JOIN reserva_participante rp ON r.id_reserva = rp.id_reserva
                        GROUP BY r.id_reserva, r.nombre_sala, r.edificio
                    ) AS reservas ON s.nombre_sala = reservas.nombre_sala AND s.edificio = reservas.edificio
                    GROUP BY s.nombre_sala, s.edificio
                """)
                salas = cursor.fetchall()

                st.subheader("PROMEDIO DE PARTICIPANTES POR SALA")
                for sala in salas:
                    st.write(
                        f"{sala['nombre_sala']} ({sala['edificio']}): {sala['promedio_participantes']:.1f} participantes")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_reservas_carrera_facultad(self):
        """Cantidad de reservas por carrera y facultad"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT pa.nombre_programa, f.nombre as facultad, COUNT(rp.id_reserva) as total_reservas
                    FROM programa_academico pa
                    JOIN facultad f ON pa.id_facultad = f.id_facultad
                    JOIN participante_programa_academico ppa ON pa.nombre_programa = ppa.nombre_programa
                    JOIN reserva_participante rp ON ppa.ci_participante = rp.ci_participante
                    GROUP BY pa.nombre_programa, f.nombre
                    ORDER BY total_reservas DESC
                """)
                resultados = cursor.fetchall()

                st.subheader("RESERVAS POR CARRERA Y FACULTAD")
                for res in resultados:
                    st.write(f"{res['nombre_programa']} ({res['facultad']}): {res['total_reservas']} reservas")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_ocupacion_edificio(self):
        """Porcentaje de ocupacion de salas por edificio"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT e.nombre_edificio,
                           COUNT(r.id_reserva) as reservas_totales,
                           (SELECT COUNT(*) FROM turno) * (SELECT COUNT(*) FROM sala WHERE sala.edificio = e.nombre_edificio) as turnos_posibles,
                           ROUND((COUNT(r.id_reserva) / ((SELECT COUNT(*) FROM turno) * (SELECT COUNT(*) FROM sala WHERE sala.edificio = e.nombre_edificio))) * 100, 2) as porcentaje_ocupacion
                    FROM edificio e
                    LEFT JOIN sala s ON e.nombre_edificio = s.edificio
                    LEFT JOIN reserva r ON s.nombre_sala = r.nombre_sala
                    GROUP BY e.nombre_edificio
                """)
                edificios = cursor.fetchall()

                st.subheader("OCUPACION POR EDIFICIO")
                for edificio in edificios:
                    st.write(f"**{edificio['nombre_edificio']}:**")
                    st.write(f"  Reservas totales: {edificio['reservas_totales']}")
                    st.write(f"  Turnos posibles: {edificio['turnos_posibles']}")
                    st.write(f"  Porcentaje de ocupacion: {edificio['porcentaje_ocupacion']}%")
                    st.write("")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_reservas_asistencias(self):
        """Cantidad de reservas y asistencias por tipo de usuario"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT 
                        ppa.rol,
                        pa.tipo,
                        COUNT(rp.id_reserva) as total_reservas,
                        SUM(CASE WHEN rp.asistencia = TRUE THEN 1 ELSE 0 END) as asistencias
                    FROM participante_programa_academico ppa
                    JOIN programa_academico pa ON ppa.nombre_programa = pa.nombre_programa
                    JOIN reserva_participante rp ON ppa.ci_participante = rp.ci_participante
                    GROUP BY ppa.rol, pa.tipo
                """)
                resultados = cursor.fetchall()

                st.subheader("RESERVAS Y ASISTENCIAS POR TIPO DE USUARIO")
                for res in resultados:
                    st.write(
                        f"{res['rol']} ({res['tipo']}): {res['total_reservas']} reservas, {res['asistencias']} asistencias")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_sanciones_tipo(self):
        """Cantidad de sanciones por tipo de usuario"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT 
                        ppa.rol,
                        pa.tipo,
                        COUNT(sp.ci_participante) as total_sanciones
                    FROM participante_programa_academico ppa
                    JOIN programa_academico pa ON ppa.nombre_programa = pa.nombre_programa
                    JOIN sancion_participante sp ON ppa.ci_participante = sp.ci_participante
                    GROUP BY ppa.rol, pa.tipo
                """)
                resultados = cursor.fetchall()

                st.subheader("SANCIONES POR TIPO DE USUARIO")
                for res in resultados:
                    st.write(f"{res['rol']} ({res['tipo']}): {res['total_sanciones']} sanciones")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_uso_reservas(self):
        """Porcentaje de reservas utilizadas vs canceladas/no asistidas"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT 
                        estado,
                        COUNT(*) as cantidad,
                        ROUND((COUNT(*) / (SELECT COUNT(*) FROM reserva)) * 100, 2) as porcentaje
                    FROM reserva
                    GROUP BY estado
                """)
                estados = cursor.fetchall()

                st.subheader("USO DE RESERVAS")
                for estado in estados:
                    st.write(f"{estado['estado']}: {estado['cantidad']} reservas ({estado['porcentaje']}%)")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    # ========== TRES CONSULTAS ADICIONALES ==========
    def reporte_participantes_activos(self):
        """Participantes con mas reservas activas"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT p.ci, p.nombre, p.apellido, COUNT(rp.id_reserva) as reservas_activas
                    FROM participante p
                    JOIN reserva_participante rp ON p.ci = rp.ci_participante
                    JOIN reserva r ON rp.id_reserva = r.id_reserva
                    WHERE r.estado = 'activa'
                    GROUP BY p.ci, p.nombre, p.apellido
                    ORDER BY reservas_activas DESC
                    LIMIT 10
                """)
                participantes = cursor.fetchall()

                st.subheader("PARTICIPANTES MAS ACTIVOS")
                for i, part in enumerate(participantes, 1):
                    st.write(
                        f"{i}. {part['nombre']} {part['apellido']} ({part['ci']}): {part['reservas_activas']} reservas activas")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_salas_sin_uso(self):
        """Salas que nunca han sido reservadas"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT s.nombre_sala, s.edificio, s.tipo_sala
                    FROM sala s
                    LEFT JOIN reserva r ON s.nombre_sala = r.nombre_sala
                    WHERE r.id_reserva IS NULL
                    ORDER BY s.edificio, s.nombre_sala
                """)
                salas = cursor.fetchall()

                st.subheader("SALAS SIN USO")
                for sala in salas:
                    st.write(f"{sala['nombre_sala']} ({sala['edificio']}) - Tipo: {sala['tipo_sala']}")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def reporte_horarios_pico(self):
        """Horarios con mas reservas"""
        if not self.puede_ver_reportes():
            return

        conn = self.get_connection()
        if not conn:
            return

        try:
            with conn.cursor() as cursor:
                cursor.execute("""
                    SELECT t.hora_inicio, t.hora_fin, DAYNAME(r.fecha) as dia, COUNT(*) as total_reservas
                    FROM reserva r
                    JOIN turno t ON r.id_turno = t.id_turno
                    GROUP BY t.hora_inicio, t.hora_fin, DAYNAME(r.fecha)
                    ORDER BY total_reservas DESC
                    LIMIT 10
                """)
                horarios = cursor.fetchall()

                st.subheader("HORARIOS PICO")
                for i, horario in enumerate(horarios, 1):
                    st.write(
                        f"{i}. {horario['dia']} {horario['hora_inicio']}-{horario['hora_fin']}: {horario['total_reservas']} reservas")

        except Error as e:
            st.error(f"Error: {e}")
        finally:
            conn.close()

    def probar_conexion(self):
        """Probar conexion a la base de datos"""
        conn = self.get_connection()
        if conn:
            try:
                with conn.cursor() as cursor:
                    cursor.execute("SHOW TABLES")
                    tablas = cursor.fetchall()
                    return True, f"Conexion exitosa! Tablas encontradas: {len(tablas)}"
            except Error as e:
                return False, f"Error: {e}"
            finally:
                conn.close()
        return False, "No se pudo conectar a la base de datos"


# INTERFAZ STREAMLIT
def main_streamlit():
    st.set_page_config(page_title="Sistema Salas UCU", layout="wide")

    st.title("SISTEMA DE GESTION DE SALAS - UCU")

    # Inicializar sistema
    if 'sistema' not in st.session_state:
        st.session_state.sistema = SistemaSalas()

    sistema = st.session_state.sistema

    # Sidebar para login y configuracion
    with st.sidebar:
        st.header("Autenticacion")

        # Si no hay usuario logueado, mostrar formulario de login
        if not sistema.usuario_actual:
            with st.form("login_form"):
                ci = st.text_input("Cedula de Identidad (o 'admin' para administrador)")
                password = st.text_input("Contraseña", type="password")

                if st.form_submit_button("Ingresar al Sistema"):
                    if ci and password:
                        if sistema.login(ci, password):
                            st.success(f"Bienvenido/a {sistema.nombre_usuario}!")
                            st.rerun()
                        # else: los errores se muestran en el método login
                    else:
                        st.error("Por favor complete todos los campos")
        else:
            # Usuario logueado - mostrar informacion y opciones
            st.success(f"Conectado como: {sistema.nombre_usuario}")
            st.info(f"Rol: {sistema.rol_actual}")
            if sistema.rol_actual != 'admin':
                st.info(f"CI: {sistema.usuario_actual}")

            # Opción para cambiar contraseña (solo para usuarios normales)
            if sistema.rol_actual != 'admin':
                with st.expander("Cambiar Contraseña"):
                    with st.form("cambiar_contrasena"):
                        nueva_contrasena = st.text_input("Nueva Contraseña", type="password")
                        confirmar_contrasena = st.text_input("Confirmar Contraseña", type="password")

                        if st.form_submit_button("Cambiar Contraseña"):
                            if nueva_contrasena == confirmar_contrasena:
                                if sistema.cambiar_contrasena(sistema.usuario_actual, nueva_contrasena):
                                    st.success("Contraseña cambiada exitosamente")
                            else:
                                st.error("Las contraseñas no coinciden")

            if st.button("Cerrar Sesion"):
                sistema.logout()
                st.rerun()

        st.header("Configuracion")
        if st.button("Probar Conexion BD"):
            resultado, mensaje = sistema.probar_conexion()
            if resultado:
                st.success(mensaje)
            else:
                st.error(mensaje)

    # Verificar si el usuario esta logueado antes de mostrar el menu principal
    if not sistema.usuario_actual:
        st.warning("🔐 Por favor, inicie sesión en el panel lateral para acceder al sistema")
        st.info("💡 Si no tiene credenciales de acceso, contacte al administrador del sistema")
        return

    # Menu principal (solo visible si esta logueado)
    # Para admin mostrar todas las opciones
    if sistema.rol_actual == 'admin':
        menu_options = [
            "Gestion de Participantes",
            "Gestion de Salas",
            "Gestion de Reservas",
            "Gestion de Sanciones",
            "Reportes y Consultas"
        ]
    # Para docentes mostrar opciones limitadas
    elif sistema.rol_actual == 'docente':
        menu_options = [
            "Gestion de Participantes",
            "Gestion de Reservas"
        ]
    # Para alumnos solo reservas
    else:  # alumno
        menu_options = ["Gestion de Reservas"]

    menu = st.selectbox("Menu Principal", menu_options)

    # Mostrar informacion del usuario actual
    st.sidebar.write("---")
    st.sidebar.write(f"**Usuario actual:** {sistema.nombre_usuario}")
    st.sidebar.write(f"**Rol:** {sistema.rol_actual}")

    # GESTION DE PARTICIPANTES
    if menu == "Gestion de Participantes":
        st.header("Gestion de Participantes")

        opcion = st.selectbox("Operacion", [
            "Alta de participante",
            "Baja de participante",
            "Modificacion de participante",
            "Listar participantes"
        ])

        if opcion == "Alta de participante":
            with st.form("alta_participante"):
                st.subheader("Alta de Participante")
                ci = st.text_input("Cédula")
                nombre = st.text_input("Nombre")
                apellido = st.text_input("Apellido")
                email = st.text_input("Email")

                # Obtener programas académicos disponibles
                programas = sistema.obtener_programas_academicos()
                programa_academico = st.selectbox("Programa Académico", programas)

                rol = st.selectbox("Rol", ["alumno", "docente"])
                password = st.text_input("Contraseña (opcional - por defecto será la cédula)", type="password")

                if st.form_submit_button("Agregar Participante"):
                    sistema.alta_participante(ci, nombre, apellido, email, programa_academico, rol, password)

        elif opcion == "Baja de participante":
            with st.form("baja_participante"):
                st.subheader("Baja de Participante")
                ci = st.text_input("Cedula del participante a eliminar")

                if st.form_submit_button("Eliminar Participante"):
                    sistema.baja_participante(ci)

        elif opcion == "Modificacion de participante":
            with st.form("modificacion_participante"):
                st.subheader("Modificacion de Participante")
                ci = st.text_input("Cedula del participante a modificar")
                nuevo_nombre = st.text_input("Nuevo nombre")
                nuevo_apellido = st.text_input("Nuevo apellido")
                nuevo_email = st.text_input("Nuevo email")

                if st.form_submit_button("Actualizar Participante"):
                    sistema.modificacion_participante(ci, nuevo_nombre, nuevo_apellido, nuevo_email)

        elif opcion == "Listar participantes":
            if st.button("Mostrar Participantes"):
                sistema.listar_participantes()

    # GESTION DE SALAS
    elif menu == "Gestion de Salas":
        st.header("Gestion de Salas")

        opcion = st.selectbox("Operacion", [
            "Alta de sala",
            "Baja de sala",
            "Listar salas"
        ])

        if opcion == "Alta de sala":
            with st.form("alta_sala"):
                st.subheader("Alta de Sala")
                nombre_sala = st.text_input("Nombre de la sala")
                edificio = st.text_input("Edificio")
                capacidad = st.number_input("Capacidad", min_value=1, value=10)
                tipo_sala = st.selectbox("Tipo de sala", ["libre", "posgrado", "docente"])

                if st.form_submit_button("Agregar Sala"):
                    sistema.alta_sala(nombre_sala, edificio, capacidad, tipo_sala)

        elif opcion == "Baja de sala":
            with st.form("baja_sala"):
                st.subheader("Baja de Sala")
                nombre_sala = st.text_input("Nombre de la sala a eliminar")

                if st.form_submit_button("Eliminar Sala"):
                    sistema.baja_sala(nombre_sala)

        elif opcion == "Listar salas":
            if st.button("Mostrar Salas"):
                sistema.listar_salas()

    # GESTION DE RESERVAS
    elif menu == "Gestion de Reservas":
        st.header("Gestion de Reservas")

        opcion_reservas = st.selectbox("Operaciones de Reservas", [
            "Nueva reserva",
            "Cancelar reserva",
            "Registrar asistencia",
            "Ver mis reservas",
            "Ver salas disponibles",
            "Ver turnos disponibles"
        ])

        if opcion_reservas == "Nueva reserva":
            with st.form("nueva_reserva"):
                st.subheader("Nueva Reserva")

                # Fecha primero
                fecha = st.date_input("Fecha", min_value=datetime.datetime.now().date())

                # Mostrar todas las salas inicialmente
                st.write("**Seleccione una sala:**")
                sistema.listar_salas()

                nombre_sala = st.text_input("Nombre de la sala que desea reservar")

                # Si se ingresó una sala, mostrar turnos disponibles
                if nombre_sala and fecha:
                    turnos_disponibles = sistema.obtener_turnos_disponibles(fecha, nombre_sala)

                    if turnos_disponibles:
                        st.write("**Turnos disponibles para esta sala y fecha:**")
                        for turno in turnos_disponibles:
                            st.write(f"ID: {turno['id_turno']} - {turno['hora_inicio']} a {turno['hora_fin']}")

                        id_turno = st.number_input("ID del turno deseado", min_value=1, value=1)
                    else:
                        st.warning("No hay turnos disponibles para esta sala en la fecha seleccionada")
                        id_turno = None
                else:
                    id_turno = st.number_input("ID del turno", min_value=1, value=1)

                # Para alumnos y docentes, usar su propia cedula automaticamente
                if sistema.rol_actual in ['alumno', 'docente']:
                    ci_participante = sistema.usuario_actual
                    st.info(f"Reservando para: {ci_participante}")
                else:
                    ci_participante = st.text_input("CI del participante")

                if st.form_submit_button("Hacer Reserva"):
                    if nombre_sala and fecha and id_turno and ci_participante:
                        sistema.hacer_reserva(nombre_sala, fecha, id_turno, ci_participante)
                    else:
                        st.error("Por favor complete todos los campos")

        elif opcion_reservas == "Cancelar reserva":
            with st.form("cancelar_reserva"):
                st.subheader("Cancelar Reserva")
                id_reserva = st.number_input("ID de la reserva a cancelar", min_value=1)

                if st.form_submit_button("Cancelar Reserva"):
                    sistema.cancelar_reserva(id_reserva)

        elif opcion_reservas == "Registrar asistencia":
            with st.form("registrar_asistencia"):
                st.subheader("Registrar Asistencia")
                id_reserva = st.number_input("ID de la reserva", min_value=1)

                if sistema.rol_actual in ['alumno', 'docente']:
                    ci_participante = sistema.usuario_actual
                    st.info(f"Registrando asistencia para: {ci_participante}")
                else:
                    ci_participante = st.text_input("Cédula del participante")

                if st.form_submit_button("Registrar Asistencia"):
                    sistema.registrar_asistencia(id_reserva, ci_participante)

        elif opcion_reservas == "Ver mis reservas":
            sistema.listar_mis_reservas()

        elif opcion_reservas == "Ver salas disponibles":
            sistema.listar_salas()

        elif opcion_reservas == "Ver turnos disponibles":
            sistema.listar_turnos()

    # GESTION DE SANCIONES
    elif menu == "Gestion de Sanciones":
        st.header("Gestion de Sanciones")

        col1, col2 = st.columns(2)

        with col1:
            with st.form("aplicar_sancion"):
                st.subheader("Aplicar Sancion")
                ci_participante = st.text_input("Cedula del participante")
                fecha_inicio = st.date_input("Fecha inicio", min_value=datetime.datetime.now().date())

                if st.form_submit_button("Aplicar Sancion"):
                    sistema.aplicar_sancion(ci_participante, fecha_inicio)

        with col2:
            st.subheader("Sanciones Automaticas")
            if st.button("Verificar Reservas Sin Asistencia"):
                sistema.verificar_reservas_sin_asistencia()

    # REPORTES Y CONSULTAS
    elif menu == "Reportes y Consultas":
        st.header("Reportes y Consultas")

        reporte = st.selectbox("Seleccione Reporte", [
            "Salas mas reservadas",
            "Turnos mas demandados",
            "Promedio participantes por sala",
            "Reservas por carrera y facultad",
            "Ocupacion por edificio",
            "Reservas y asistencias por tipo",
            "Sanciones por tipo de usuario",
            "Uso de reservas",
            "Participantes mas activos",
            "Salas sin uso",
            "Horarios pico"
        ])

        if st.button("Generar Reporte"):
            if reporte == "Salas mas reservadas":
                sistema.reporte_salas_populares()
            elif reporte == "Turnos mas demandados":
                sistema.reporte_turnos_demandados()
            elif reporte == "Promedio participantes por sala":
                sistema.reporte_promedio_participantes()
            elif reporte == "Reservas por carrera y facultad":
                sistema.reporte_reservas_carrera_facultad()
            elif reporte == "Ocupacion por edificio":
                sistema.reporte_ocupacion_edificio()
            elif reporte == "Reservas y asistencias por tipo":
                sistema.reporte_reservas_asistencias()
            elif reporte == "Sanciones por tipo de usuario":
                sistema.reporte_sanciones_tipo()
            elif reporte == "Uso de reservas":
                sistema.reporte_uso_reservas()
            elif reporte == "Participantes mas activos":
                sistema.reporte_participantes_activos()
            elif reporte == "Salas sin uso":
                sistema.reporte_salas_sin_uso()
            elif reporte == "Horarios pico":
                sistema.reporte_horarios_pico()


if __name__ == "__main__":
    main_streamlit()
