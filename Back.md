# Primer-proyecto-programacion-avanzada
Juan Pablo Moreno Guerrero y Jose Manuel Herrera Mora
#-------------------------------------------------------------------------------------------------#
from flask import Flask, request, jsonify, render_template
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import func
app = Flask(__name__)
# Configuración basedatos
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///site.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
# cosito de modelo de datos para clases
class Doll(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    nombre = db.Column(db.String(100), nullable=False)
    edad = db.Column(db.Integer, default=0)
    estado = db.Column(db.String(50), default='en servicio')
    cartas_escritas = db.Column(db.Integer, default=0)
class Cliente(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    nombre = db.Column(db.String(100), nullable=False)
    ciudad = db.Column(db.String(100))
    motivo_de_la_carta = db.Column(db.String(200))
    contacto = db.Column(db.String(100))
class Carta(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    cliente_id = db.Column(db.Integer, db.ForeignKey('cliente.id'), nullable=False)
    doll_asignada_id = db.Column(db.Integer, db.ForeignKey('doll.id'))
    estado = db.Column(db.String(50), default='nueva')
    contenido = db.Column(db.Text, nullable=False)
    resumen = db.Column(db.String(255))
# Rutas API para las ruticas
@app.route('/')
def home():
    """pag principal de la app."""
    return render_template('index.html')
@app.route('/dolls', methods=['GET', 'POST'])
def gestionar_dolls():
    """Creación y listado de Dolls."""
    if request.method == 'POST':
        datos = request.json
        if not datos or 'nombre' not in datos:
            return jsonify({"error": "Falta el nombre de la Doll."}), 400
        
        nueva_doll = Doll(nombre=datos['nombre'], edad=datos.get('edad', 0), estado=datos.get('estado', 'en servicio'))
        db.session.add(nueva_doll)
        db.session.commit()
        return jsonify({"mensaje": "Doll registrada correctamente", "id": nueva_doll.id}), 201
    #MUNECAS
    listado_dolls = Doll.query.all()
    lista_json = [{"id": d.id, "nombre": d.nombre, "edad": d.edad, "estado": d.estado, "cartas_escritas": d.cartas_escritas} for d in listado_dolls]
    return jsonify(lista_json)
@app.route('/dolls/<int:doll_id>', methods=['PUT', 'DELETE'])
def doll_detalles(doll_id):
    """Permite actualizar y borrar una Doll específica."""
    doll = Doll.query.get_or_404(doll_id)
    if request.method == 'PUT':
        datos = request.json
        if 'nombre' in datos:
            doll.nombre = datos['nombre']
        if 'estado' in datos:
            doll.estado = datos['estado']
        db.session.commit()
        return jsonify({"mensaje": "Doll actualizada con éxito."})
    if request.method == 'DELETE':
        db.session.delete(doll)
        db.session.commit()
        return jsonify({"mensaje": "Doll eliminada."}), 200
#CLIENTES
@app.route('/clientes', methods=['GET', 'POST'])
def gestionar_clientes():
    """Crea un nuevo cliente o lista todos los clientes."""
    if request.method == 'POST':
        datos = request.json
        if not datos or 'nombre' not in datos:
            return jsonify({"error": "El cliente debe tener un nombre."}), 400
        nuevo_cliente = Cliente(nombre=datos['nombre'], ciudad=datos.get('ciudad'))
        db.session.add(nuevo_cliente)
        db.session.commit()
        return jsonify({"mensaje": "Cliente registrado.", "id": nuevo_cliente.id}), 201
    listado_clientes = Cliente.query.all()
    lista_json = [{"id": c.id, "nombre": c.nombre, "ciudad": c.ciudad} for c in listado_clientes]
    return jsonify(lista_json)
@app.route('/clientes/<int:cliente_id>', methods=['PUT', 'DELETE'])
def cliente_detalles(cliente_id):
    """Actualiza o borra un cliente."""
    cliente = Cliente.query.get_or_404(cliente_id)
    if request.method == 'PUT':
        datos = request.json
        if 'nombre' in datos:
            cliente.nombre = datos['nombre']
        if 'ciudad' in datos:
            cliente.ciudad = datos['ciudad']
        db.session.commit()
        return jsonify({"mensaje": "Cliente modificado."})
    if request.method == 'DELETE':
        db.session.delete(cliente)
        db.session.commit()
        return jsonify({"mensaje": "Cliente borrado."}), 200
#CARTAS
@app.route('/cartas', methods=['GET', 'POST'])
def gestionar_cartas():
    """Maneja el registro de cartas y la asignación a Dolls."""
    if request.method == 'POST':
        datos = request.json
        if not datos or 'cliente_id' not in datos or 'contenido' not in datos:
            return jsonify({"error": "Faltan datos en la carta (ID de cliente o contenido).."}), 400
        #Asignar muneca
        doll_disponible = db.session.query(Doll, func.count(Carta.id)).outerjoin(
            Carta, Doll.id == Carta.doll_asignada_id
        ).filter(
            Doll.estado == 'en servicio',
            Carta.estado.in_(['nueva', 'en revisión'])
        ).group_by(Doll.id).order_by(func.count(Carta.id).asc()).first()
        if not doll_disponible:
            # Buscar activa
            doll_disponible = Doll.query.filter_by(estado='en servicio').first()
            if not doll_disponible:
                return jsonify({"error": "No hay Dolls disponibles en este momento."}), 400
        nueva_carta = Carta(
            cliente_id=datos['cliente_id'],
            doll_asignada_id=doll_disponible.Doll.id if hasattr(doll_disponible, 'Doll') else doll_disponible.id,
            contenido=datos['contenido']
        )
        db.session.add(nueva_carta)
        db.session.commit()
        return jsonify({"mensaje": "Carta creada y asignada a la Doll con ID " + str(nueva_carta.doll_asignada_id)}), 201
    listado_cartas = Carta.query.all()
    lista_json = [{"id": c.id, "cliente_id": c.cliente_id, "doll_asignada_id": c.doll_asignada_id, "estado": c.estado} for c in listado_cartas]
    return jsonify(lista_json)
@app.route('/cartas/<int:carta_id>', methods=['PUT', 'DELETE'])
def carta_detalles(carta_id):
    """Actualiza o borra una carta."""
    carta = Carta.query.get_or_404(carta_id)
    if request.method == 'PUT':
        datos = request.json
        if 'estado' in datos:
            if datos['estado'] == 'enviada' and carta.estado == 'revisada':
                doll = Doll.query.get(carta.doll_asignada_id)
                if doll:
                    doll.cartas_escritas += 1
                carta.estado = 'enviada'
                db.session.commit()
                return jsonify({"mensaje": "Carta marcada como 'enviada'."})
            carta.estado = datos['estado']
            db.session.commit()
            return jsonify({"mensaje": "Estado de la carta actualizado."})
    if request.method == 'DELETE':
        if carta.estado == 'nueva' or carta.estado == 'borrador':
            db.session.delete(carta)
            db.session.commit()
            return jsonify({"mensaje": "Carta borrada."}), 200
        return jsonify({"error": "Solo se pueden borrar cartas que no han sido asignadas o están en borrador."}), 403
#Repoertar
@app.route('/reporte/dolls', methods=['GET'])
def reporte_dolls():
    """Genera un reporte de cartas enviadas por cada Doll."""
    # reporte de las munecas...REVISAR
    reporte_raw = db.session.query(
        Doll.nombre,
        func.count(Carta.id),
        func.count(func.distinct(Carta.cliente_id))
    ).join(Carta).filter(Carta.estado == 'enviada').group_by(Doll.id).all()
    reporte_final = [{'doll': r[0], 'cartas_enviadas': r[1], 'clientes_distintos': r[2]} for r in reporte_raw]
    return jsonify(reporte_final)
#INICIAR
if __name__ == '__main__':
    with app.app_context():
        #BASEDATOS TABLAS
        db.create_all()
    app.run(debug=True)
