<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Violet Evergarden: gestion AMD</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            max-width: 850px;
            margin: auto;
            padding: 20px;
            background-color: #f8f9fa;
        }
        .container {
            padding: 15px;
            border-radius: 10px;
            background-color: #ffffff;
            box-shadow: 0 4px 8px rgba(0,0,0,0.1);
        }
        .section {
            margin-bottom: 25px;
            border: 1px solid #e9ecef;
            padding: 20px;
            border-radius: 8px;
            background-color: #f8f9fa;
        }
        h1, h2 {
            color: #343a40;
            border-bottom: 2px solid #007bff;
            padding-bottom: 5px;
            margin-top: 0;
        }
        form div { margin-bottom: 12px; }
        label { display: block; margin-bottom: 5px; font-weight: 600; color: #495057; }
        input, textarea, button, select {
            width: 100%;
            padding: 10px;
            box-sizing: border-box;
            border: 1px solid #ced4da;
            border-radius: 5px;
        }
        button {
            cursor: pointer;
            background-color: #007bff;
            color: white;
            border: none;
            margin-top: 10px;
            transition: background-color 0.3s ease;
        }
        button:hover { background-color: #0056b3; }
        .delete-btn { background-color: #dc3545; }
        .delete-btn:hover { background-color: #c82333; }
        .list-container {
            margin-top: 20px;
        }
        ul { list-style-type: none; padding: 0; }
        li {
            background-color: #e9ecef;
            padding: 12px;
            margin-bottom: 8px;
            border-radius: 5px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Violet Evergarden: gestion AMD</h1>
        
        <div class="section">
            <h2>Crear una Nueva Doll</h2>
            <form id="crear-doll-form">
                <div>
                    <label for="doll-nombre">Nombre:</label>
                    <input type="text" id="doll-nombre" required>
                </div>
                <div>
                    <label for="doll-edad">Edad:</label>
                    <input type="number" id="doll-edad">
                </div>
                <div>
                    <label for="doll-estado">Estado (ej. "en servicio", "inactiva"):</label>
                    <input type="text" id="doll-estado" value="en servicio">
                </div>
                <button type="submit">Registrar Doll</button>
            </form>
        </div>

        <div class="section">
            <h2>Crear un Cliente</h2>
            <form id="crear-cliente-form">
                <div>
                    <label for="cliente-nombre">Nombre del Cliente:</label>
                    <input type="text" id="cliente-nombre" required>
                </div>
                <div>
                    <label for="cliente-ciudad">Ciudad:</label>
                    <input type="text" id="cliente-ciudad">
                </div>
                <button type="submit">Registrar Cliente</button>
            </form>
        </div>

        <div class="section">
            <h2>Crear una Carta</h2>
            <form id="crear-carta-form">
                <div>
                    <label for="carta-cliente-id">ID del Cliente:</label>
                    <input type="number" id="carta-cliente-id" required>
                </div>
                <div>
                    <label for="carta-contenido">Contenido de la carta:</label>
                    <textarea id="carta-contenido" required></textarea>
                </div>
                <button type="submit">Asignar Carta</button>
            </form>
        </div>

        <div class="section">
            <h2>Listado de Dolls Registradas</h2>
            <ul id="lista-dolls"></ul>
        </div>
        
        <div class="section">
            <h2>Listado de Clientes</h2>
            <ul id="lista-clientes"></ul>
        </div>
        
        <div class="section">
            <h2>Listado de Cartas por Escribir</h2>
            <ul id="lista-cartas"></ul>
        </div>

        <div class="section">
            <h2>Reporte de Trabajo por Doll</h2>
            <ul id="reporte-dolls"></ul>
        </div>
    </div>

    <script>
        const API_BASE = 'http://127.0.0.1:5000';

        async function cargarListas() {
            try {
                const dolls = await (await fetch(`${API_BASE}/dolls`)).json();
                renderizarLista('lista-dolls', dolls, d => `ID: ${d.id}, ${d.nombre}, Estado: ${d.estado}, Cartas: ${d.cartas_escritas}`, 'dolls');
                
                const clientes = await (await fetch(`${API_BASE}/clientes`)).json();
                renderizarLista('lista-clientes', clientes, c => `ID: ${c.id}, ${c.nombre}, Ciudad: ${c.ciudad}`, 'clientes');

                const cartas = await (await fetch(`${API_BASE}/cartas`)).json();
                renderizarLista('lista-cartas', cartas, c => `ID: ${c.id}, Cliente ID: ${c.cliente_id}, Doll ID: ${c.doll_asignada_id}, Estado: ${c.estado}`, 'cartas');

                const reporte = await (await fetch(`${API_BASE}/reporte/dolls`)).json();
                renderizarLista('reporte-dolls', reporte, r => `${r.doll}: ${r.cartas_enviadas} cartas enviadas a ${r.clientes_distintos} clientes distintos.`);

            } catch (error) {
                console.error("Error al cargar los datos:", error);
                alert("Ocurrió un error al conectar con el servidor.");
            }
        }

        function renderizarLista(idElemento, datos, formatear, tipoRecurso = null) {
            const ul = document.getElementById(idElemento);
            ul.innerHTML = '';
            if (!datos || datos.length === 0) {
                ul.innerHTML = '<li>No hay registros.</li>';
                return;
            }
            datos.forEach(item => {
                const li = document.createElement('li');
                li.textContent = formatear(item);

                if (tipoRecurso) {
                    const botonBorrar = document.createElement('button');
                    botonBorrar.textContent = 'Eliminar';
                    botonBorrar.className = 'delete-btn';
                    botonBorrar.onclick = () => borrarRecurso(item.id, tipoRecurso);
                    li.appendChild(botonBorrar);
                }
                ul.appendChild(li);
            });
        }
        
        async function enviarFormulario(url, datos, metodo = 'POST') {
            const respuesta = await fetch(url, {
                method: metodo,
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(datos)
            });
            const resultado = await respuesta.json();
            alert(resultado.mensaje || resultado.error);
            cargarListas();
        }

        async function borrarRecurso(id, tipo) {
            const url = `${API_BASE}/${tipo}/${id}`;
            const respuesta = await fetch(url, { method: 'DELETE' });
            const resultado = await respuesta.json();
            alert(resultado.mensaje || resultado.error);
            cargarListas();
        }

        // Manejadores de los formularios
        document.getElementById('crear-doll-form').addEventListener('submit', async (e) => {
            e.preventDefault();
            const datos = {
                nombre: document.getElementById('doll-nombre').value,
                edad: parseInt(document.getElementById('doll-edad').value) || null,
                estado: document.getElementById('doll-estado').value
            };
            enviarFormulario(`${API_BASE}/dolls`, datos);
        });

        document.getElementById('crear-cliente-form').addEventListener('submit', async (e) => {
            e.preventDefault();
            const datos = {
                nombre: document.getElementById('cliente-nombre').value,
                ciudad: document.getElementById('cliente-ciudad').value
            };
            enviarFormulario(`${API_BASE}/clientes`, datos);
        });

        document.getElementById('crear-carta-form').addEventListener('submit', async (e) => {
            e.preventDefault();
            const datos = {
                cliente_id: parseInt(document.getElementById('carta-cliente-id').value),
                contenido: document.getElementById('carta-contenido').value
            };
            enviarFormulario(`${API_BASE}/cartas`, datos);
        });

        cargarListas();
    </script>
</body>
</html>
