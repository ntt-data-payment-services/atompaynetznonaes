# Atom Paynetz Flutter
Atom Paynetz dart package for Flutter Integration.

## Getting Started
This is a helper class for Flutter Application to generate atom paynetz payment URL with Non AES Request.

## Installation
This plugin is available on Pub: https://pub.dev/packages/ndpsflutternonaes

Add this to dependencies in your app's pubspec.yaml
```sh
ndpsflutternonaes: ^1.0.1
webview_flutter: ^3.0.4
```
Note for Android: Make sure that the minimum API level for your app is 19 or higher.

## Usage
Follow below steps to create our payment demo

1) Mention below code inside the main.dart file
```sh
 import 'package:flutter/material.dart';
 import 'home.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Web Views',
      theme: ThemeData(
          primarySwatch: Colors.blue,
          fontFamily: "Arial",
          textTheme: const TextTheme(
              button: TextStyle(color: Colors.white, fontSize: 18.0),
              subtitle1: TextStyle(color: Colors.red))),
      home: const Home(),
    );
  }
}
```
2) Create home.dart file and mention below code
```sh
 // ignore: import_of_legacy_library_into_null_safe
import 'package:ndpsflutternonaes/ndpsflutternonaes.dart';
import 'package:flutter/material.dart';
import 'web_view_container.dart';

class Home extends StatelessWidget {
  // to test uat try below
  final login = "197"; //mandatory
  final pass = 'Test@123'; //mandatory
  final prodid = 'NSE'; //mandatory
  final requesthashKey = 'KEY123657234'; //mandatory
  final responsehashKey = 'KEYRESP123657234'; //mandatory

  final amt = '10.00'; //mandatory
  final username = 'test user'; //optional
  final mobile = '8888888888'; //optional
  final email = 'test@gmail.com'; //optional
  final address = 'mumbai'; //optional
  final date = '12/06/2022 16:50:00'; //mandatory //current transaction date should be matching like this
  final txnid = '123456'; //mandatory // this should be unique each time
  final custacc = '0'; //mandatory
  final clientcode = "NAVIN"; //mandatory
  final udf9 = "testdata1,testdata2,testdata3"; //optional, you can send comma seperated data here
  final mode = 'uat'; //mandatory // change to live for production

  const Home({Key? key}) : super(key: key);
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Atom Paynetz Sample App'),
      ),
      body: Center(
        child: Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => _handleURLButtonPress(context, responsehashKey),
              child: const Text('Open'),
            ),
          ],
        ),
      ),
    );
  }

  void _handleURLButtonPress(BuildContext context, String responsehashKey) {
    Navigator.push(
        context,
        MaterialPageRoute(
            builder: (context) => WebViewContainer(getUrl(), responsehashKey)));
  }

  getUrl() {
    var ndpsflutternonaes = NdpsFlutterNonAes(
        login: login,
        pass: pass,
        prodid: prodid,
        amt: amt,
        username: username,
        mobile: mobile,
        email: email,
        address: address,
        date: date,
        clientcode: clientcode,
        txnid: txnid,
        custacc: custacc,
        udf9: udf9,
        requesthashKey: requesthashKey,
        responsehashKey: responsehashKey,
        mode: mode);
    var urlToSend = ndpsflutternonaes.getUrl();
    return urlToSend;
  }
}

```

3) Create web_view_container.dart file and mention below code

// In this file we have created webview and getting the final response
```sh
// ignore: import_of_legacy_library_into_null_safe
import 'package:ndpsflutternonaes/ndpsflutternonaes.dart';
import 'package:flutter/material.dart';
// ignore: import_of_legacy_library_into_null_safe
import 'package:webview_flutter/webview_flutter.dart';

class WebViewContainer extends StatefulWidget {
  final url;
  final resHashKey;
  WebViewContainer(this.url, this.resHashKey);
  @override
  createState() => _WebViewContainerState(this.url, this.resHashKey);
}

class _WebViewContainerState extends State<WebViewContainer> {
  final _url;
  final _resHashKey;
  final _key = UniqueKey();
  late WebViewController _controller;
  _WebViewContainerState(this._url, this._resHashKey);
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        appBar: AppBar(
          backgroundColor: Colors.transparent,
          automaticallyImplyLeading: false,
          elevation: 0,
          toolbarHeight: 2,
        ),
        body: Column(
          children: [
            Expanded(
                child: WebView(
              key: _key,
              javascriptMode: JavascriptMode.unrestricted,
              onWebViewCreated: (controller) {
                _controller = controller;
              },
              initialUrl: _url,
              onPageFinished: (String url) async {
                if (url.contains('/mobilesdk/param')) {
                  // print('onPageFinished blocking navigation to $url}');
                  final response = await _controller.runJavascriptReturningResult(
                      "(function() { let htmlH5 = document.getElementsByTagName('h5')[0].innerHTML; return htmlH5; })();");

                  List responseArray = [];
                  var responseMap = {};
                  var splitRawResponse =
                      response.trim().split('|').map((text) => text).toList();
                  for (var i = 0; i < splitRawResponse.length; i++) {
                    if (splitRawResponse[i].isNotEmpty &&
                        splitRawResponse[i].length > 2) {
                      var splitNewResponse = splitRawResponse[i]
                          .trim()
                          .split('=')
                          .map((text) => text)
                          .toList();
                      responseArray.add(splitNewResponse);
                    }
                  }
                  for (var i = 0; i < responseArray.length; i++) {
                    responseMap[responseArray[i][0]] = responseArray[i][1];
                  }
                  var ndpsflutternonaes = VerifyTransaction();
                  var checkFinalTransaction = ndpsflutternonaes
                      .validateSignature(responseMap, _resHashKey);
                  var transactionResult = "";
                  if (checkFinalTransaction) {
                    if (responseMap['f_code'] == 'success_00' ||
                        responseMap['f_code'] == 'Ok') {
                      transactionResult = "success";
                    } else if (responseMap['f_code'] == 'C_06' ||
                        responseMap['f_code'] == 'C') {
                      transactionResult = "cancelled";
                    } else {
                      transactionResult = "failed";
                    }
                  } else {
                    // ignore: avoid_print
                    print("signature mismatched");
                    transactionResult = "failed";
                  }
                  // ignore: use_build_context_synchronously
                  Navigator.pop(context); // Close current window
                  // ignore: use_build_context_synchronously
                  ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                      content:
                          Text("Transaction Status = $transactionResult")));
                }
              },
              gestureNavigationEnabled: true,
            ))
          ],
        ));
  }
}
```
