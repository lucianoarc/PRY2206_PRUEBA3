-- Creación del Package
CREATE OR REPLACE PACKAGE pkg_calculo_remuneraciones AS
    -- Procedimiento para registrar errores en la tabla ERROR_CALC
    PROCEDURE registrar_error(p_subprograma VARCHAR2, p_mensaje VARCHAR2, p_descripcion VARCHAR2);
    
    -- Función que calcula el promedio de ventas del año anterior
    FUNCTION promedio_ventas_anio_anterior RETURN NUMBER;
    
    -- Variable global para almacenar el promedio de ventas
    v_promedio_ventas NUMBER;
END pkg_calculo_remuneraciones;
/
create or replace PACKAGE BODY pkg_calculo_remuneraciones AS
    -- Procedimiento para registrar errores en la tabla ERROR_CALC
    PROCEDURE registrar_error(p_subprograma VARCHAR2, p_mensaje VARCHAR2, p_descripcion VARCHAR2) IS
    BEGIN
        INSERT INTO ERROR_CALC (CORREL_ERROR, RUTINA_ERROR, DESCRIP_ERROR, DESCRIP_USER)
        VALUES (SEQ_ERROR.NEXTVAL, p_subprograma, p_mensaje, p_descripcion);
    END registrar_error;
    
    -- Función que calcula el promedio de ventas del año anterior
    FUNCTION promedio_ventas_anio_anterior RETURN NUMBER IS
        v_promedio NUMBER;
    BEGIN
        SELECT NVL(AVG(MONTO_TOTAL_BOLETA), 0) INTO v_promedio
        FROM BOLETA
        WHERE EXTRACT(YEAR FROM FECHA) = EXTRACT(YEAR FROM SYSDATE) -2;
        RETURN v_promedio;
    EXCEPTION
        WHEN OTHERS THEN
            registrar_error('promedio_ventas_anio_anterior', SQLERRM, 'Error al calcular promedio de ventas.');
            RETURN 0;
    END promedio_ventas_anio_anterior;

END pkg_calculo_remuneraciones;
/
--procedimiento para completar la tabla LIQUIDACION_EMPLEADO
CREATE OR REPLACE PROCEDURE calcular_liquidacion(p_fecha_proceso DATE) IS
BEGIN
    DELETE FROM LIQUIDACION_EMPLEADO WHERE ANNO = EXTRACT(YEAR FROM p_fecha_proceso) AND MES = EXTRACT(MONTH FROM p_fecha_proceso);
    
    FOR emp IN (SELECT e.RUN_EMPLEADO, e.NOMBRE,e.PATERNO, e.SUELDO_BASE, e.FECHA_CONTRATO, e.COD_ESCOLARIDAD FROM EMPLEADO e) LOOP
        DECLARE
            v_anios_trabajados NUMBER := FLOOR(MONTHS_BETWEEN(p_fecha_proceso, emp.FECHA_CONTRATO) / 12);
            v_porcentaje_antiguedad NUMBER := obtener_porcentaje_antiguedad(emp.RUN_EMPLEADO, v_anios_trabajados);
            v_porcentaje_estudios NUMBER := obtener_porcentaje_estudios(emp.COD_ESCOLARIDAD);
            v_asignacion_antiguedad NUMBER := emp.SUELDO_BASE * v_porcentaje_antiguedad;
            v_asignacion_estudios NUMBER := emp.SUELDO_BASE * v_porcentaje_estudios;
            v_total_haberes NUMBER := emp.SUELDO_BASE + v_asignacion_antiguedad + v_asignacion_estudios;
        BEGIN
            INSERT INTO LIQUIDACION_EMPLEADO (MES, ANNO, RUN_EMPLEADO, NOMBRE_EMPLEADO, SUELDO_BASE, ASIG_ESPECIAL, ASIG_ESTUDIOS, TOTAL_HABERES)
            VALUES (EXTRACT(MONTH FROM p_fecha_proceso), EXTRACT(YEAR FROM p_fecha_proceso), emp.RUN_EMPLEADO, emp.NOMBRE||' '||emp.PATERNO, emp.SUELDO_BASE, v_asignacion_antiguedad, v_asignacion_estudios, v_total_haberes);
        EXCEPTION
            WHEN OTHERS THEN
                pkg_calculo_remuneraciones.registrar_error('calcular_liquidacion', SQLERRM, 'Error en INSERT LIQUIDACION_EMPLEADO');
        END;
    END LOOP;
END calcular_liquidacion;
/
--Funcion para calcular porcentaje_espcial del los empleados
CREATE OR REPLACE FUNCTION obtener_porcentaje_antiguedad(p_run_empleado VARCHAR2, p_anios NUMBER) RETURN NUMBER IS
    v_porcentaje NUMBER;
    v_ventas_anuales NUMBER;
    v_promedio_ventas NUMBER;
BEGIN
    -- Obtener las ventas anuales del empleado
    SELECT NVL(SUM(MONTO_TOTAL_BOLETA), 0) INTO v_ventas_anuales
    FROM BOLETA
    WHERE EXTRACT(YEAR FROM FECHA) = EXTRACT(YEAR FROM SYSDATE) -1
    AND RUN_EMPLEADO = p_run_empleado;

    -- Obtener el promedio de ventas del año anterior
    v_promedio_ventas := pkg_calculo_remuneraciones.promedio_ventas_anio_anterior;

    -- Validar si el 17% de las ventas anuales supera el promedio del año pasado
    IF (NVL(v_ventas_anuales, 0) * 0.17) > NVL(v_promedio_ventas, 0) THEN
        -- Obtener el porcentaje de asignación por antigüedad
        SELECT PORC_ANTIGUEDAD INTO v_porcentaje
        FROM PCT_ANTIGUEDAD
        WHERE p_anios BETWEEN ANNOS_ANTIGUEDAD_INF AND ANNOS_ANTIGUEDAD_SUP;
    ELSE
        v_porcentaje := 0;  -- No cumple la condición, no se otorga asignación
    END IF;

    RETURN v_porcentaje;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 0;
    WHEN OTHERS THEN
        pkg_calculo_remuneraciones.registrar_error('PCT ESPECIAL', SQLERRM, 'Error al obtener porcentaje.');
        RETURN 0;
END obtener_porcentaje_antiguedad;
/
--Funcion para calcular porcentaje_estudios del los empleados
create or replace FUNCTION obtener_porcentaje_estudios(p_nivel NUMBER) RETURN NUMBER IS
    v_porcentaje NUMBER;
BEGIN
    SELECT PORC_ESCOLARIDAD INTO v_porcentaje
    FROM PCT_NIVEL_ESTUDIOS
    WHERE COD_ESCOLARIDAD = p_nivel;
    RETURN v_porcentaje / 100;
EXCEPTION
    WHEN OTHERS THEN
        pkg_calculo_remuneraciones.registrar_error('ESTUDIOS', SQLERRM, 'Error al obtener porcentaje estudios.');
        RETURN 0;
END obtener_porcentaje_estudios;
/
--trigger para actualizar precios
create or replace TRIGGER trg_actualizacion_precio
BEFORE UPDATE OF VALOR_UNITARIO ON PRODUCTO
FOR EACH ROW
DECLARE
    v_promedio_ventas NUMBER;
BEGIN
    v_promedio_ventas := pkg_calculo_remuneraciones.promedio_ventas_anio_anterior;
    IF :NEW.VALOR_UNITARIO > (v_promedio_ventas * 1.1) THEN
        UPDATE DETALLE_BOLETA
        SET VALOR_TOTAL = CANTIDAD * :NEW.VALOR_UNITARIO
        WHERE COD_PRODUCTO = :OLD.COD_PRODUCTO;
    END IF;
END;
/
--trigger para control de productos
create or replace TRIGGER trg_control_productos
BEFORE INSERT OR DELETE ON PRODUCTO
FOR EACH ROW
DECLARE
    v_dia_semana NUMBER;
BEGIN
    v_dia_semana := TO_CHAR(SYSDATE, 'D');
    IF v_dia_semana BETWEEN 2 AND 6 THEN
        IF INSERTING THEN
            RAISE_APPLICATION_ERROR(-20501, 'No se pueden insertar productos de lunes a viernes.');
        ELSIF DELETING THEN
            RAISE_APPLICATION_ERROR(-20500, 'No se pueden eliminar productos de lunes a viernes.');
        END IF;
    END IF;
END;
/
--Creación de usuario si está trabajando con BD Oracle Cloud 
CREATE USER PRY2206_PRUEBA3 IDENTIFIED BY "PRY2206.prueba_3"
DEFAULT TABLESPACE DATA
TEMPORARY TABLESPACE TEMP;
ALTER USER PRY2206_PRUEBA3 QUOTA UNLIMITED ON DATA;
GRANT CREATE SESSION TO PRY2206_PRUEBA3;
GRANT RESOURCE TO PRY2206_PRUEBA3;
ALTER USER PRY2206_PRUEBA3 DEFAULT ROLE RESOURCE;

BEGIN
    calcular_liquidacion(TO_DATE('01-06-2024', 'DD-MM-YYYY'));
END;

SELECT * FROM LIQUIDACION_EMPLEADO ORDER BY RUN_EMPLEADO ASC;
SELECT * FROM PRODUCTO;
SELECT * FROM ERROR_CALC;

INSERT INTO PRODUCTO (COD_PRODUCTO, DESCRIPCION, COD_UNIDAD, VALOR_UNITARIO, TOTAL_STOCK, STOCK_MINIMO, PROCEDENCIA) 
VALUES (999, 'Producto Prueba', 'UN', 5000, 100, 10, 'N');

DELETE FROM PRODUCTO WHERE COD_PRODUCTO = 999;

UPDATE PRODUCTO 
SET VALOR_UNITARIO = 1000
WHERE COD_PRODUCTO = 19;

SELECT * FROM DETALLE_BOLETA WHERE COD_PRODUCTO = 19;

