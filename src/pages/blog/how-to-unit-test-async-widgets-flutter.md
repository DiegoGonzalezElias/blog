---
layout: '../../layouts/Post.astro'
title: How to test async widgets on Flutter
image: /images/flutter-async-widget-testing
publishedAt: "2024-01-01"
category: 'Flutter'
---

## Introduction

First of all. I have to say that i am not an expert on dart or any other programming language, but i've been searching for about this topic and did not fount so much information at all, so... Here i am, giving my best to try to as clear as posible my opion about how to test a cusotm widget wich content depends on an async call ( like from an API ).

If you are reading this, you may have been struggling as me trying to test the data that comes from an external source inside a custom widget. If this is your case, then keep reading.

## Structure

Yes! structure, thats the key. If you are building the widget in a way that the api call that retrieves the necesary data to display inside of it, it's been called (as it looks obvious) from the mothod initState(). You are wrong (as well as i was), becouse in a StatefulWidget, you can not access to the class that holds that initState method or any data at all, becouse it is a private class and the data inside of it can not be accesible from the outside.

There my be some other ways to test this kind of widgets, but in my opinion this is the easiest way.

### **Services structure**

Let's start by imaging we have a repository that have a method that retrieves our data from an API to later feed our widget with that data.

```js
import 'package:dio/dio.dart';

class StarWarsRepository {
  final client = Dio();
  final String BASE_URL = 'https://swapi.dev/api/';

  Future<dynamic> getFilm(int number) async {
    final response = await client.get('${BASE_URL}films/$number');

    if (response.statusCode == 200) {
      return response.data;
    } else {
      throw Exception('Error getting film info');
    }
  }
}
```

In this case, we are getting info about one film of Star Wars.

### **Widget structure**

Inside my main.dart, I have made a StatefulWidget called testableWidget that it is been call from my root widget MyApp and only displays the title of the movie if there is data.

```js
import 'package:flutter/material.dart';
import 'package:testable_widget/services/api.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: TestableWidget(
        filmData: StarWarsRepository().getFilm(1),
      ),
    );
  }
}

class TestableWidget extends StatefulWidget {
  final Future<dynamic> filmData;
  const TestableWidget({super.key, required this.filmData});

  @override
  State<TestableWidget> createState() => _TestableWidgetState();
}

class _TestableWidgetState extends State<TestableWidget> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('TestableWidget'),
      ),
      body: FutureBuilder(
          future: widget.filmData,
          builder: (context, snapshot) {
            if (snapshot.hasData) {
              final film = snapshot.data;

              return Center(child: Text(film['title'].toString()));
            } else if (snapshot.hasError) {
              return const Center(child: Text('Error obtaining film data.'));
            }
            return const Center(child: CircularProgressIndicator());
          }),
    );
  }
}
```

The thing here is that we are using the widget FutureBuilder, that let's you create a widget that builds itself based on the latest snapshot of interaction with a [Future]. So we are passing as prop the method from the repository that calls the API.

```js
 home: TestableWidget(
        filmData: StarWarsRepository().getFilm(1),
      ),
```

Building the widget by this way, the logic reminds outside the widget and it is testable as you can find out trying to run this tests.

## Tests

```js
 import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

import 'package:testable_widget/main.dart';

void main() {
  group('TestableWidget', () {
    final filmData = {
      "title": "The Empire Strikes Back",
      "episode_id": 5,
      "opening_crawl":
          "It is a dark time for the\r\nRebellion. Although the Death\r\nStar has been destroyed,\r\nImperial troops have driven the\r\nRebel forces from their hidden\r\nbase and pursued them across\r\nthe galaxy.\r\n\r\nEvading the dreaded Imperial\r\nStarfleet, a group of freedom\r\nfighters led by Luke Skywalker\r\nhas established a new secret\r\nbase on the remote ice world\r\nof Hoth.\r\n\r\nThe evil lord Darth Vader,\r\nobsessed with finding young\r\nSkywalker, has dispatched\r\nthousands of remote probes into\r\nthe far reaches of space....",
      "director": "Irvin Kershner",
      "producer": "Gary Kurtz, Rick McCallum",
      "release_date": "1980-05-17",
      "characters": [],
      "planets": [],
      "starships": [],
      "vehicles": [],
      "species": [],
      "created": "2014-12-12T11:26:24.656000Z",
      "edited": "2014-12-15T13:07:53.386000Z",
      "url": "https://swapi.dev/api/films/2/"
    };

    Future<dynamic> mockGetFilmData() {
      return Future.delayed(const Duration(seconds: 1), () => filmData);
    }

    testWidgets('should display circularProgressIndicator',
        (WidgetTester tester) async {
      await tester.pumpWidget(MaterialApp(
        home: TestableWidget(
          filmData: mockGetFilmData(),
        ),
      ));

      expect(find.byType(CircularProgressIndicator), findsOneWidget);
      await tester.pumpAndSettle();
    });

    testWidgets('should display film title', (WidgetTester tester) async {
      await tester.pumpWidget(MaterialApp(
        home: TestableWidget(
          filmData: mockGetFilmData(),
        ),
      ));

      await tester.pumpAndSettle();
      expect(find.text('The Empire Strikes Back'), findsOneWidget);
    });

    testWidgets('should display error message', (WidgetTester tester) async {
      Future<dynamic> mockErrorFilmData() {
        return Future.delayed(const Duration(seconds: 1),
            () => throw Exception('Error getting film info'));
      }

      await tester.pumpWidget(MaterialApp(
        home: TestableWidget(
          filmData: mockErrorFilmData(),
        ),
      ));

      await tester.pumpAndSettle();
      expect(find.text('Error obtaining film data.'), findsOneWidget);
    });
  });
}

```