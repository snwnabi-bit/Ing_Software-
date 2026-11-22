import uuid

from app.database import DomainHttpError, atomic_transaction
from app.services import _now


def registrar_discrepancia_picking(
    dark_store_id: str,
    sku_id: str,
    conteo_fisico: int,
    picker_id: str = "PICKER-01",
    motivo: str = "Conteo físico en anaquel menor a existencia lógica (AT-07)",
) -> dict:
    """
    AT-07 y S2-03:
    Simula discrepancia (ej. sistema=10, físico=8), registra la incidencia
    auditable, bloquea de inmediato la disponibilidad incierta
    (estado_sku = 'INHABILITADO') y registra el movimiento FALTANTE_PICKING
    sin cancelar a ciegas pedidos ya pagados.
    """
    if conteo_fisico < 0:
        raise DomainHttpError(
            400, "CONTEO_INVALIDO", "El conteo físico no puede ser negativo."
        )

    with atomic_transaction() as conn:
        now = _now(0.0)

        inv = conn.execute(
            "SELECT * FROM inventario_tienda WHERE dark_store_id = ? AND sku_id = ?",
            (dark_store_id, sku_id),
        ).fetchone()
        if not inv:
            raise DomainHttpError(
                404, "SKU_NO_EXISTE", f"SKU {sku_id} no existe en {dark_store_id}"
            )

        existencia_logica = inv["existencia_e"]
        oculto_anterior = inv["oculto_h"]
        diferencia = conteo_fisico - existencia_logica  # 8 - 10 = -2
        unidades_inciertas = max(0, existencia_logica - conteo_fisico)
        incidencia_id = f"INC-{uuid.uuid4().hex[:8].upper()}"

        # 1) Bloquear disponibilidad incierta de inmediato
        conn.execute(
            """
            UPDATE inventario_tienda
            SET estado_sku = 'INHABILITADO',
                oculto_h = ?,
                version_concurrency = version_concurrency + 1
            WHERE inventario_id = ?
            """,
            (unidades_inciertas, inv["inventario_id"]),
        )

        # 2) Evidencia formal de la discrepancia
        conn.execute(
            """
            INSERT INTO incidencia_inventario
                (incidencia_id, inventario_id, dark_store_id, sku_id,
                 reportado_por, existencia_logica, conteo_fisico, diferencia,
                 estado_incidencia, motivo_detalle, registrada_epoch)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, 'PENDIENTE_VERIFICACION_SUPERVISOR', ?, ?)
            """,
            (
                incidencia_id, inv["inventario_id"], dark_store_id, sku_id,
                picker_id, existencia_logica, conteo_fisico, diferencia,
                motivo, now,
            ),
        )

        # 3) Movimiento auditable (delta_h = cambio real en unidades ocultas)
        delta_h = unidades_inciertas - oculto_anterior
        conn.execute(
            """
            INSERT INTO movimiento_inventario
                (movimiento_id, inventario_id, usuario_id, tipo_movimiento,
                 delta_e, delta_r, delta_c, delta_h, correlation_id,
                 motivo_detalle, registrado_epoch)
            VALUES (?, ?, ?, 'FALTANTE_PICKING', 0, 0, 0, ?, ?, ?, ?)
            """,
            (
                str(uuid.uuid4()), inv["inventario_id"], picker_id, delta_h,
                incidencia_id,
                f"Discrepancia lógica={existencia_logica} vs física={conteo_fisico}. SKU INHABILITADO.",
                now,
            ),
        )

    return {
        "incidencia_id": incidencia_id,
        "dark_store_id": dark_store_id,
        "sku_id": sku_id,
        "existencia_logica": existencia_logica,
        "conteo_fisico": conteo_fisico,
        "diferencia": diferencia,
        "estado_sku_nuevo": "INHABILITADO",
        "estado_incidencia": "PENDIENTE_VERIFICACION_SUPERVISOR",
        "disponible_para_venta": 0,
        "mensaje": "AT-07: Incidencia registrada y disponibilidad incierta bloqueada de inmediato.",
    }
