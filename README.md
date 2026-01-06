# FastCar-Roraima-
name: transporte_local
description: App de transporte local tipo Uber (Pacaraima - Santa Elena)
environment:
  sdk: ">=3.3.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^3.3.0
  firebase_auth: ^5.2.0
  google_maps_flutter: ^2.5.0
  geolocator: ^11.0.0
  http: ^1.2.0

dev_dependencies:
  flutter_test:
    sdk: flutter
// main.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';

import 'login_screen.dart';
import 'home_screen.dart';

// Nota: configura tus opciones de Firebase (Android/iOS/Web) en firebase_options.dart si usas flutterfire.
// Aquí asumimos inicialización estándar con Firebase.initializeApp().

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(); // Asegúrate de tener google-services.json / GoogleService-Info.plist.
  runApp(const TransporteLocalApp());
}

class TransporteLocalApp extends StatelessWidget {
  const TransporteLocalApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Transporte Local',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      debugShowCheckedModeBanner: false,
      // Escucha el estado de autenticación para navegar
      home: StreamBuilder<User?>(
        stream: FirebaseAuth.instance.authStateChanges(),
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Scaffold(
              body: Center(child: CircularProgressIndicator()),
            );
          }
          if (snapshot.hasData) {
            return const HomeScreen();
          }
          return const LoginScreen();
        },
      ),
      // Rutas opcionales si deseas navegar por nombre
      routes: {
        '/login': (_) => const LoginScreen(),
        '/home': (_) => const HomeScreen(),
      },
    );
  }
}
// login_screen.dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';

class LoginScreen extends StatefulWidget {
  const LoginScreen({super.key});

  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _loading = false;
  String? _error;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  Future<void> _signIn() async {
    setState(() {
      _loading = true;
      _error = null;
    });
    try {
      await FirebaseAuth.instance.signInWithEmailAndPassword(
        email: _emailController.text.trim(),
        password: _passwordController.text,
      );
      // Al autenticarse, main.dart redirige automáticamente al HomeScreen.
    } on FirebaseAuthException catch (e) {
      setState(() => _error = e.message);
    } finally {
      if (mounted) setState(() => _loading = false);
    }
  }

  Future<void> _signUp() async {
    setState(() {
      _loading = true;
      _error = null;
    });
    try {
      await FirebaseAuth.instance.createUserWithEmailAndPassword(
        email: _emailController.text.trim(),
        password: _passwordController.text,
      );
    } on FirebaseAuthException catch (e) {
      setState(() => _error = e.message);
    } finally {
      if (mounted) setState(() => _loading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Iniciar sesión')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            TextField(
              controller: _emailController,
              keyboardType: TextInputType.emailAddress,
              decoration: const InputDecoration(labelText: 'Correo electrónico'),
            ),
            const SizedBox(height: 12),
            TextField(
              controller: _passwordController,
              obscureText: true,
              decoration: const InputDecoration(labelText: 'Contraseña'),
            ),
            const SizedBox(height: 24),
            if (_error != null)
              Text(_error!, style: const TextStyle(color: Colors.red)),
            const SizedBox(height: 12),
            Row(
              children: [
                Expanded(
                  child: ElevatedButton(
                    onPressed: _loading ? null : _signIn,
                    child: _loading
                        ? const SizedBox(
                            height: 20, width: 20, child: CircularProgressIndicator())
                        : const Text('Entrar'),
                  ),
                ),
              ],
            ),
            const SizedBox(height: 8),
            TextButton(
              onPressed: _loading ? null : _signUp,
              child: const Text('Crear cuenta'),
            ),
          ],
        ),
      ),
    );
  }
}
// home_screen.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:geolocator/geolocator.dart';
import 'package:firebase_auth/firebase_auth.dart';

import 'map_service.dart';

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final Completer<GoogleMapController> _mapController = Completer();
  final TextEditingController _destinationController = TextEditingController();

  // Coordenadas aproximadas de referencia
  static const LatLng pacaraima = LatLng(-4.476, -61.147); // Pacaraima, BR
  static const LatLng santaElena = LatLng(4.602, -61.105); // Santa Elena de Uairén (ejemplo: usar coordenadas reales)

  // Sugerencia: reemplaza santaElena por coordenadas reales: LatLng(4.602, -61.105) no es correcto en hemisferio;
  // Santa Elena ~ LatLng(4.602, -61.105) debe ser negativo para latitud sur si aplica. Ajusta con geocoder.
  // Para fines demostrativos, centramos cerca de la frontera BR-174:
  static const LatLng fronteraBR174 = LatLng(4.600, -61.110);

  Position? _currentPosition;
  Set<Marker> _markers = {};
  Set<Polyline> _polylines = {};
  bool _locating = true;

  @override
  void initState() {
    super.initState();
    _initLocation();
  }

  @override
  void dispose() {
    _destinationController.dispose();
    super.dispose();
  }

  Future<void> _initLocation() async {
    try {
      final permission = await Geolocator.requestPermission();
      if (permission == LocationPermission.denied ||
          permission == LocationPermission.deniedForever) {
        // Si no hay permisos, centramos en la frontera BR-174
        setState(() {
          _locating = false;
          _markers = {
            const Marker(
              markerId: MarkerId('frontera'),
              position: fronteraBR174,
              infoWindow: InfoWindow(title: 'BR-174 Frontera'),
            ),
          };
        });
        return;
      }
      final position = await Geolocator.getCurrentPosition(
        desiredAccuracy: LocationAccuracy.high,
      );
      setState(() {
        _currentPosition = position;
        _locating = false;
        _markers = {
          Marker(
            markerId: const MarkerId('yo'),
            position: LatLng(position.latitude, position.longitude),
            infoWindow: const InfoWindow(title: 'Mi ubicación'),
          ),
        };
      });
      final controller = await _mapController.future;
      await controller.animateCamera(
        CameraUpdate.newLatLngZoom(
          LatLng(position.latitude, position.longitude),
          14.5,
        ),
      );
    } catch (e) {
      // En caso de error, centramos en la frontera
      setState(() {
        _locating = false;
        _markers = {
          const Marker(
            markerId: MarkerId('frontera'),
            position: fronteraBR174,
            infoWindow: InfoWindow(title: 'BR-174 Frontera'),
          ),
        };
      });
    }
  }

  Future<void> _requestRide() async {
    final destText = _destinationController.text.trim();
    if (destText.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Ingresa un destino')),
      );
      return;
    }
    // Obtener coordenadas del destino (ej. "Santa Elena de Uairén, Plaza...").
    final destLatLng = await MapService().getPlaceLatLng(destText);
    if (destLatLng == null) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('No se pudo encontrar el destino')),
      );
      return;
    }

    // Añadimos el marcador del destino
    final originLatLng = _currentPosition != null
        ? LatLng(_currentPosition!.latitude, _currentPosition!.longitude)
        : fronteraBR174;

    setState(() {
      _markers = {
        Marker(
          markerId: const MarkerId('origen'),
          position: originLatLng,
          infoWindow: const InfoWindow(title: 'Origen'),
        ),
        Marker(
          markerId: const MarkerId('destino'),
          position: destLatLng,
          infoWindow: const InfoWindow(title: 'Destino'),
        ),
      };
      // Polyline simple (recta) como placeholder; para rutas reales usa Directions API.
      _polylines = {
        Polyline(
          polylineId: const PolylineId('ruta'),
          points: [originLatLng, destLatLng],
          width: 4,
          color: Colors.blue,
        )
      };
    });

    // Centrar la cámara para que se vean ambos puntos
    final controller = await _mapController.future;
    final bounds = _boundsFromLatLngList([originLatLng, destLatLng]);
    await controller.animateCamera(CameraUpdate.newLatLngBounds(bounds, 60));
  }

  LatLngBounds _boundsFromLatLngList(List<LatLng> list) {
    double? x0, x1, y0, y1;
    for (final latLng in list) {
      if (x0 == null) {
        x0 = x1 = latLng.latitude;
        y0 = y1 = latLng.longitude;
      } else {
        if (latLng.latitude > x1!) x1 = latLng.latitude;
        if (latLng.latitude < x0) x0 = latLng.latitude;
        if (latLng.longitude > y1!) y1 = latLng.longitude;
        if (latLng.longitude < y0!) y0 = latLng.longitude;
      }
    }
    return LatLngBounds(
      southwest: LatLng(x0!, y0!),
      northeast: LatLng(x1!, y1!),
    );
  }

  @override
  Widget build(BuildContext context) {
    final user = FirebaseAuth.instance.currentUser;
    return Scaffold(
      appBar: AppBar(
        title: const Text('Transporte Local'),
        actions: [
          if (user != null)
            IconButton(
              icon: const Icon(Icons.logout),
              onPressed: () => FirebaseAuth.instance.signOut(),
              tooltip: 'Cerrar sesión',
            ),
        ],
      ),
      body: Column(
        children: [
          // Campo de destino
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 12, 16, 8),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _destinationController,
                    decoration: const InputDecoration(
                      labelText: 'Destino (ej. Santa Elena, BR-174...)',
                      border: OutlineInputBorder(),
                    ),
                  ),
                ),
                const SizedBox(width: 8),
                ElevatedButton(
                  onPressed: _requestRide,
                  child: const Text('Solicitar'),
                ),
              ],
            ),
          ),
          // Mapa
          Expanded(
            child: Stack(
              children: [
                GoogleMap(
                  initialCameraPosition: const CameraPosition(
                    target: fronteraBR174, // Posición por defecto
                    zoom: 11,
                  ),
                  onMapCreated: (controller) => _mapController.complete(controller),
                  myLocationEnabled: !_locating,
                  myLocationButtonEnabled: true,
                  markers: _markers,
                  polylines: _polylines,
                  compassEnabled: true,
                  tiltGesturesEnabled: true,
                ),
                if (_locating)
                  const Positioned.fill(
                    child: Center(child: CircularProgressIndicator()),
                  ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

