---
title:  "Flutter - Rest API!"

read_time: false
share: false
author_profile: false
classes: wide

categories:
  - flutter

tags:
  - flutter
  - cookbook


toc: true

---

# Rest API

클라이언트가 특정 서비스로부터 HTTP 프로토콜을 통해 json 형식의 데이터를 가져오는 너무나도 당연한 형식의 코드를 Flutter 로 작성해보자.  

> https://pub.dev/packages/http  
> https://pub.dev/packages/mockito  


`mockito` 의 경우 `dev_dependencies`에서 추가한다.  

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito: ^4.1.1
```

`http` 패키지를 사용해 `response`를 가져오는 **테스트** 진행  

```dart
void main() {
  test("http 통신 테스트", () async {
    var response = await http.get(
        "https://api.airvisual.com/v2/nearest_city?key={{mykey}}");
    expect(response.statusCode, 200);
  });
}
```

`Http status` 가 `200`이 출력되는지 확인  

## json to Dart

이제 `response.body` 의 `json` 데이터를 `Dart`에서 사용할 수 있도록하면 된다.  

`DTO` 같은 `json` 매핑용 클래스를 미리 정의해 두고 json 데이터를 받아 초기화 해주어야 한다.  

> https://javiercbk.github.io/json_to_dart/

위 사이트에서 `json` 형식을 `Dart` 클래스로 변환시켜준다.  

```dart 
AirResult result = AirResult.fromJson(json.decode(response.body));
expect(result.status, "success");
```

`json.decode` 를 사용하려면 아래 패키지를 `import`.

```dart
import 'dart:convert';
```

`json` 데이터가 `AirResult` 라는 `Dart` 클래스로 변환되고 `status` 변수에 `"success"` 문자열이 저장되어 있는지 확인.  


# bloc 패턴 

먼저 Stream, RxDart 에 대해 알아야한다.  

## Stream, RxDart

```dart
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Retrieve Text Input',
      home: Scaffold(appBar: AppBar(title: Text("카운터")), body: Counter()),
    );
  }
}

class Counter extends StatefulWidget {
  @override
  _CounterState createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _counter = 0;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: <Widget>[
          RaisedButton(
            child: Text("Add"),
            onPressed: () {
              setState(() {
                _counter++;
              });
            },
          ),
          Text("${_counter}", style: TextStyle(fontSize: 30))
        ],
      ),
    );
  }
}
```

SetState 말고 Stream을 사용해 int 값 증가  

RxDart를 사용해 Stream을 구현하자.  

> https://pub.dev/packages/rxdart

`BehaviorSubject`

```dart
class _CounterState extends State<Counter> {
  final countSubjectg = BehaviorSubject<int>();
  int conut = 0;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: <Widget>[
          RaisedButton(
            child: Text("Add"),
            onPressed: () {
              countSubjectg.add(++conut);
            },
          ),
          StreamBuilder<int>(
              stream: countSubjectg.stream,
              initialData: 0,
              builder: (context, snapshot) {
                if (snapshot.hasData) {
                  return Text("${snapshot.data}", style: TextStyle(fontSize: 30));
                }
              })
        ],
      ),
    );
  }
}
```

## bloc패턴 사용하기  

그냥 setState 쓰면 되긴 하지만 bloc을 쓰면 좋은 이유  

UI test, Unit Test 분리하기 쉽다.  

비지니스 로직, 필요한 변수를 모두 Bloc 애 저장 해서 사용한다.  


우선 위에서 지정했던 conut 관련된 stream, BehaviorSubject 객체, count변수 모두 삭제  

이제 count 값과관련된 모든 로직은 bloc 객체로 뺴

```dart
import 'package:flutter/material.dart';
import 'package:rxdart/rxdart.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Retrieve Text Input',
      home: Scaffold(appBar: AppBar(title: Text("카운터")), body: Counter()),
    );
  }
}

class Counter extends StatefulWidget {
  @override
  _CounterState createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: <Widget>[
          RaisedButton(
            child: Text("Add"),
            onPressed: () {
              //TODO count ++
            },
          ),
          Text("0", style: TextStyle(fontSize: 30))
        ],
      ),
    );
  }
}
```

그리고 bloc 으로 사용할 CounterBloc 정의, RxDart 패키지도 추가  

```dart
import 'package:rxdart/rxdart.dart';

class CounterBloc {
  int _count = 0;
  final _countSubject = BehaviorSubject.seeded(0); //초기값은 0

  void addCount() {
    _count ++;
    _countSubject.add(_count);
  }
  
  Stream<int> get count$ => _countSubject.stream;
}
```


```dart 
import 'package:flutter/material.dart';
import 'package:flutter_basic/bloc/counter_bloc.dart';

void main() => runApp(MyApp());

final countbloc = CounterBloc();

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Retrieve Text Input',
      home: Scaffold(appBar: AppBar(title: Text("카운터")), body: Counter()),
    );
  }
}

class Counter extends StatefulWidget {
  @override
  _CounterState createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: <Widget>[
          RaisedButton(
            child: Text("Add"),
            onPressed: () {
              countbloc.addCount();
            },
          ),
          StreamBuilder<int>(
              stream: countbloc.count$,
              initialData: 0,
              builder: (context, snapshot) {
                if (snapshot.hasData) {
                  return Text("${snapshot.data}",
                      style: TextStyle(fontSize: 30));
                }
              })
        ],
      ),
    );
  }
}
```


## flutter_bloc

