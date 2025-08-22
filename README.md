# FFM-TOURNAMENT
# pubspec.yaml
name: ff_tourney
publish_to: 'none'
version: 1.0.0+1
environment:
  sdk: ">=3.2.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.6
  firebase_core: ^3.3.0
  firebase_auth: ^5.1.2
  cloud_firestore: ^5.2.1
  firebase_messaging: ^15.0.3
  razorpay_flutter: ^1.3.7
  intl: ^0.19.0

flutter:
  uses-material-design: true

# ------------------------------
# lib/main.dart
# ------------------------------
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'screens/auth_gate.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(); // Add platform options if not using google-services.json on iOS
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'FF Tourney',
      theme: ThemeData(
        brightness: Brightness.dark,
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF6C63FF),
          brightness: Brightness.dark,
        ),
        useMaterial3: true,
      ),
      home: const AuthGate(),
    );
  }
}

// ------------------------------
// lib/screens/auth_gate.dart
// ------------------------------
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';
import 'home_screen.dart';

class AuthGate extends StatefulWidget {
  const AuthGate({super.key});
  @override
  State<AuthGate> createState() => _AuthGateState();
}

class _AuthGateState extends State<AuthGate> {
  final _phone = TextEditingController();
  final _otp = TextEditingController();
  String? _verId;
  bool _sent = false;
  bool _busy = false;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Login – Phone OTP')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            TextField(
              controller: _phone,
              decoration: const InputDecoration(labelText: 'Phone (+91...)'),
              keyboardType: TextInputType.phone,
              enabled: !_sent,
            ),
            const SizedBox(height: 12),
            if (_sent)
              TextField(
                controller: _otp,
                decoration: const InputDecoration(labelText: 'Enter OTP'),
                keyboardType: TextInputType.number,
              ),
            const SizedBox(height: 16),
            SizedBox(
              width: double.infinity,
              child: ElevatedButton(
                onPressed: _busy ? null : () => _sent ? _verify() : _send(),
                child: Text(_sent ? 'Verify & Continue' : 'Send OTP'),
              ),
            ),
          ],
        ),
      ),
    );
  }

  Future<void> _send() async {
    setState(() => _busy = true);
    await FirebaseAuth.instance.verifyPhoneNumber(
      phoneNumber: _phone.text.trim(),
      verificationCompleted: (cred) async {
        await FirebaseAuth.instance.signInWithCredential(cred);
        if (!mounted) return;
        Navigator.pushReplacement(context, MaterialPageRoute(builder: (_) => const HomeScreen()));
      },
      verificationFailed: (e) {
        ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(e.message ?? 'Failed')));
      },
      codeSent: (id, _) {
        setState(() {
          _verId = id;
          _sent = true;
        });
      },
      codeAutoRetrievalTimeout: (id) => _verId = id,
    );
    setState(() => _busy = false);
  }

  Future<void> _verify() async {
    if (_verId == null) return;
    setState(() => _busy = true);
    final cred = PhoneAuthProvider.credential(verificationId: _verId!, smsCode: _otp.text.trim());
    await FirebaseAuth.instance.signInWithCredential(cred);
    if (!mounted) return;
    Navigator.pushReplacement(context, MaterialPageRoute(builder: (_) => const HomeScreen()));
    setState(() => _busy = false);
  }
}

// ------------------------------
// lib/screens/home_screen.dart
// ------------------------------
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';
import 'tournament_detail.dart';
import 'leaderboard_screen.dart';
import 'wallet_screen.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Tournaments'),
        actions: [
          IconButton(
            icon: const Icon(Icons.account_balance_wallet_outlined),
            onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (_) => const WalletScreen())),
          ),
          IconButton(
            icon: const Icon(Icons.emoji_events_outlined),
            onPressed: () => Navigator.push(context, MaterialPageRoute(builder: (_) => const LeaderboardScreen())),
          ),
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () async {
              await FirebaseAuth.instance.signOut();
            },
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: () async {
          // Seed one demo tournament (dev only)
          await FirebaseFirestore.instance.collection('tournaments').add({
            'title': 'Daily Squad – Bermuda',
            'mode': 'squad',
            'entry_fee': 10,
            'prize_pool': 500,
            'match_time': DateTime.now().add(const Duration(hours: 6)),
            'room_id': '',
            'room_pass': '',
            'status': 'upcoming',
          });
        },
        label: const Text('Seed Demo'),
        icon: const Icon(Icons.add),
      ),
      body: StreamBuilder<QuerySnapshot>(
        stream: FirebaseFirestore.instance
            .collection('tournaments')
            .orderBy('match_time')
            .snapshots(),
        builder: (context, snap) {
          if (!snap.hasData) {
            return const Center(child: CircularProgressIndicator());
          }
          final docs = snap.data!.docs;
          if (docs.isEmpty) return const Center(child: Text('No tournaments'));
          return ListView.separated(
            itemCount: docs.length,
            separatorBuilder: (_, __) => const Divider(height: 0),
            itemBuilder: (_, i) {
              final d = docs[i];
              return ListTile(
                title: Text(d['title']),
                subtitle: Text('Entry ₹${d['entry_fee']}  •  Prize ₹${d['prize_pool']}  •  ${d['mode']}'),
                trailing: const Icon(Icons.chevron_right),
                onTap: () => Navigator.push(
                  _,
                  MaterialPageRoute(builder: (_) => TournamentDetail(docId: d.id)),
                ),
              );
            },
          );
        },
      ),
    );
  }
}

// ------------------------------
// lib/screens/tournament_detail.dart
// ------------------------------
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';
import 'package:intl/intl.dart';
import 'join_success.dart';

class TournamentDetail extends StatelessWidget {
  final String docId;
  const TournamentDetail({super.key, required this.docId});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Tournament')),
      body: FutureBuilder<DocumentSnapshot>(
        future: FirebaseFirestore.instance.collection('tournaments').doc(docId).get(),
        builder: (context, snap) {
          if (!snap.hasData) return const Center(child: CircularProgressIndicator());
          final d = snap.data!;
          final dt = (d['match_time'] as Timestamp).toDate();
          return Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(d['title'], style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
                const SizedBox(height: 8),
                Text('Mode: ${d['mode']}'),
                Text('Entry: ₹${d['entry_fee']}'),
                Text('Prize: ₹${d['prize_pool']}'),
                Text('Time: ${DateFormat('dd MMM, hh:mm a').format(dt)}'),
                const Spacer(),
                SizedBox(
                  width: double.infinity,
                  child: ElevatedButton(
                    onPressed: () async {
                      final uid = FirebaseAuth.instance.currentUser!.uid;
                      // For MVP mobile-only: mark as joined without online payment.
                      // Later replace with Razorpay checkout screen.
                      await FirebaseFirestore.instance.collection('entries').add({
                        'tournament_id': docId,
                        'uid': uid,
                        'payment_status': 'success_dev',
                        'created_at': DateTime.now(),
                      });
                      if (context.mounted) {
                        Navigator.pushReplacement(
                          context,
                          MaterialPageRoute(builder: (_) => const JoinSuccess()),
                        );
                      }
                    },
                    child: const Text('Join (Dev Mode)'),
                  ),
                )
              ],
            ),
          );
        },
      ),
    );
  }
}

// ------------------------------
// lib/screens/join_success.dart
// ------------------------------
import 'package:flutter/material.dart';

class JoinSuccess extends StatelessWidget {
  const JoinSuccess({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.check_circle_outline, size: 96),
              const SizedBox(height: 16),
              const Text('You have joined!'),
              const SizedBox(height: 8),
              const Text('Room ID/Pass will be shown before match start.'),
              const SizedBox(height: 24),
              ElevatedButton(
                onPressed: () => Navigator.pop(context),
                child: const Text('Back'),
              )
            ],
          ),
        ),
      ),
    );
  }
}

// ------------------------------
// lib/screens/leaderboard_screen.dart
// ------------------------------
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutter/material.dart';

class LeaderboardScreen extends StatelessWidget {
  const LeaderboardScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Leaderboard (All)')),
      body: StreamBuilder<QuerySnapshot>(
        stream: FirebaseFirestore.instance.collection('results').orderBy('points_total', descending: true).snapshots(),
        builder: (context, snap) {
          if (!snap.hasData) return const Center(child: CircularProgressIndicator());
          final docs = snap.data!.docs;
          if (docs.isEmpty) return const Center(child: Text('No results yet'));
          return ListView.builder(
            itemCount: docs.length,
            itemBuilder: (_, i) {
              final d = docs[i];
              return ListTile(
                leading: CircleAvatar(child: Text('${i + 1}')),
                title: Text(d['name'] ?? d['uid_or_team'] ?? 'Player'),
                subtitle: Text('Kills: ${d['kills']}  •  Place: ${d['placement']}'),
                trailing: Text('${d['points_total']} pts'),
              );
            },
          );
        },
      ),
    );
  }
}

// ------------------------------
// lib/screens/wallet_screen.dart
// ------------------------------
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';

class WalletScreen extends StatelessWidget {
  const WalletScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final uid = FirebaseAuth.instance.currentUser!.uid;
    return Scaffold(
      appBar: AppBar(title: const Text('Wallet (Dev)')),
      body: StreamBuilder<DocumentSnapshot>(
        stream: FirebaseFirestore.instance.collection('users').doc(uid).snapshots(),
        builder: (context, snap) {
          final data = (snap.data?.data() as Map<String, dynamic>?) ?? {'wallet_balance': 0};
          final bal = (data['wallet_balance'] ?? 0).toDouble();
          return Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text('Balance: ₹${bal.toStringAsFixed(2)}', style: const TextStyle(fontSize: 22, fontWeight: FontWeight.bold)),
                const SizedBox(height: 16),
                Row(children: [
                  Expanded(
                    child: OutlinedButton(
                      onPressed: () async {
                        await FirebaseFirestore.instance.collection('users').doc(uid).set({
                          'wallet_balance': bal + 50,
                        }, SetOptions(merge: true));
                      },
                      child: const Text('Add ₹50 (Dev)'),
                    ),
                  ),
                  const SizedBox(width: 12),
                  Expanded(
                    child: ElevatedButton(
                      onPressed: () async {
                        if (bal >= 10) {
                          await FirebaseFirestore.instance.collection('users').doc(uid).set({
                            'wallet_balance': bal - 10,
                          }, SetOptions(merge: true));
                        }
                      },
                      child: const Text('Withdraw ₹10 (Dev)'),
                    ),
                  ),
                ])
              ],
            ),
          );
        },
      ),
    );
  }
}

// ------------------------------
// README (Quick Setup – Mobile Friendly)
// ------------------------------
/*
1) Firebase Console → Create project `ff_tourney`
2) Add Android app → package name e.g. com.example.ff_tourney → download `google-services.json` → put in android/app/
3) Firebase → Authentication → Enable Phone
4) Firebase → Firestore Database → Start in Test Mode (dev only)
5) Android: set minSdkVersion 23 and apply google-services plugin in Gradle.

Collections (create when app runs):
- users: { wallet_balance: number }
- tournaments: { title, mode, entry_fee, prize_pool, match_time(Timestamp), room_id, room_pass, status }
- entries: { tournament_id, uid, payment_status, created_at }
- results: { uid_or_team, kills, placement, points_total, name }

For mobile-only build without PC:
- Use a cloud builder like Codemagic (website) from your phone browser to build APK from this repo.
- Or import this source into FlutterFlow as a custom project (requires plan) and export APK.

Payments (later):
- Replace the Join button with Razorpay checkout (razorpay_flutter) and verify on server using webhook.
*/
