# 📖 AtlasSpence — Contexto del Proyecto y Arquitectura v2

> **Documento de Arquitectura y Negocio**
> Define el estándar técnico, el modelo de datos relacional y las reglas operativas para el sistema integrado de gestión de activos Atlas Copco en la faena Spence.

---

## 1. Visión General del Proyecto

AtlasSpence es un **sistema integrado de gestión de activos y control de inventario** diseñado para los sectores de Hidrometalurgia y SGO. 

La plataforma abarca el ciclo de vida completo de los equipos industriales (compresores GA, secadores, etc.). Centraliza la planificación y ejecución de mantenimientos, el control financiero del inventario mediante un Kardex transaccional, la captura de parámetros críticos (horómetros y mediciones en JSON) y la generación de entregables validados con firmas digitales.

El sistema está diseñado como una **PWA (Progressive Web App) con capacidades offline** robustas. Los técnicos pueden operar en terreno, registrar bitácoras, ejecutar intervenciones y capturar firmas sin conexión, para luego sincronizar los datos de forma segura al recuperar la red.

---

## 2. Stack Tecnológico Objetivo

*   **Base de Datos & Backend:** Supabase (PostgreSQL). Motor de reglas, seguridad (RLS/Column-Level) y triggers de auditoría.
*   **Frontend PWA:** React + Vite + TypeScript.
*   **Sincronización:** PowerSync (SQLite local + Postgres remoto).
*   **Lógica Serverless:** Supabase Edge Functions / Microservicios (Generación de PDF y envío de correos).

---

## 3. Arquitectura de Base de Datos (Dominios)

El esquema físico está normalizado (3FN) y dividido en cuatro grandes dominios operativos:

### 3.1. Personas, Activos y Auditoría
*   **`perfil_de_usuario` / `catalogo_contacto_cliente`:** Gestión de accesos internos (Supabase Auth) y directorio de aprobadores externos (BHP). El perfil extrae su división implícitamente del sufijo del rol (ej. `tecnico_hidro`).
*   **`equipo`:** Maestro de activos físicos. Incluye un campo `especificaciones_tecnicas` (JSONB) para alojar datos variables de la placa.
*   **`observacion_equipo`:** Bitácora libre para reportar hallazgos en terreno sin abrir una orden de trabajo formal.
*   **`log_auditoria`:** Registro inmutable (vía triggers) que captura quién modificó qué tabla, guardando el estado anterior y nuevo (`detalles_cambio`).

### 3.2. CMMS (Planificación y Ejecución)
*   **`planificacion_mantenimiento`:** Cabecera del ciclo de trabajo (Semanal o Mensual).
*   **`tarea_programada`:** Entidad atómica que cruza el plan con el equipo. Su estado alimenta directamente el KPI de cumplimiento.
*   **`intervencion`:** Ejecución real del trabajo. Almacena la carga útil del informe y actúa como llave obligatoria para retirar repuestos de bodega.
*   **`registro_firma`:** Separa las firmas del informe, permitiendo múltiples firmantes por intervención.

### 3.3. Catálogos y Kits (BOM)
*   **`catalogo_de_repuestos`:** Maestro de SKUs. Diferencia la naturaleza del ítem (`componente` vs `kit_cerrado`), y de forma ortogonal si requiere trazabilidad unitaria (`es_serializado`).
*   **`kit_de_mantenimiento` / `composicion_del_kit`:** Estructura la lista de materiales (Receta/BOM) para cada categoría de equipo y nivel de mantención (P1-P4).
*   **`kits_por_equipo`:** Relaciona qué kits específicos aplican a cada máquina.

### 3.4. Bodega y Kardex
*   **`documento_de_movimiento`:** Cabecera transaccional que define la `naturaleza` (ingreso, egreso, transferencia, ajuste).
*   **`movimiento_de_inventario`:** Detalle del Kardex. Toda variación de stock es un registro aquí.
*   **`activo_serializado`:** Trazabilidad física unitaria (En stock, instalado, de baja).
*   **`existencias_en_stock`:** Vista SQL que deriva el stock actual calculando dinámicamente las sumas y restas del Kardex por bodega y repuesto.

---

## 4. Reglas de Negocio Inquebrantables

1.  **El Mes Minero (Periodo Financiero):** El ciclo de planificación "Mensual_Programada" opera estricta y exclusivamente **desde el día 16 del mes anterior al día 15 del mes en curso**.
2.  **Lógica Fuera de Servicio (F/S):** Si un equipo reporta estado F/S, únicamente sus tareas **pendientes correspondientes al mes minero en curso** pasan a estado `fs`, excluyéndose de la penalización en el KPI actual sin alterar el historial pasado ni la planificación futura.
3.  **Integridad de Consumo:** Todo documento de movimiento con naturaleza de `egreso` (consumo) exige obligatoriamente un `intervencion_id` asociado. 
4.  **Idempotencia Offline:** La tabla `intervencion` posee una `idempotency_key` única que evita la duplicación de reportes PDF y correos al sincronizar tras periodos sin conexión.

---

## 5. Seguridad, Aislamiento y Enmascaramiento

*   **Aislamiento Geográfico (Row-Level Security - RLS):** Políticas estrictas a nivel de fila aíslan los datos por división. La división del usuario se infiere de su rol. Para proteger el acceso a las tablas del Kardex, las políticas RLS realizan un `JOIN` dinámico hacia la tabla `bodega` asociada para verificar la división permitida.
*   **Enmascaramiento Financiero (Column-Level Security / Vistas):** La restricción de visibilidad sobre los datos monetarios (`costo_referencia`, `costo_adquisicion`, `costo_unitario_historico`) se gestiona mediante privilegios a nivel de columna o a través de Vistas SQL en la capa de aplicación. El parámetro `puede_ver_costos` determina si la API entrega estos campos al cliente, impidiendo que los perfiles técnicos visualicen información confidencial.