# Taller-de-Gesti-n-Avanzada-de-Bases-de-Datos-con-MySQL-8.0

Ejercicio 1: Creación de Índices Básicos
Objetivo: Mejorar el rendimiento de búsquedas en la tabla Libros.

Conéctate a la base de datos Biblioteca:

    USE Biblioteca;

Crea los índices necesarios:

    CREATE INDEX idx_libros_titulo ON Libros(titulo);
    CREATE INDEX idx_libros_autor ON Libros(autor);

Verifica el plan de ejecución antes y después de crear los índices:

    EXPLAIN SELECT id_libro, titulo, autor 
    FROM Libros 
    WHERE titulo LIKE 'Algoritmos%' OR autor = 'Donald Knuth';

Resultado esperado: El EXPLAIN debería mostrar una mejora en el tipo de acceso (type) de ALL 
(escaneo completo) a range o ref, indicando que se están utilizando los índices creados.

Ejercicio 2: Optimización de Consultas con EXPLAIN
Objetivo: Identificar y solucionar cuellos de botella en una consulta compleja.

Analiza la consulta problemática:

    EXPLAIN SELECT p.id_prestamo, u.nombre, l.titulo, p.fecha_prestamo
    FROM Prestamos p
    JOIN Usuarios u ON p.id_usuario = u.id_usuario
    JOIN Ejemplares e ON p.id_ejemplar = e.id_ejemplar
    JOIN Libros l ON e.id_libro = l.id_libro
    WHERE p.fecha_devolucion IS NULL
      AND l.categoria = 'Ciencia de la Computación';

Crea los índices sugeridos:

    CREATE INDEX idx_prestamos_devolucion ON Prestamos(fecha_devolucion);
    CREATE INDEX idx_libros_categoria_id ON Libros(categoria, id_libro);
    CREATE INDEX idx_ejemplares_libro_ejemplar ON Ejemplares(id_libro, id_ejemplar);

Vuelve a ejecutar el EXPLAIN y compara los resultados.

Resultado esperado: La consulta optimizada debería mostrar un menor número de filas escaneadas (rows) y un uso adecuado de los índices creados.

Ejercicio 3: Implementación de Transacción Segura
Objetivo: Garantizar la consistencia de los datos durante operaciones concurrentes de préstamo.

Implementa una transacción segura:

    START TRANSACTION;

    -- Bloquea el ejemplar para evitar préstamos simultáneos
    SELECT disponible 
    FROM Ejemplares 
    WHERE id_ejemplar = 321 
    FOR UPDATE;

    -- Verifica disponibilidad (la aplicación debe chequear el resultado)
    -- Si está disponible, procede con el préstamo
    INSERT INTO Prestamos(fecha_prestamo, fecha_limite, id_ejemplar, id_usuario)
    VALUES (CURRENT_DATE(), DATE_ADD(CURRENT_DATE(), INTERVAL 14 DAY), 321, 789);

    -- Marca el ejemplar como no disponible
    UPDATE Ejemplares 
    SET disponible = FALSE 
    WHERE id_ejemplar = 321;

    COMMIT;

Prueba: Ejecuta esta transacción desde dos sesiones simultáneamente para verificar que no se dupliquen préstamos del mismo ejemplar.

Ejercicio 4: Procedimiento Almacenado para Cálculo de Multas
Objetivo: Centralizar la lógica de cálculo de multas en la base de datos.

Crea el procedimiento almacenado:

    DELIMITER //
    CREATE PROCEDURE calcular_multa_simple(
    IN p_id_prestamo INT,
    OUT p_monto_multa DECIMAL(10,2)
    )
    BEGIN
    DECLARE v_fecha_limite DATE;
    DECLARE v_fecha_devolucion DATE;
    DECLARE v_dias_retraso INT;

    SELECT fecha_limite, fecha_devolucion 
    INTO v_fecha_limite, v_fecha_devolucion
    FROM Prestamos
    WHERE id_prestamo = p_id_prestamo;

    IF v_fecha_devolucion IS NULL THEN
        SET v_fecha_devolucion = CURRENT_DATE();
    END IF;

    SET v_dias_retraso = DATEDIFF(v_fecha_devolucion, v_fecha_limite);

    IF v_dias_retraso > 0 THEN
        SET p_monto_multa = v_dias_retraso * 1.00;
    ELSE
        SET p_monto_multa = 0.00;
    END IF;
    END //
    DELIMITER ;

Prueba el procedimiento:

    SET @multa = 0.00;
    CALL calcular_multa_simple(15, @multa);
    SELECT @multa AS monto_multa_calculado;

  Ejercicio 5: Función para Identificar Usuarios Morosos
Objetivo: Crear una función que determine si un usuario tiene préstamos vencidos.

Implementa la función:

    DELIMITER //
    CREATE FUNCTION es_usuario_moroso(
    p_id_usuario INT
    ) RETURNS BOOL
    DETERMINISTIC
    READS SQL DATA
    BEGIN
    DECLARE v_count INT;

    SELECT COUNT(*)
    INTO v_count
    FROM Prestamos
    WHERE id_usuario = p_id_usuario
      AND fecha_devolucion IS NULL
      AND fecha_limite < CURRENT_DATE();

    RETURN v_count > 0;
    END //
    DELIMITER ;

Prueba la función:

    SELECT id_usuario, es_usuario_moroso(id_usuario) AS moroso
    FROM Usuarios
    WHERE id_usuario IN (123, 456, 789);

  Ejercicio 6: Triggers para Historial de Préstamos
Objetivo: Automatizar el registro de auditoría para préstamos y devoluciones.

Crea la tabla de historial si no existe:

    CREATE TABLE IF NOT EXISTS Historial_Prestamos (
    id_historial INT AUTO_INCREMENT PRIMARY KEY,
    id_prestamo INT NOT NULL,
    id_usuario INT NOT NULL,
    id_ejemplar INT NOT NULL,
    tipo_evento ENUM('PRESTAMO','DEVOLUCION') NOT NULL,
    fecha_evento DATETIME NOT NULL
    );

 Implementa los triggers:

    DELIMITER //
    CREATE TRIGGER trg_historial_prestamo
    AFTER INSERT ON Prestamos
    FOR EACH ROW
    BEGIN
    INSERT INTO Historial_Prestamos(
        id_prestamo,
        id_usuario,
        id_ejemplar,
        tipo_evento,
        fecha_evento
    ) VALUES (
        NEW.id_prestamo,
        NEW.id_usuario,
        NEW.id_ejemplar,
        'PRESTAMO',
        NOW()
    );
    END //
    DELIMITER ;

    DELIMITER //
    CREATE TRIGGER trg_historial_devolucion
    AFTER UPDATE ON Prestamos
    FOR EACH ROW
    BEGIN
    IF NEW.fecha_devolucion IS NOT NULL AND OLD.fecha_devolucion IS NULL THEN
        INSERT INTO Historial_Prestamos(
            id_prestamo,
            id_usuario,
            id_ejemplar,
            tipo_evento,
            fecha_evento
        ) VALUES (
            NEW.id_prestamo,
            NEW.id_usuario,
            NEW.id_ejemplar,
            'DEVOLUCION',
            NOW()
        );
    END IF;
    END //
    DELIMITER ;
