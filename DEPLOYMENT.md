# CI/CD და განთავსება

## 1. უწყვეტი ინტეგრაცია

ფაილი: `.github/workflows/ci.yml` — ეშვება ავტომატურად ყოველ push-ზე.
საჯარო რეპოზიტორიისთვის უფასოა.

| Job | რას ამოწმებს |
|---|---|
| Backend · Go | proto-ს გენერაცია, gofmt, `go vet`, ტესტები, 5 სერვისის აგება |
| Backend · .NET | restore და Release build |
| Database | ნამდვილ PostgreSQL 16-ზე ისხმება სქემა და seed, მოწმდება ჩანაწერები |
| Mobile · Flutter | pub get, analyze, web-ბილდი |
| Docker | gateway და auth образების აგება |

შედეგები: რეპოზიტორიის **Actions** ჩანართი.

---

## 2. განთავსება Render-ზე

ფაილი: `render.yaml` — აღწერს 7 რესურსს: ბაზა, gateway, auth და ოთხი
მიკროსერვისი, პლუს Flutter web სტატიკური საიტი.

### ნაბიჯი 1 — Blueprint

1. <https://dashboard.render.com> → **New +** → **Blueprint**
2. აირჩიე რეპოზიტორია `georgia-travel-guide` → **Connect**
3. დაარქვი სახელი და **Apply**

### ნაბიჯი 2 — JWT_SECRET

შეავსე **ორივე** სერვისზე, **ზუსტად ერთი და იგივე** მნიშვნელობით
(მინიმუმ 32 სიმბოლო):

- `guide-gateway` → Environment → `JWT_SECRET`
- `guide-auth` → Environment → `JWT_SECRET`

> auth ხელს აწერს ტოკენს, gateway კი ამოწმებს. ერთი სიმბოლოს სხვაობაც
> ავტორიზაციას ჩააგდებს.

### ნაბიჯი 3 — სერვისების მისამართები

დაელოდე, სანამ auth, city, places, route და favorite აეწყობა. შემდეგ
`guide-gateway` → Environment-ში ჩაწერე მათი მისამართები:

| ცვლადი | ფორმატი |
|---|---|
| `AUTH_ADDR` | `guide-auth-xxxx.onrender.com:443` |
| `CITY_ADDR` | `guide-city-xxxx.onrender.com:443` |
| `PLACES_ADDR` | `guide-places-xxxx.onrender.com:443` |
| `ROUTE_ADDR` | `guide-route-xxxx.onrender.com:443` |
| `FAVORITE_ADDR` | `guide-favorite-xxxx.onrender.com:443` |

⚠️ **`https://` გარეშე, `:443`-ით ბოლოში.**

შემდეგ `guide-web` → `API_BASE_URL` = `https://guide-gateway-xxxx.onrender.com`
(აქ პირიქით: `https://`-ით და პორტის გარეშე).

ბოლოს `guide-gateway`-ზე **Manual Deploy → Deploy latest commit**.

### ნაბიჯი 4 — ბაზის შევსება

Render-ის ბაზა ცარიელი იქმნება. ერთხელ უნდა შეავსო:

1. Dashboard → `guide-db` → **Connect** → დააკოპირე **PSQL Command**
2. გაუშვი ტერმინალში (საჭიროა `psql`; ალტერნატივა — Render-ის ვებ-შელი)
3. ჩასვი `backend/db/migrations/001_init.sql`-ის შიგთავსი, შემდეგ `backend/db/seed.sql`-ისა

შემოწმება:
```sql
SELECT id, name FROM cities;
```

### ნაბიჯი 5 — შემოწმება

```
https://guide-gateway-xxxx.onrender.com/health       → {"status":"ok"}
https://guide-gateway-xxxx.onrender.com/api/cities   → ქალაქები
https://guide-web-xxxx.onrender.com                  → აპლიკაცია
```

---

## 3. უფასო ტარიფის შეზღუდვები

| შეზღუდვა | გავლენა |
|---|---|
| სერვისი ითიშება 15 წუთში | პირველი მოთხოვნა 30–60 წამს გრძელდება |
| 750 ინსტანს-საათი თვეში ყველა სერვისზე | 6 სერვისი სწრაფად ხარჯავს |
| უფასო ბაზა იშლება ~30 დღეში | deploy გააკეთე დემონსტრაციასთან ახლოს |
| სტატიკური საიტი არ ითიშება | Flutter web ყოველთვის ხელმისაწვდომია |

**დემონსტრაციამდე 5 წუთით ადრე გახსენი gateway-ის URL**, რომ სერვისებმა
გაიღვიძონ.

---

## 4. სარეზერვო გეგმა

ლოკალური გაშვება ყოველთვის მუშაობს და არქიტექტურას უკეთესად აჩვენებს —
იქ ხუთივე სერვისი მართლა ცალკე კონტეინერშია:

```bash
cd backend
docker compose up -d --build
curl http://localhost:8080/api/cities

cd ../mobile
flutter run -d chrome --dart-define=API_BASE_URL=http://localhost:8080
```
