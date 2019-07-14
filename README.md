![Laravel - Najlepsze Praktyki](/images/logo-english.png?raw=true)

[Oryginał w języku angielskim](https://github.com/alexeymezenin/laravel-best-practices) (autorstwa [alexeymezenin](https://github.com/alexeymezenin))

Niniejszy dokument nie stanowi adaptacji zasad SOLID lub jakichkolwiek wzorców. Znajdziesz tutaj najlepsze praktyki, które zwykle są nieświadomie pomijane podczas tworzenia aplikacji z wykorzystaniem frameworka Laravel.

## Zawartość

[Stosuj zasadę pojedynczej odpowiedzialności](#stosuj-zasadę-pojedynczej-odpowiedzialności)

[Oszczędzaj kod w kontrolerach, kosztem modeli](#oszczędzaj-kod-w-kontrolerach-kosztem-modeli)

[Walidacja](#walidacja)

[Umieszczaj logikę biznesową w serwisach](#umieszczaj-logikę-biznesową-w-serwisach)

[Zasada reużywalności kodu](#zasada-reużywalności-kodu)

[Używaj Eloquent'a zamiast Query Builder'a / surowych zapytań SQL. Używaj kolekcji zamiast tablic](#używaj-eloquenta-zamiast-query-buildera--surowych-zapytań-sql-używaj-kolekcji-zamiast-tablic)

[Masowe przypisywanie (Mass assignment)](#masowe-przypisywanie-mass-assignment)

[Nie wykonuj zapytań bezpośrednio w szablonach. Używaj funkcjonalności Eager loading'u (problem N + 1)](#nie-wykonuj-zapytań-bezpośrednio-w-szablonach-używaj-funkcjonalności-eager-loadingu-problem-n--1)

[Komentuj swój kod wszędzie](#komentuj-swój-kod-wszędzie)

[Nie umieszczaj kodu JS i CSS w szablonach Blade, ani kodu HTML w klasach PHP](#nie-umieszczaj-kodu-js-i-css-w-szablonach-blade-ani-kodu-html-w-klasach-php)

[Używaj konfiguracji, plików językowych i stałych zamiast czystego tekstu w kodzie](#używaj-konfiguracji-plików-językowych-i-stałych-zamiast-czystego-tekstu-w-kodzie)

[Używaj paczek i narzędzi preferowanych przez społeczność Laravel'a](#używaj-paczek-i-narzędzi-preferowanych-przez-społeczność-laravela)

[Stosuj się do konwencji nazewnictwa w Laravelu](#stosuj-się-do-konwencji-nazewnictwa-w-laravelu)

[Używaj krótszej oraz czytelniejszej składni wszędzie gdzie to możliwe](#używaj-krótszej-oraz-czytelniejszej-składni-wszędzie-gdzie-to-możliwe)

[Używaj IoC container lub facades zamiast new Class](#używaj-ioc-container-lub-facades-zamiast-new-class)

[Nie pobieraj danych bezpośrednio z pliku `.env`](#nie-pobieraj-danych-bezpośrednio-z-pliku-env)

[Przechowuj daty w standardowych formatach. Używaj akcesorów i mutatorów aby wyświetlić je w prawidłowym formacie](#przechowuj-daty-w-standardowych-formatach-używaj-akcesorów-i-mutatorów-aby-wyświetlić-je-w-prawidłowym-formacie)

[Inne dobre praktyki](#inne-dobre-praktyki)

### **Stosuj zasadę pojedynczej odpowiedzialności**

Każda klasa oraz metoda powinna mieć wyłącznie [jedną odpowiedzialność](https://pl.wikipedia.org/wiki/Zasada_jednej_odpowiedzialności). Innymi słowy powinna być stworzona i wykorzystywana tylko w jednym, określonym i z reguły prostym celu.

Źle:

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

Dobrze:

```php
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[🔝 Wróć do zawartości](#zawartość)

### **Oszczędzaj kod w kontrolerach, kosztem modeli**

Jeżeli używasz *Query Builder* lub surowych zapytań SQL, umieść całą logikę bazy danych w modelach (*Eloquent Models*)
lub w klasach z repozytoriami (*Repository Classes*).

Źle:

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

Dobrze:

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[🔝 Wróć do zawartości](#zawartość)

### **Walidacja**

Przenieś logikę odpowiedzialną za walidację z kontrolerów do *Request Classes*.

Źle:

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

Dobrze:

```php
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[🔝 Wróć do zawartości](#zawartość)

### **Umieszczaj logikę biznesową w serwisach**
 
Skomplikowaną logikę biznesową umieszczaj w serwisach (*ServiceContainer Classes*), aby ograniczyć kod w kontrolerach do
niezbędnego minimum.

Źle:

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ....
}
```

Dobrze:

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

[🔝 Wróć do zawartości](#zawartość)

### **Zasada reużywalności kodu**

Staraj się wydzielać powtarzalne części tworzonego kodu, które będzie można wykorzystywać w wielu miejscach aplikacji.
Zwróć uwagę na fakt, że najwięcej reużywalnych bloków kodu można stworzyć w tych obszarach: *Blate Templates*,
*Eloquent Scopes*, *Service Containers*, *Helpers* itd.

Źle:

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Dobrze:

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[🔝 Wróć do zawartości](#zawartość)

### **Używaj Eloquent'a zamiast Query Builder'a / surowych zapytań SQL. Używaj kolekcji zamiast tablic**

*Eloquent* pozwala na pisanie czytelnego oraz łatwego w utrzymaniu kodu. Ponadto, *Eloquent* posiada wiele przydatnych
wbudowanych narzędzi, takich jak: miękkie usuwanie (*Soft Deletes*), wydarzenia (*Events*), *Scopes* itd.

Źle:

```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

Dobrze:

```php
Article::has('user.profile')->verified()->latest()->get();
```

[🔝 Wróć do zawartości](#zawartość)

### **Masowe przypisywanie (Mass assignment)**

Wykorzystuj wbudowaną funkcjonalność *Mass Assignment* - dzięki temu kod stanie się bardziej czytelniejszy. Nie zapomnij
o walidacji danych, a także określeniu polityki bezpieczeństwa pól modelu (*fillable* oraz *guarded*).

Źle:

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
// Add category to article
$article->category_id = $category->id;
$article->save();
```

Dobrze:

```php
$category->article()->create($request->validated());
```

[🔝 Wróć do zawartości](#zawartość)

### **Nie wykonuj zapytań bezpośrednio w szablonach. Używaj funkcjonalności Eager loading'u (problem N + 1)**

Źle (dla 100 użytkowników zostanie wykonanych 101 zapytań do bazy danych):

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Dobrze (dla 100 użytkowników zostaną wykonane 2 zapytania do bazy danych):

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[🔝 Wróć do zawartości](#zawartość)

### **Komentuj swój kod wszędzie**

Komentuj swój kod wszędzie gdzie to możliwe, ale postaraj się zastępować tradycyjne komentarze czytelniejszym nazywaniem
metod i zmiennych.

Źle:

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

Lepiej:

```php
// Determine if there are any joins.
if (count((array) $builder->getQuery()->joins) > 0)
```

Dobrze:

```php
if ($this->hasJoins())
```

[🔝 Wróć do zawartości](#zawartość)

### **Nie umieszczaj kodu JS i CSS w szablonach Blade, ani kodu HTML w klasach PHP**

Źle:

```php
let article = `{{ json_encode($article) }}`;
```

Lepiej:

```php
<input id="article" type="hidden" value="@json($article)">

Lub

<button class="js-fav-article" data-article="@json($article)">{{ $article->name }}<button>
```

W pliku Javascript:

```javascript
let article = $('#article').val();
```

Najlepzym rozwiązaniem jest użycie wyspecjalizowanego pakietu przesyłającego dane z PHP do JS.

[🔝 Wróć do zawartości](#zawartość)

### **Używaj konfiguracji, plików językowych i stałych zamiast czystego tekstu w kodzie**

Źle:

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

Dobrze:

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[🔝 Wróć do zawartości](#zawartość)

### **Używaj paczek i narzędzi preferowanych przez społeczność Laravel'a**

Korzystaj z wbudowanych funkcjonalności Laravela bądź paczek stworzonych przez jego społeczność, zamiast używać
rozwiązań i narzędzi osób trzecich. Każdy deweloper który będzie pracował z Twoją aplikacją w przyszłości, będzie musiał
uczyć się ich wszystkich, co znacznie wydłuży czas wprowadzania jakichkolwiek zmian. Ponadto, szansa na uzyskanie pomocy
wśród społeczności Laravela spadnie, jeżeli będziesz wykorzystywać paczki/narzędzia osób trzecich. Nie pozwól, aby Twój
klient musiał płacić za nieprzemyślane wybory oraz decyzje.


Zadanie | Standardowe narzędzia | Narzędzia osób trzecich
------------ | ------------- | -------------
Autoryzacja | Policies | Entrust, Sentinel and other packages
Kompilowanie zasobów | Laravel Mix | Grunt, Gulp, 3rd party packages
Środowisko deweloperskie | Homestead | Docker
Integracja ciągła | Laravel Forge | Deployer and other solutions
Testowanie jednostkowe | PHPUnit, Mockery | Phpspec
Testy przeglądarki | Laravel Dusk | Codeception
Interfejs (PDO) bazy danych | Eloquent | SQL, Doctrine
Szablony | Blade | Twig
Praca z danymi | Laravel collections | Arrays
Walidacja formularzy | Request classes | 3rd party packages, validation in controller
Uwierzytelnianie | Wbudowane | 3rd party packages, your own solution
Uwierzytelnianie w API | Laravel Passport | 3rd party JWT and OAuth packages
Tworzenie API | Wbudowane | Dingo API and similar packages
Zarządzanie strukturą bazy danych | Migrations | Working with DB structure directly
Tłumaczenia / Lokalizacja | Wbudowane | 3rd party packages
Interfejsy czasu rzeczywistego | Laravel Echo, Pusher | 3rd party packages and working with WebSockets directly
Generowanie danych testowych | Seeder classes, Model Factories, Faker | Creating testing data manually
Planowanie zadań (CRON) | Laravel Task Scheduler | Scripts and 3rd party packages
Bazy danych | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

[🔝 Wróć do zawartości](#zawartość)

### **Stosuj się do konwencji nazewnictwa w Laravelu**

Stosuj się do [standardów PSR](http://www.php-fig.org/psr/psr-2/), a także do konwencji nazewnictwa w Laravelu
stosowanej wśród społeczności deweloperów:

Co? | Jak? | Dobrze | Źle
------------ | ------------- | ------------- | -------------
Kontroler | liczba pojedyncza | ArticleController | ~~ArticlesController~~
Ścieżka URI (*Route*) | liczba mnoga | articles/1 | ~~article/1~~
*Named route* | snake_case wraz z notacją kropkową | users.show_active | ~~users.show-active, show-active-users~~
Model | liczba pojedyncza | User | ~~Users~~
Relacja hasOne lub belongsTo | liczba pojedyncza | articleComment | ~~articleComments, article_comment~~
Wszystkie inne relacje | liczba mnoga | articleComments | ~~articleComment, article_comments~~
Tabela | liczba mnoga | article_comments | ~~article_comment, articleComments~~
*Pivot table* | liczba pojedyncza; nazwy modeli w kolejności alfabetycznej | article_user | ~~user_article, articles_users~~
Kolumna tabeli | snake_case bez nazwy modelu | meta_title | ~~MetaTitle; article_meta_title~~
*Model property* | snake_case | $model->created_at | ~~$model->createdAt~~
Klucz obcy | liczba pojedyncza; nazwa modelu z postfixem _id | article_id | ~~ArticleId, id_article, articles_id~~
Klucz podstawowy | - | id | ~~custom_id~~
Migracja | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
Metoda w klasie | camelCase | getAll | ~~get_all~~
Metoda w *Resource Controller* | [table](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
Metoda w klasie testowej | camelCase | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
Zmienna | camelCase | $articlesWithAuthor | ~~$articles_with_author~~
*Collection* | liczba mnoga; opisowa | $activeUsers = User::active()->get() | ~~$active, $data~~
Obiekt | liczba pojedyncza; opisowa | $activeUser = User::active()->first() | ~~$users, $obj~~
Pliki konfiguracyjne oraz językowe | snake_case | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
Widok | snake_case | show_filtered.blade.php | ~~showFiltered.blade.php, show-filtered.blade.php~~
Plik konfiguracyjny | snake_case | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
*Contract (interface)* | przymiotnik lub rzeczownik | Authenticatable | ~~AuthenticationInterface, IAuthentication~~
*Trait* | przymiotnik | Notifiable | ~~NotificationTrait~~

[🔝 Wróć do zawartości](#zawartość)

### **Używaj krótszej oraz czytelniejszej składni wszędzie gdzie to możliwe**

Źle:

```php
$request->session()->get('cart');
$request->input('name');
```

Dobrze:

```php
session('cart');
$request->name;
```

Więcej przykładów:

Standardowa składnia | Skrócona i czytelniejsza składnia
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? null : $object->relation->id` | `optional($object->relation)->id`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

[🔝 Wróć do zawartości](#zawartość)

### **Używaj IoC container lub facades zamiast new Class**

Składnia *new Class* tworzy ścisłe połączenie między klasami i komplikuje testowanie. Zamiast tego użyj *IoC Container*
or *Facades*.

Źle:

```php
$user = new User;
$user->create($request->validated());
```

Dobrze:

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->validated());
```

[🔝 Wróć do zawartości](#zawartość)

### **Nie pobieraj danych bezpośrednio z pliku `.env`**

Przekaż dane z `.env` do plików konfiguracyjnych, a następnie użyj funkcji `config ()`, aby użyć danych w aplikacji.

Źle:

```php
$apiKey = env('API_KEY');
```

Dobrze:

```php
// config/api.php
'key' => env('API_KEY'),

// Use the data
$apiKey = config('api.key');
```

[🔝 Wróć do zawartości](#zawartość)

### **Przechowuj daty w standardowych formatach. Używaj akcesorów i mutatorów aby wyświetlić je w prawidłowym formacie**

Źle:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

Dobrze:

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at'];
public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

[🔝 Wróć do zawartości](#zawartość)

### **Inne dobre praktyki**

* Nigdy nie umieszczaj logiki w plikach ze ścieżkami (*Routes Files*)

* Zminimalizuj używanie czystego PHP w szablonach Blade'a.

* Stosuj się do [najlepszych praktyk pisania czystego kodu w PHP (ang.)](https://github.com/jupeter/clean-code-php)

[🔝 Wróć do zawartości](#zawartość)
