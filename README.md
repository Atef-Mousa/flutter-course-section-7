# MyAnime — Flutter Course, Section 7: Networking with Dio

A small Flutter app that loads an anime list from the public [Kitsu API](https://kitsu.docs.apiary.io/) and shows it in a grid with infinite scrolling.

Use this README as your **study guide** for the networking part of the course. Each concept below points to the file in the project where you can see it in action.

> **Where this section fits in the course**
> 1. **Basic networking with Dio** ← *you are here*
> 2. State management with Riverpod
> 3. Interceptors, authentication (JWT) and secure storage

---

## Table of contents

1. [Run the project](#1-run-the-project)
2. [Project structure](#2-project-structure)
3. [How an HTTP request works](#3-how-an-http-request-works)
4. [The API we use: Kitsu](#4-the-api-we-use-kitsu)
5. [Setting up Dio: `BaseOptions`](#5-setting-up-dio-baseoptions)
6. [Sending a GET request](#6-sending-a-get-request)
7. [From JSON to a Dart model](#7-from-json-to-a-dart-model)
8. [Error handling](#8-error-handling)
9. [Showing the data in the UI](#9-showing-the-data-in-the-ui)
10. [Pagination (infinite scroll)](#10-pagination-infinite-scroll)
11. [The full flow, step by step](#11-the-full-flow-step-by-step)
12. [Known limitations](#12-known-limitations-we-will-fix-these-later)
13. [Exercises](#13-exercises)
14. [Quick review questions](#14-quick-review-questions)

---

## 1. Run the project

```bash
flutter pub get
flutter run
```

Packages used (`pubspec.yaml`):

| Package | Why we use it |
|---|---|
| [`dio`](https://pub.dev/packages/dio) | HTTP client: sends requests and receives responses |
| [`go_router`](https://pub.dev/packages/go_router) | Navigation between pages |
| [`shared_preferences`](https://pub.dev/packages/shared_preferences) | Simple local storage (used in later lessons) |

---

## 2. Project structure

```
lib/
├── main.dart            → App entry point (MaterialApp.router)
├── router.dart          → Routes: /home and /fav
├── anime.dart           → MODEL: the Anime class + fromJson
├── anime_service.dart   → NETWORK LAYER: Dio setup + API calls
├── api_exception.dart   → ERRORS: turns Dio errors into readable messages
├── home_page.dart       → UI: loads data, shows loading/error/grid, pagination
├── anime_card.dart      → UI: one anime card (poster, rating, name, description)
└── fav_page.dart        → Favorites page (placeholder for now)
```

Each file has **one job**:

```
UI (home_page)  ──calls──▶  Service (anime_service)  ──uses──▶  Dio  ──HTTP──▶  Kitsu API
      ▲                              │
      │                              ├── converts JSON  ──▶  Anime model (anime.dart)
      └──── List<Anime> or ──────────┘
            ApiException (api_exception.dart)
```

The UI never sees Dio or raw JSON. It only gets a `List<Anime>`, or an `ApiException` with a message it can show.

---

## 3. How an HTTP request works

Your app (the **client**) sends a **request** to a **server**, and the server sends back a **response**.

**A request contains:**

| Part | Example in our app |
|---|---|
| Method | `GET` (read data). Others: `POST` create, `PUT`/`PATCH` update, `DELETE` remove |
| URL | `https://kitsu.app/api/edge/anime` |
| Query parameters | `?page[offset]=10` |
| Headers | `Accept: application/vnd.api+json` |
| Body | none for GET (used with POST/PUT) |

**A response contains:**

| Part | Example |
|---|---|
| Status code | `200` OK, `404` Not Found, `500` Server Error |
| Headers | content type, etc. |
| Body | JSON data |

**Status code families:**

| Range | Meaning |
|---|---|
| `2xx` | Success (`200` OK, `201` Created) |
| `3xx` | Redirect |
| `4xx` | **Client** error: something is wrong with *our* request (`400`, `401`, `403`, `404`, `429`) |
| `5xx` | **Server** error: something broke on the server (`500`, `502`, `503`) |

---

## 4. The API we use: Kitsu

Endpoint: `GET https://kitsu.app/api/edge/anime`

Try it in your browser or in Postman first. **Always look at the real response before writing the model.**

A shortened response looks like this:

```json
{
  "data": [
    {
      "id": "1",
      "type": "anime",
      "attributes": {
        "slug": "cowboy-bebop",
        "description": "In the year 2071, humanity has colonized...",
        "averageRating": "82.13",
        "posterImage": {
          "tiny": "https://media.kitsu.app/.../tiny.jpg",
          "medium": "https://media.kitsu.app/.../medium.jpg"
        }
      }
    }
  ],
  "links": {
    "first": "...",
    "next": "...",
    "last": "..."
  }
}
```

Notes:
- The list is inside `"data"`.
- Each item's fields are inside `"attributes"`, nested one level deeper.
- Kitsu returns **10 items per page** by default. We move between pages with `page[offset]`.
- `averageRating` is a **String** (`"82.13"`), not a number. Always check the types!

---

## 5. Setting up Dio: `BaseOptions`

📄 `lib/anime_service.dart`

```dart
static final _dio = Dio(
  BaseOptions(
    baseUrl: "https://kitsu.app/api/edge",
    connectTimeout: const Duration(seconds: 10),
    receiveTimeout: const Duration(seconds: 15),
    sendTimeout: const Duration(seconds: 10),
    headers: {
      "Accept": "application/vnd.api+json",
      "Content-Type": "application/vnd.api+json",
    },
  ),
);
```

We create **one** Dio instance and reuse it for every request. `BaseOptions` are the defaults that every request made with this instance gets.

### `baseUrl`
The common start of every URL. After that we only write the path:
```dart
_dio.get("/anime")   // → https://kitsu.app/api/edge/anime
```
If the server address changes, you only change it in one place.

### Timeouts — why we need them
Without a timeout, a request on a bad network can wait **forever**, and the user sees an endless loading screen.

| Timeout | What it limits | Example situation |
|---|---|---|
| `connectTimeout` | Time to **open the connection** to the server | Server is down or unreachable |
| `sendTimeout` | Time to **upload** the request body | Uploading a big image on slow internet (POST/PUT) |
| `receiveTimeout` | Time to **receive** the response | Server is very slow to answer, or the response is huge |

When a timeout is reached, Dio throws a `DioException` with the type `connectionTimeout`, `sendTimeout` or `receiveTimeout`. See [Error handling](#8-error-handling).

### Headers
Extra information sent with every request:
- `Accept` → the response format we want back. Kitsu uses the **JSON:API** format, `application/vnd.api+json`.
- `Content-Type` → the format of the body *we* send.

> Later, with interceptors, we will add the `Authorization: Bearer <token>` header here.

---

## 6. Sending a GET request

📄 `lib/anime_service.dart`

```dart
static Future<List<Anime>> getAllData({int offset = 0}) async {
  try {
    final response = await _dio.get(
      "/anime",
      queryParameters: {"page[offset]": offset},
    );
    final data = response.data["data"] as List;
    final List<Anime> animes = [];
    for (var item in data) {
      animes.add(Anime.fromJson(item));
    }
    return animes;
  } on DioException catch (e) {
    throw ApiException.fromDioException(e);
  } catch (e) {
    throw const ApiException("Failed to read data from the server.");
  }
}
```

Key ideas:

- **`Future` + `async` / `await`.** A network call takes time. A `Future` is a value that *will* be ready later. `await` waits for it **without freezing the UI**.
- **`queryParameters`.** Dio builds the query string for you, safely encoded. `{"page[offset]": 10}` becomes `/anime?page%5Boffset%5D=10`. Don't concatenate strings into the URL yourself.
- **`response.data`.** Dio **decodes the JSON automatically**, so `response.data` is already a `Map<String, dynamic>` and you don't need `jsonDecode`.
- **Other `response` fields:** `response.statusCode`, `response.headers`, `response.requestOptions`.
- **Dio throws for bad status codes.** By default Dio treats any status outside `2xx` as an error and throws a `DioException`. You don't have to check `statusCode == 200` yourself.

Other methods follow the same pattern (we will use them later):

```dart
_dio.post("/path", data: {...});     // create
_dio.put("/path/1", data: {...});    // replace
_dio.patch("/path/1", data: {...});  // update some fields
_dio.delete("/path/1");              // remove
```

---

## 7. From JSON to a Dart model

📄 `lib/anime.dart`

```dart
factory Anime.fromJson(Map<String, dynamic> data) {
  final id = data["id"];
  final name = data["attributes"]["slug"];
  final desc = data["attributes"]["description"];
  final image = data["attributes"]["posterImage"]["medium"];
  final rating = data["attributes"]["averageRating"];

  return Anime(id: id, name: name, desc: desc, image: image, rating: rating);
}
```

**Why use a model instead of working with the Map directly?**
- ✅ Type safety: `anime.name` instead of `item["attributes"]["slug"]` all over the UI.
- ✅ Autocomplete in the editor, and fewer typos.
- ✅ If the API changes, you fix **one** place (`fromJson`), not every widget.

**`factory` constructor:** a constructor that runs logic (here, reading the JSON) before creating the object.

The flow:
```
JSON (String) ──Dio──▶ Map<String, dynamic> ──fromJson──▶ Anime ──▶ List<Anime>
```

> ⚠️ If the API sends `null` for a field (some anime have no rating), `fromJson` fails. See [Known limitations](#12-known-limitations-we-will-fix-these-later).

---

## 8. Error handling

Networking **will** fail sometimes: no internet, slow server, wrong URL, server crash. A production app must:
1. **Not crash**,
2. **Tell the user** what happened in plain language,
3. **Let the user retry**.

### 8.1 `DioException` and its types

Whenever something goes wrong, Dio throws a `DioException`. Its `type` tells you *why*:

| `DioExceptionType` | When it happens |
|---|---|
| `connectionTimeout` | Couldn't connect within `connectTimeout` |
| `sendTimeout` | Couldn't upload the body within `sendTimeout` |
| `receiveTimeout` | Response didn't arrive within `receiveTimeout` |
| `transformTimeout` | Processing the response took too long |
| `badResponse` | Server answered with an error status (4xx / 5xx). Check `e.response?.statusCode` |
| `connectionError` | No internet, DNS failure, server unreachable |
| `badCertificate` | The HTTPS certificate is invalid |
| `cancel` | The request was cancelled with a `CancelToken` |
| `unknown` | Anything else |

### 8.2 Our own exception: `ApiException`

📄 `lib/api_exception.dart`

We don't want the UI to know about Dio. So we **translate** every `DioException` into our own `ApiException`, which carries a clear `message`:

```dart
factory ApiException.fromDioException(DioException e) {
  switch (e.type) {
    case DioExceptionType.connectionTimeout:
      return const ApiException("Connection timed out. Please try again.");
    case DioExceptionType.connectionError:
      return const ApiException("No internet connection.");
    case DioExceptionType.badResponse:
      return ApiException._fromStatusCode(e.response?.statusCode);
    // ... other cases
  }
}
```

And for `badResponse`, we decide the message by status code:

| Code | Message |
|---|---|
| 400 | Bad request. |
| 401 | Unauthorized. Please log in again. |
| 403 | You don't have permission to do this. |
| 404 | Requested data was not found. |
| 429 | Too many requests. Slow down a little. |
| 500 / 502 / 503 | Server error. Please try again later. |

**Why do this?**
- The UI only catches **one** type: `ApiException`.
- All error messages live in **one file**.
- If we replace Dio with another package someday, the UI doesn't change.

### 8.3 Catch at the right layer

```dart
// Service layer: converts errors
} on DioException catch (e) {
  throw ApiException.fromDioException(e);   // network/HTTP problems
} catch (e) {
  throw const ApiException("Failed to read data from the server.");  // e.g. bad JSON shape
}
```

```dart
// UI layer: shows errors
try {
  final data = await AnimeService.getAllData(offset: offset);
  ...
} on ApiException catch (e) {
  setState(() => value = e.message);
}
```

> ❌ **Avoid** `catch (e) { rethrow; }`. It does nothing useful and still crashes the UI.
> ✅ **Do** convert the error into something meaningful, then handle it where you can show it to the user.

---

## 9. Showing the data in the UI

📄 `lib/home_page.dart`

### 9.1 Loading data when the page opens
```dart
@override
void initState() {
  super.initState();
  _getData();
}
```
`initState` runs **once** when the widget is created, which makes it the right place to start the first request.

### 9.2 The three UI states
Every screen that loads data has states. In this project we use a `value` string:

| State | `value` | What the user sees |
|---|---|---|
| Loading | `"Loading ..."` | Loading message |
| Error | `e.message` | Error message + **Retry** button |
| Success | `"Data Loaded "` | The grid of anime |

```dart
if (animes.isEmpty) {
  return Scaffold(
    body: Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(value, textAlign: TextAlign.center),
          if (value != "Loading ...")
            ElevatedButton(onPressed: _getData, child: const Text("Retry")),
        ],
      ),
    ),
  );
}
// otherwise → GridView with the anime cards
```

> In the Riverpod section, we will replace this `String` with proper states (`AsyncValue`: loading / error / data).

### 9.3 `mounted`: is the page still on screen?
```dart
final data = await AnimeService.getAllData(offset: offset);
if (!mounted) return;   // the user may have left the page while we waited
setState(() { ... });
```
A request can take seconds. If the user navigates away in the meantime, calling `setState` on a removed widget throws an error. **Always check `mounted` after an `await` before calling `setState`.**

### 9.4 Loading images from the network
📄 `lib/anime_card.dart`
```dart
Image.network(
  anime.image,
  errorBuilder: (context, object, trace) => Container(color: Colors.red),
)
```
`errorBuilder` shows a fallback widget when an image fails to load, instead of breaking the card.

---

## 10. Pagination (infinite scroll)

APIs don't send thousands of items at once. They send **pages**. Kitsu uses **offset pagination**:

| Request | Items returned |
|---|---|
| `page[offset]=0` | 1 – 10 |
| `page[offset]=10` | 11 – 20 |
| `page[offset]=20` | 21 – 30 |

📄 `lib/home_page.dart`
```dart
_scrollController.addListener(() {
  if (_scrollController.position.pixels + 200 >
      _scrollController.position.maxScrollExtent) {
    offset += 10;
    _getData();
  }
});
```

- `pixels` → how far the user has scrolled.
- `maxScrollExtent` → the end of the list.
- `+ 200` → start loading **before** the user hits the bottom, so scrolling feels smooth.
- New items are **added** to the list (`animes.addAll(data)`), not replaced.

### Dispose your controllers
```dart
@override
void dispose() {
  _scrollController.dispose();
  super.dispose();
}
```
Controllers hold listeners and resources. If you don't dispose them, they stay in memory after the page is gone (a **memory leak**).
**Rule:** if you create a controller in a `State`, dispose it in `dispose()`.

---

## 11. The full flow, step by step

```
1. HomePage.initState()           → calls _getData()
2. _getData()                     → value = "Loading ..."  → setState
3. AnimeService.getAllData(0)     → _dio.get("/anime?page[offset]=0")
4. Dio                            → sends request with baseUrl + headers + timeouts
5. Kitsu server                   → responds 200 + JSON
6. Dio                            → decodes JSON → response.data (Map)
7. Service                        → data["data"] → each item → Anime.fromJson → List<Anime>
8. HomePage                       → checks mounted → animes.addAll → setState
9. build()                        → list not empty → GridView of AnimeCards
10. User scrolls near bottom      → offset += 10 → back to step 2

❌ If anything fails in step 3–7:
   DioException → ApiException.fromDioException → caught in HomePage
   → value = e.message → user sees the message + Retry button
```

---

## 12. Known limitations (we will fix these later)

This project is intentionally simple. These are on purpose, and each one is a topic in a later lesson:

| Limitation | Fixed in |
|---|---|
| The scroll listener can send several requests at the same time (no `isLoading` guard) and never knows when the last page is reached | Riverpod section |
| `value` is a plain `String` for state; an error while loading page 2+ isn't shown | Riverpod section (`AsyncValue`) |
| `AnimeService` is `static`, which makes it hard to test or swap | Riverpod section (providers / dependency injection) |
| `fromJson` fails if the API sends `null` (e.g. a missing `averageRating`) | Exercise below |
| No logging, no auth token, no retry | Interceptors + JWT section |
| The release `android/app/src/main/AndroidManifest.xml` has no `INTERNET` permission | Exercise below |

---

## 13. Exercises

1. **Null safety in `fromJson`.** Make the model safe when `description`, `averageRating` or `posterImage` is `null`. *Hint:* `data["attributes"]["averageRating"] ?? "N/A"` and `?["medium"]`.
2. **Show the real name.** Use `attributes.canonicalTitle` instead of `slug`.
3. **Test the timeouts.** Set `connectTimeout` to `Duration(milliseconds: 1)` and run the app. Which message appears? Then turn off your internet. Which message appears now?
4. **Test a 404.** Change the path to `/animee`. Which `DioExceptionType` and status code do you get?
5. **Page size.** Add `"page[limit]": 20` to `queryParameters`. What else must change in the pagination code?
6. **Stop double requests.** Add a `bool _isLoading` flag so `_getData()` does nothing while a request is already running.
7. **Release permission.** Add `<uses-permission android:name="android.permission.INTERNET"/>` to `android/app/src/main/AndroidManifest.xml`, build a release APK (`flutter build apk`), and explain why it didn't work without it.
8. **Anime details.** Create a function `getAnimeById(String id)` that calls `GET /anime/{id}` and returns one `Anime`.

---

## 14. Quick review questions

1. What is the difference between `baseUrl` and the path passed to `_dio.get()`?
2. Explain `connectTimeout`, `receiveTimeout` and `sendTimeout` with a real-life example for each.
3. Why don't we call `jsonDecode` when using Dio?
4. Does Dio throw an error for a `404` response? For a `200`?
5. Why do we convert `DioException` into `ApiException` instead of catching `DioException` in the UI?
6. What's wrong with `catch (e) { rethrow; }` if nobody catches it higher up?
7. Why do we check `mounted` after `await`?
8. What happens if we forget to call `_scrollController.dispose()`?
9. Why use `queryParameters` instead of building the URL with string concatenation?
10. What is the difference between 4xx and 5xx status codes? Whose "fault" is each?

---

### Useful links
- Dio package: https://pub.dev/packages/dio
- Kitsu API docs: https://kitsu.docs.apiary.io/
- HTTP status codes (MDN): https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
- Flutter networking cookbook: https://docs.flutter.dev/cookbook/networking/fetch-data
