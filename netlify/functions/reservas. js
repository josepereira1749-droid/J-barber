const { neon } = require('@netlify/neon');

const HEADERS = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'Content-Type',
  'Access-Control-Allow-Methods': 'GET,POST,PATCH,DELETE,OPTIONS',
  'Content-Type': 'application/json'
};

exports.handler = async (event) => {
  if (event.httpMethod === 'OPTIONS') {
    return { statusCode: 200, headers: HEADERS, body: '' };
  }

  const sql = neon(); // usa NETLIFY_DATABASE_URL automáticamente

  // Crea la tabla si todavía no existe (no hace daño si ya existe)
  await sql`
    CREATE TABLE IF NOT EXISTS reservas (
      id SERIAL PRIMARY KEY,
      nombre TEXT NOT NULL,
      telefono TEXT NOT NULL,
      servicio TEXT NOT NULL,
      fecha DATE NOT NULL,
      hora TEXT NOT NULL,
      estado TEXT NOT NULL DEFAULT 'pendiente',
      creado_en TIMESTAMP DEFAULT now()
    )
  `;

  try {
    // ---- GET: consultar disponibilidad (cliente) o listado completo (barbero) ----
    if (event.httpMethod === 'GET') {
      const params = event.queryStringParameters || {};

      // Panel del barbero: requiere clave
      if (params.admin) {
        if (params.admin !== process.env.ADMIN_KEY) {
          return { statusCode: 401, headers: HEADERS, body: JSON.stringify({ error: 'Clave incorrecta' }) };
        }
        const rows = await sql`SELECT * FROM reservas ORDER BY fecha DESC, hora ASC`;
        return { statusCode: 200, headers: HEADERS, body: JSON.stringify(rows) };
      }

      // Cliente: consulta sus propias reservas por teléfono
      if (params.telefono) {
        const rows = await sql`
          SELECT nombre, servicio, fecha, hora, estado FROM reservas
          WHERE telefono = ${params.telefono}
          ORDER BY fecha DESC, hora DESC
        `;
        return { statusCode: 200, headers: HEADERS, body: JSON.stringify(rows) };
      }

      // Cliente: consulta horarios ocupados de un día
      if (!params.fecha) {
        return { statusCode: 400, headers: HEADERS, body: JSON.stringify({ error: 'Falta la fecha' }) };
      }
      const rows = await sql`
        SELECT hora, estado FROM reservas
        WHERE fecha = ${params.fecha} AND estado != 'rechazada'
      `;
      return { statusCode: 200, headers: HEADERS, body: JSON.stringify(rows) };
    }

    // ---- POST: crear una reserva nueva ----
    if (event.httpMethod === 'POST') {
      const data = JSON.parse(event.body || '{}');
      const { nombre, telefono, servicio, fecha, hora } = data;

      if (!nombre || !telefono || !servicio || !fecha || !hora) {
        return { statusCode: 400, headers: HEADERS, body: JSON.stringify({ error: 'Faltan datos' }) };
      }

      const ocupado = await sql`
        SELECT id FROM reservas
        WHERE fecha = ${fecha} AND hora = ${hora} AND estado != 'rechazada'
      `;
      if (ocupado.length > 0) {
        return { statusCode: 409, headers: HEADERS, body: JSON.stringify({ error: 'Ese horario ya está ocupado' }) };
      }

      const [nueva] = await sql`
        INSERT INTO reservas (nombre, telefono, servicio, fecha, hora)
        VALUES (${nombre}, ${telefono}, ${servicio}, ${fecha}, ${hora})
        RETURNING *
      `;
      return { statusCode: 200, headers: HEADERS, body: JSON.stringify(nueva) };
    }

    // ---- PATCH: el barbero acepta o rechaza una reserva ----
    if (event.httpMethod === 'PATCH') {
      const data = JSON.parse(event.body || '{}');
      const { id, estado, admin } = data;

      if (admin !== process.env.ADMIN_KEY) {
        return { statusCode: 401, headers: HEADERS, body: JSON.stringify({ error: 'Clave incorrecta' }) };
      }
      if (!id || !['aceptada', 'rechazada'].includes(estado)) {
        return { statusCode: 400, headers: HEADERS, body: JSON.stringify({ error: 'Datos inválidos' }) };
      }

      const [actualizada] = await sql`
        UPDATE reservas SET estado = ${estado} WHERE id = ${id} RETURNING *
      `;
      return { statusCode: 200, headers: HEADERS, body: JSON.stringify(actualizada) };
    }

    // ---- DELETE: borrar una reserva individual o todas las de un día ----
    if (event.httpMethod === 'DELETE') {
      const data = JSON.parse(event.body || '{}');
      const { id, fecha, admin } = data;

      if (admin !== process.env.ADMIN_KEY) {
        return { statusCode: 401, headers: HEADERS, body: JSON.stringify({ error: 'Clave incorrecta' }) };
      }

      if (id) {
        await sql`DELETE FROM reservas WHERE id = ${id}`;
        return { statusCode: 200, headers: HEADERS, body: JSON.stringify({ ok: true }) };
      }
      if (fecha) {
        await sql`DELETE FROM reservas WHERE fecha = ${fecha}`;
        return { statusCode: 200, headers: HEADERS, body: JSON.stringify({ ok: true }) };
      }
      return { statusCode: 400, headers: HEADERS, body: JSON.stringify({ error: 'Falta id o fecha' }) };
    }

    return { statusCode: 405, headers: HEADERS, body: JSON.stringify({ error: 'Método no permitido' }) };
  } catch (err) {
    return { statusCode: 500, headers: HEADERS, body: JSON.stringify({ error: err.message }) };
  }
};
