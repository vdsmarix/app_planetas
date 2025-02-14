import 'package:flutter/material.dart';
import 'package:sqflite/sqflite.dart';
imporT 'package:path/path.dart';

void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Gerenciador de Planetas',
      theme: ThemeData(primarySwatch: Colors.blue),
      home: PlanetListScreen(),
    );
  }
}

class Planet {
  int? id;
  String name;
  double distance;
  double size;
  String? nickname;

  Planet({this.id, required this.name, required this.distance, required this.size, this.nickname});

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'name': name,
      'distance': distance,
      'size': size,
      'nickname': nickname,
    };
  }

  factory Planet.fromMap(Map<String, dynamic> map) {
    return Planet(
      id: map['id'],
      name: map['name'],
      distance: map['distance'],
      size: map['size'],
      nickname: map['nickname'],
    );
  }
}

class PlanetDatabase {
  static final PlanetDatabase instance = PlanetDatabase._init();
  static Database? _database;

  PlanetDatabase._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('planets.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);

    return await openDatabase(path, version: 1, onCreate: _createDB);
  }

  Future<void> _createDB(Database db, int version) async {
    await db.execute('''
      CREATE TABLE planets (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        distance REAL NOT NULL,
        size REAL NOT NULL,
        nickname TEXT
      )
    ''');
  }

  Future<int> insertPlanet(Planet planet) async {
    final db = await instance.database;
    return await db.insert('planets', planet.toMap());
  }

  Future<List<Planet>> getPlanets() async {
    final db = await instance.database;
    final result = await db.query('planets');
    return result.map((map) => Planet.fromMap(map)).toList();
  }

  Future<int> updatePlanet(Planet planet) async {
    final db = await instance.database;
    return await db.update('planets', planet.toMap(), where: 'id = ?', whereArgs: [planet.id]);
  }

  Future<int> deletePlanet(int id) async {
    final db = await instance.database;
    return await db.delete('planets', where: 'id = ?', whereArgs: [id]);
  }
}

class PlanetListScreen extends StatefulWidget {
  @override
  _PlanetListScreenState createState() => _PlanetListScreenState();
}

class _PlanetListScreenState extends State<PlanetListScreen> {
  late Future<List<Planet>> _planets;

  @override
  void initState() {
    super.initState();
    _refreshPlanets();
  }

  void _refreshPlanets() {
    setState(() {
      _planets = PlanetDatabase.instance.getPlanets();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Planetas')),
      body: FutureBuilder<List<Planet>>(
        future: _planets,
        builder: (context, snapshot) {
          if (!snapshot.hasData) return Center(child: CircularProgressIndicator());
          return ListView.builder(
            itemCount: snapshot.data!.length,
            itemBuilder: (context, index) {
              final planet = snapshot.data![index];
              return ListTile(
                title: Text(planet.name),
                subtitle: Text(planet.nickname ?? 'Sem apelido'),
                onTap: () => Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (context) => PlanetDetailScreen(planet: planet, refresh: _refreshPlanets),
                  ),
                ),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () => Navigator.push(
          context,
          MaterialPageRoute(
            builder: (context) => PlanetFormScreen(refresh: _refreshPlanets),
          ),
        ),
      ),
    );
  }
}

class PlanetFormScreen extends StatefulWidget {
  final VoidCallback refresh;
  PlanetFormScreen({required this.refresh});

  @override
  _PlanetFormScreenState createState() => _PlanetFormScreenState();
}

class _PlanetFormScreenState extends State<PlanetFormScreen> {
  final _formKey = GlobalKey<FormState>();
  String name = '';
  double distance = 0;
  double size = 0;
  String? nickname;

  void _savePlanet() async {
    if (_formKey.currentState!.validate()) {
      _formKey.currentState!.save();
      await PlanetDatabase.instance.insertPlanet(Planet(name: name, distance: distance, size: size, nickname: nickname));
      widget.refresh();
      Navigator.pop(context);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Adicionar Planeta')),
      body: Padding(
        padding: EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(
                decoration: InputDecoration(labelText: 'Nome'),
                validator: (value) => value!.isEmpty ? 'Campo obrigatório' : null,
                onSaved: (value) => name = value!,
              ),
              TextFormField(
                decoration: InputDecoration(labelText: 'Distância do Sol (UA)'),
                keyboardType: TextInputType.number,
                onSaved: (value) => distance = double.parse(value!),
              ),
              TextFormField(
                decoration: InputDecoration(labelText: 'Tamanho (km)'),
                keyboardType: TextInputType.number,
                onSaved: (value) => size = double.parse(value!),
              ),
              ElevatedButton(
                child: Text('Salvar'),
                onPressed: _savePlanet,
              ),
            ],
          ),
        ),
      ),
    );
  }
}
