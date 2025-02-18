import os
import shutil

# إنشاء مجلد المشروع وإضافة الملفات الأساسية
project_folder = "/mnt/data/BuracoGulfProject"
os.makedirs(project_folder, exist_ok=True)

# إنشاء ملف الكود الرئيسي main.dart
dart_code = """\
import 'package:flutter/material.dart';
import 'package:audioplayers/audioplayers.dart';
import 'dart:math';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_database/firebase_database.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(BuracoGulfApp());
}

class BuracoGulfApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Buraco Gulf',
      theme: ThemeData(
        primarySwatch: Colors.orange,
        scaffoldBackgroundColor: Colors.black,
        textTheme: TextTheme(bodyMedium: TextStyle(color: Colors.white)),
      ),
      home: GameScreen(),
    );
  }
}

class GameScreen extends StatefulWidget {
  @override
  _GameScreenState createState() => _GameScreenState();
}

class _GameScreenState extends State<GameScreen> {
  final DatabaseReference _gameRef = FirebaseDatabase.instance.ref('gameSession');
  final AudioPlayer _audioPlayer = AudioPlayer();
  List<String> deck = [];
  List<String> playerHand = [];
  List<String> aiHand = [];
  List<String> discardPile = [];
  bool isPlayerTurn = true;
  List<String> emojiReactions = ['😀', '😂', '😎', '🤔', '🔥', '🎉'];
  Color tableColor = Colors.green;

  @override
  void initState() {
    super.initState();
    initializeGame();
  }

  void initializeGame() async {
    DatabaseEvent event = await _gameRef.once();
    DataSnapshot snapshot = event.snapshot;
    if (!snapshot.exists) {
      initializeDeck();
      drawInitialCards();
      saveGameState();
    } else {
      loadGameState(snapshot);
    }
  }

  void initializeDeck() {
    List<String> suits = ['♥', '♦', '♣', '♠'];
    List<String> ranks = ['A', '2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K'];
    
    deck = [
      for (var suit in suits)
        for (var rank in ranks) '$rank$suit'
    ];
    deck.shuffle(Random());
  }

  void drawInitialCards() {
    for (int i = 0; i < 11; i++) {
      playerHand.add(deck.removeLast());
      aiHand.add(deck.removeLast());
    }
    discardPile.add(deck.removeLast());
  }

  void saveGameState() {
    _gameRef.set({
      'deck': deck,
      'playerHand': playerHand,
      'aiHand': aiHand,
      'discardPile': discardPile,
      'isPlayerTurn': isPlayerTurn,
    });
  }

  void loadGameState(DataSnapshot snapshot) {
    setState(() {
      deck = List<String>.from(snapshot.child('deck').value ?? []);
      playerHand = List<String>.from(snapshot.child('playerHand').value ?? []);
      aiHand = List<String>.from(snapshot.child('aiHand').value ?? []);
      discardPile = List<String>.from(snapshot.child('discardPile').value ?? []);
      isPlayerTurn = snapshot.child('isPlayerTurn').value as bool? ?? true;
    });
  }

  void drawCard() {
    if (deck.isNotEmpty) {
      setState(() {
        playerHand.add(deck.removeLast());
        isPlayerTurn = false;
      });
      _audioPlayer.play(Source.asset('assets/sounds/draw_card.mp3'));
      saveGameState();
      Future.delayed(Duration(seconds: 1), aiTurn);
    }
  }

  void discardCard(String card) {
    setState(() {
      playerHand.remove(card);
      discardPile.add(card);
      isPlayerTurn = false;
    });
    _audioPlayer.play(Source.asset('assets/sounds/discard_card.mp3'));
    saveGameState();
    Future.delayed(Duration(seconds: 1), aiTurn);
  }

  void aiTurn() {
    if (deck.isEmpty) return;
    setState(() {
      String drawnCard = deck.removeLast();
      aiHand.add(drawnCard);
      String discardedCard = aiHand.removeAt(Random().nextInt(aiHand.length));
      discardPile.add(discardedCard);
      isPlayerTurn = true;
    });
    saveGameState();
  }

  void sendEmojiReaction(String emoji) {
    _audioPlayer.play(Source.asset('assets/sounds/emoji_react.mp3'));
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('أرسل رمزًا: $emoji', style: TextStyle(fontSize: 20))),
    );
  }

  void changeTableColor(Color color) {
    setState(() {
      tableColor = color;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Buraco Gulf - Multiplayer')),
      body: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Container(
            height: 300,
            width: double.infinity,
            color: tableColor,
            child: Center(child: Text('طاولة اللعب', style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold, color: Colors.white))),
          ),
          SizedBox(height: 10),
          ElevatedButton(
            onPressed: () => changeTableColor(Colors.blueGrey),
            child: Text('تغيير لون الطاولة'),
          ),
          SizedBox(height: 20),
          Text('بطاقاتك:', style: TextStyle(fontSize: 22, fontWeight: FontWeight.bold, color: Colors.white)),
          SizedBox(height: 10),
          Wrap(
            children: playerHand
                .map((card) => GestureDetector(
                      onTap: () => discardCard(card),
                      child: Card(
                        color: Colors.deepOrangeAccent,
                        child: Padding(
                          padding: const EdgeInsets.all(8.0),
                          child: Text(card, style: TextStyle(fontSize: 20)),
                        ),
                      ),
                    ))
                .toList(),
          ),
        ],
      ),
    );
  }
}
"""

# حفظ الكود في ملف
with open(os.path.join(project_folder, "main.dart"), "w") as f:
    f.write(dart_code)

# ضغط المجلد
zip_file_path = "/mnt/data/BuracoGulfProject.zip"
shutil.make_archive(zip_file_path.replace(".zip", ""), 'zip', project_folder)

zip_file_path
