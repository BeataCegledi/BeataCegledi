# OBSERVER MINTA (BEOBACHTER-MUSTER)

## A probléma, amit megold
**Mit csinál:** Egy objektum (pl. egy óra-ticker) adatot állít elő, és több kijelzőnek kell megmutatnia. Observer nélkül a ticker maga dönti el, ki jelenítse meg, és ebből egyre hosszabb if-else lánc lesz. Minden új kijelzőnél bele kell nyúlni a tickerbe.
**Példa:**
```java
// ❌ Observer nélkül: a ticker mindenkit ismer
if (konzolKell) {
    System.out.println(uhrzeit);
} else if (fenster) {
    button.setText(uhrzeit);
} else if (...) {   // minden új kijelzőnél új ág kell
    ...
}
```
**Az ötlet:** a ticker ne ismerje a kijelzőket. Csak egy listája legyen, és mindenkinek szóljon, aki rajta van.
**Hasonlat:** YouTube-csatorna és feliratkozók. A csatorna nem ismeri személyesen a feliratkozókat, csak a listát. Új videónál mindenki értesítést kap.
---

## Szerepek
**Mit csinál:** Az Observer mintában négy szerep van (Gamma-könyv, 368. o.).

| Szerep | Feladata | Órai példa (UhrTicker) |
|---|---|---|
| **Subject** | előírja, hogyan lehet fel- és leiratkozni | `Subject` (interface, a tanárnál nincs meg) |
| **ConcreteSubject** | tárolja a listát, változáskor mindenkit értesít | `UhrTicker` |
| **Observer** | előírja az értesítő metódust | `UhrViewer` (interface) |
| **ConcreteObserver** | reagál az értesítésre | `KonsoleViewer`, `UhrFenster` |
| **Client** | létrehozza és összeköti az objektumokat | `StartKlasse` |

**Laza csatolás (lose Kopplung):** a Subject csak az Observer interface-t ismeri, a konkrét osztályokat nem. Egy új kijelzőhöz elég egy új osztály `implements UhrViewer`-rel és egy `add`, a tickert nem kell módosítani.
---

## Az órai példa (UhrTicker)
**Mit csinál:** A ticker saját szálon fut, másodpercenként lekéri az időt, és a lista minden elemén meghívja a `zeigeUhrzeit`-et.
**Példa:**
```java
public interface UhrViewer {
    void zeigeUhrzeit(String uzeit);
}

public class KonsoleViewer implements UhrViewer {
    @Override
    public void zeigeUhrzeit(String uzeit) { System.out.println(uzeit); }
}

public class UhrTicker extends Thread {
    ArrayList<UhrViewer> uListe = new ArrayList<>();

    private void ticke() {
        String uhrzeit = LocalTime.now().withNano(0).toString();
        for (UhrViewer u : uListe) {      // a kulcssor: mindenkit értesít
            u.zeigeUhrzeit(uhrzeit);
        }
    }

    @Override
    public void run() {
        while (true) {
            ticke();
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
        }
    }
}

// main:
UhrTicker tickTok = new UhrTicker();
tickTok.uListe.add(new KonsoleViewer());
tickTok.uListe.add(new UhrFenster());
tickTok.start();
```
**A program menete:**
```
main()
 ├─ new UhrTicker()             → üres lista
 ├─ new KonsoleViewer() / new UhrFenster()
 ├─ uListe.add(...) ×2          → "feliratkozás"
 └─ tickTok.start()             → új szál → run()
                                     └─ while(true): ticke() → for-each → zeigeUhrzeit(); sleep(1000)
```
A ticker csak `UhrViewer`-t lát, futáskor mégis mindig a megfelelő `zeigeUhrzeit` fut le. Ez a **polimorfizmus**.

**Hiányosságok a tanár kódjában:**
- a `main` közvetlenül a listába ír (`tickTok.uListe.add(...)`), és a lista nem `private`, ezért sérül a kapszulázás;
- nincs `attach`/`detach` metódus és `Subject` interface;
- az `import javax.swing.JButton;` a `UhrTicker`-ben felesleges.
---

## Subject interface (`attach` / `detach` / `notify`)
**Mit csinál:** A megfigyelt fél (Subject) szerződése. Minden Subjectnél ugyanúgy működik a feliratkozás, a leiratkozás és az értesítés.
**Példa:**
```java
// ❌ Subject interface nélkül: mindenki máshogy nevezi el
ticker.uListe.add(v);
station.registriere(a);
boerse.addListener(x);

// ✅ Subject interface-szel: mindenhol ugyanaz
public interface Subject {
    void attach(UhrViewer v);    // feliratkozás (anmelden)
    void detach(UhrViewer v);    // leiratkozás (abmelden)
    void notifyViewers();        // mindenkinek szól
}
```
**Mikor kell?** Ha több fajta Subject van (minden Subject egy-egy „levelezési lista”: időpont-lista, időjárás-lista, árfolyam-lista), vagy ha a teljes, tankönyvi UML-t kell lerajzolni. Egyetlen Subjectnél nem kötelező.
Egy Observer több Subjecten is rajta lehet (több listára is feliratkozhat).
**Hasonlat:** USB-csatlakozó. Mindegy, milyen készülék, ugyanúgy dugod be (`attach`) és húzod ki (`detach`).
A Gamma-könyvben (C++) a `Subject` absztrakt osztály, mert a lista és a `notify` már készen van benne.
---

## Push vs. pull
**Mit csinál:** Megmondja, hogyan jut el az adat az Observerhez.

| | **Push** (küldés) | **Pull** (lekérés) |
|---|---|---|
| Ötlet | a Subject elküldi az adatot | a Subject csak szól, az Observer maga kérdezi le |
| Metódus | `refresche(double temperatur)` | `refresche()` + `station.getTemperatur()` |
| Előny | egyszerű; az Observer nem ismeri a Subjectet | rugalmas: új adatnál a szignatúra nem változik |
| Hátrány | új adat = új paraméter, minden Observert át kell írni | szorosabb csatolás (ismeri a Subjectet és a gettereit), több kód, az adatok inkonzisztensek lehetnek |
| Hol láttuk | tanár: `zeigeUhrzeit(String uzeit)` | Gamma: `update()` + `getState()` |

**Középút: push egy adatobjektummal (DTO).** Az értékek egyetlen objektumban mennek: `refresche(WetterDaten daten)`. Új adatnál csak a `WetterDaten` bővül, a szignatúra marad, és az adatok egy pillanatból származnak. Ez a legelterjedtebb (pl. Swing: `actionPerformed(ActionEvent e)`).
---

## DTO (Data Transfer Object)
**Mit csinál:** Logika nélküli objektum, amely csak adatokat szállít rétegek vagy objektumok között.
**Példa:**
```java
new WetterDaten(21.5, 60, 12)   // temp, Feuchte, Wind
```
---

## Observer és MVC
**Mit csinál:** Az MVC-ben az Observer köti össze a Model-t és a View-t. A Model nem ismeri a konkrét View-kat, ezért bármikor lehet új nézetet hozzáadni.

| Rész | Feladata | Wetterstation-példa |
|---|---|---|
| **Model** | adatokat és logikát tárol, változáskor értesít | `Wetterstation` |
| **View** | megjeleníti az adatokat | a checkerek (`KonsoleAnzeige`, `FensterAnzeige` …) |
| **Controller** | fogadja a felhasználó bemenetét, vezérli a Model-t | pl. egy „Messen”-gomb |

```
Felhasználó ──► Controller ──► Model
                                 │  Observer: bescheid()
                                 ▼
                          View 1, View 2, … ──► Felhasználó látja
```
- Observer push `double`-lal vagy DTO-val: Model–View szétválasztás ✅, Controller ❌.
- Controllerrel együtt: **teljes MVC**.
- A Gamma-könyv szerint az MVC-ben: Observer (Model ↔ View), Composite (egymásba ágyazott nézetek), Strategy (View ↔ Controller).

> [!TIP] IHK-tipp
> Tipikus feladat: osztályokat besorolni a Model/View/Controller szerepekbe, vagy elmagyarázni az előnyét.
> „MVC trennt Datenhaltung (Model), Darstellung (View) und Steuerung (Controller). Vorteil: Austauschbarkeit und Wiederverwendbarkeit, z. B. mehrere Views für dasselbe Model.“
---

## Saját gyakorlat: Wetterstation (Subject interface `default` metódusokkal)
**Mit csinál:** A Subject interface kész metódusokat ad, a listát pedig egy absztrakt getteren keresztül kéri el az implementáló osztálytól.
**Példa:**
```java
public interface WetterList {

    List<WetterChecker> getList();   // absztrakt: az osztály adja a SAJÁT listáját

    default void hinzufuegen(WetterChecker subscriber) {
        getList().add(subscriber);
    }

    default void entfernen(WetterChecker subscriber) {
        getList().remove(subscriber);
    }

    default void bescheid() {
        for (WetterChecker subscriber : getList()) {
            subscriber.refresche();
        }
    }
}
```
**Az első (hibás) próbálkozás tanulságai:**
```java
List<WetterList> liste = getList();          // ❌ mező interface-ben = static final → nincs objektum, akinek a getList()-jét hívni lehetne
public static void hinzufuegen(...) {...}    // ❌ static metódusból nem hívható getList() → default kell
List<WetterList> ...                         // ❌ a listában a megfigyelők (WetterChecker) vannak, nem Subjectek
bescheid(...) { liste.add(...); }            // ❌ értesítés = for-each + az Observer metódusának hívása
```
---

> [!TIP] IHK-tipp
> Az Observer gyakori téma a Design Patterns között, sokszor UML-osztálydiagrammal. Felismerés: két interface (Subject, Observer) és egy `0..*` kapcsolat a Subjecttől az Observer felé.
> „Beim Beobachter-Muster benachrichtigt ein Subjekt alle registrierten Beobachter über Zustandsänderungen, ohne deren konkrete Klassen zu kennen (lose Kopplung).“

# INTERFACE ÉS ABSZTRAKT OSZTÁLY

## `interface`
**Mit csinál:** Szerződés: csak metódusfejléceket ír elő, objektumot nem lehet belőle létrehozni (nincs konstruktora).
**Példa:**
```java
public interface UhrViewer {
    void zeigeUhrzeit(String uzeit);   // automatikusan public abstract
}
new UhrViewer();                       // ❌ interface-ből nem lehet objektum
```
**Hasonlat:** álláshirdetés: „Aki itt dolgozik, annak tudnia kell `zeigeUhrzeit`-et.” Hogy hogyan, az a saját dolga.
---

## `implements`
**Mit csinál:** Az osztály vállalja, hogy megvalósítja az interface összes absztrakt metódusát. Egy osztály több interface-t is implementálhat.
**Példa:**
```java
class KonsoleViewer implements UhrViewer { ... }
class UhrFenster extends JFrame implements UhrViewer { ... }
```
---

## `@Override`
**Mit csinál:** Jelzi, hogy a metódus az interface-ből vagy az ősosztályból jön. Elírásnál a fordító hibát dob.
**Példa:**
```java
@Override
public void zeigeUhrzeit(String uzeit) { ... }
```
---

## Interface mint típus
**Mit csinál:** A változó típusa az interface, az objektum egy konkrét osztály. Így egy listában különböző osztályok lehetnek.
**Példa:**
```java
UhrViewer v = new KonsoleViewer();
ArrayList<UhrViewer> liste = new ArrayList<>();   // KonsoleViewer és UhrFenster is belefér
```
---

## Mező interface-ben
**Mit csinál:** Automatikusan `public static final`: egyetlen közös konstans, nem objektumonkénti változó.
**Példa:**
```java
int MAX = 10;   // = public static final int MAX = 10;
List<WetterChecker> checkers = new ArrayList<>();   // ⚠️ EGY közös lista minden Subjectnek!
```
⚠️ Csapda: két `Wetterstation` (Berlin, München) ugyanabba a listába írna.
---

## `private` metódus interface-ben (Java 9+)
**Mit csinál:** Csak a `default` metódusok belső segédje lehet, kívülről nem hívható. Feliratkozó metódusnak ezért alkalmatlan.
**Példa:**
```java
private void hinzufuegen(WetterChecker c) { ... }   // ❌ kívülről nem hívható
void attach(WetterChecker c);                       // ✅
```
---

## `default` metódus (interface)
**Mit csinál:** A `default` kulcsszó az interface-ben azt jelöli, hogy a metódusnak van törzse, és ez az alapértelmezett megvalósítás. Ha az implementáló osztály nem írja felül, ez fut le. Ha felülírja (`@Override`), akkor a saját változata fut.
**Példa:**
```java
interface WetterList {
    List<WetterChecker> getList();        // absztrakt: kötelező megírni
    default void bescheid() {             // default: kész, felülírni opcionális
        for (WetterChecker c : getList()) c.refresche();
    }
}
```

⚠️ **A névzavar oka: a `default` szó három dolgot jelenthet Javában**

| Jelentés | Hol | Példa |
|---|---|---|
| default metódus | interface-ben | `default void bescheid() { … }` |
| default ág | `switch`-ben | `default: System.out.println("?");` |
| „default” hozzáférés (package-private) | osztályokban | `void m() { … }` (nincs kulcsszó!) |

- A `default` **nem** hozzáférés-módosító, hanem olyan kulcsszó, mint a `static` vagy az `abstract`. A metódus láthatósága `public` marad.
- Osztályban `default void m()` fordítási hiba.
- Miért vezették be (Java 8)? Hogy régi interface-ekhez (pl. `List.forEach`) új metódust adhassanak úgy, hogy a meglévő implementációk ne romoljanak el.

| Hozzáférés-módosítók | Egyéb módosítók |
|---|---|
| `public`, `protected`, `private`, (semmi = package-private) | `static`, `final`, `abstract`, `default` |
---

## `static` vs. `default` metódus interface-ben
**Mit csinál:** A `static` az interface-hez tartozik (`WetterList.metodus()`), a `default` egy objektumon fut.

| | `static` | `default` |
|---|---|---|
| Kell hozzá objektum? | nem | igen |
| Hívhat absztrakt metódust (pl. `getList()`)? | ❌ | ✅ |
| Az osztály felülírhatja? | ❌ | ✅ |
---

## Absztrakt getter + `default` (állapot az osztályban)
**Mit csinál:** Az interface nem tárolhat állapotot, ezért egy absztrakt getteren keresztül kéri el az osztálytól. A listát az implementáló osztály tárolja (mező), a `default` metódusok a getteren át dolgoznak.
**Példa:**
```java
interface WetterSubject {
    List<WetterChecker> getCheckers();
    default void attach(WetterChecker c) { getCheckers().add(c); }
}
```
```
Wetterstation-objektum (Berlin)
 ├─ mező: checkers  ← SAJÁT lista
 ├─ getCheckers()   → visszaadja a saját listát
 └─ attach(c)       → a default törzs fut, a berlini listán
```
- ✅ minden objektumnak saját listája van;
- ✅ az osztály közben örökölhet mástól (pl. `Thread`);
- ⚠️ a `getCheckers()` kötelezően `public`, tehát bárki hozzányúlhat a listához (gyengébb kapszulázás).

**Hasonlat:** az interface egy kézikönyv: „Tedd a levelet a postaládába.” Postaládája mindenkinek a saját házán van, a getter azt mutatja meg, hol.
---

## `abstract class`
**Mit csinál:** Nem példányosítható osztály. Lehetnek benne mezők, kész metódusok és absztrakt metódusok. Csak egyetlen osztályból lehet örökölni.
**Példa:**
```java
public abstract class WetterSubject {
    protected List<WetterChecker> checkers = new ArrayList<>();   // minden objektumnak saját lista
    public void attach(WetterChecker c) { checkers.add(c); }       // kész, örökölhető
}
public class Wetterstation extends WetterSubject { ... }
```

| | Interface | Absztrakt osztály |
|---|---|---|
| Példányváltozó (mező) | ❌ (csak `static final` konstans) | ✅ |
| Kész metódus | csak `default` (állapot nélkül) | ✅ |
| Konstruktor | ❌ | ✅ |
| Egy osztály hányat használhat | többet (`implements A, B`) | csak egyet (`extends`) |

A tanár `UhrTicker`-e már `extends Thread`, ezért nem örökölhetne egy absztrakt `Subject` osztályból is.

> [!TIP] IHK-tipp
> „Eine abstrakte Klasse kann Attribute und implementierte Methoden enthalten, eine Klasse kann aber nur von einer Klasse erben. Ein Interface definiert nur einen Vertrag; eine Klasse kann mehrere Interfaces implementieren.“
---

# THREAD

## `extends Thread` + `run()` / `start()`
**Mit csinál:** A `run()` tartalmazza, ami a saját szálon fut. A `start()` indítja el az új szálat. Ha közvetlenül a `run()`-t hívod, nem jön létre új szál.
**Példa:**
```java
class UhrTicker extends Thread {
    @Override public void run() { while (true) { ... } }
}
tickTok.start();   // ✅ új szál
tickTok.run();     // ❌ a main szálon fut, blokkol
```
---

## `Thread.sleep(ms)`
**Mit csinál:** Megállítja az aktuális szálat ennyi milliszekundumra. `InterruptedException`-t dobhat, ezért kell a `try/catch`.
**Példa:**
```java
try { Thread.sleep(1000); } catch (InterruptedException e) {}
```
---

# PRIMITÍV ÉS REFERENCIATÍPUSOK

## Referenciatípus
**Mit csinál:** A változó egy objektumra mutató hivatkozást (címet) tárol, nem magát az objektumot. Két változó mutathat ugyanarra az objektumra.
**Példa:**
```java
List<String> a = new ArrayList<>();
List<String> b = a;        // nem másolat, ugyanaz a lista
b.add("Hallo");
System.out.println(a);     // [Hallo]
```
A getter egy lista esetén ugyanazt a listát adja vissza, ezért a `getList().add(x)` az eredetit módosítja. **Ehhez nem kell setter.**
**Hasonlat:** a getter nem fénymásolatot ad a postaládáról, hanem a kulcsot.
---

## Getter vs. setter
**Mit csinál:** A getter kiadja a mező értékét. A setter kívülről kap egy új értéket, azzal **lecseréli** a mezőt, és `void`.
**Példa:**
```java
public List<X> getCheckers() { return checkers; }          // getter: kiad
public void setCheckers(List<X> l) { this.checkers = l; }   // setter: lecserél
```

| | Getter | Setter |
|---|---|---|
| Irány | mezőtől kifelé | kívülről a mezőbe |
| Visszatérési érték | a mező értéke | `void` |
| Mikor kell | olvasáshoz, és referencián át módosításhoz is | ha az egész objektumot másikra cseréled, vagy primitív mezőnél |
---

## Másolat készítése (védekező getter)
**Mit csinál:** Új listát hoz létre ugyanazokkal az elemekkel. Getterben arra használják, hogy a listát kívülről ne lehessen módosítani.
**Példa:**
```java
public List<X> getCheckers() { return new ArrayList<>(checkers); }  // másolat
// getCheckers().add(c) → a másolatba kerül, az eredeti nem változik
```
---

## Primitív típus
**Mit csinál:** A változó közvetlenül az értéket tárolja. Értékadásnál és visszatérésnél másolat készül, ezért kívülről csak setterrel lehet módosítani.
**Példa:**
```java
int t = station.getTemperatur();   // másolat: 20
t++;                               // 21, de az objektumban továbbra is 20
station.setTemperatur(21);         // ✅ így változik
```

| | Primitív (`int`, `double`, `boolean` …) | Referencia (`List`, `JButton`, saját osztály …) |
|---|---|---|
| Mit tárol a változó? | az értéket | egy címet az objektumra |
| A getter visszaad … | egy másolatot | ugyanazt az objektumot |
| Kívülről módosítás | csak setterrel | a getteren át is (pl. `.add()`) |

> [!TIP] IHK-tipp
> „Primitive Datentypen speichern den Wert direkt, Referenztypen speichern einen Verweis auf ein Objekt im Heap.“ Gyakori kérdés a méret is (pl. `int` = 4 Byte).
---

## Immutable objektum
**Mit csinál:** Létrehozás után nem változtatható meg, minden művelet új objektumot ad vissza. A `String` és az `Integer` referenciatípus, mégsem lehet őket a getteren át módosítani.
**Példa:**
```java
String s = "Hi";
s.toUpperCase();   // s továbbra is "Hi"
s = s.toUpperCase();   // ✅ új objektum, "HI"
```
---

# UML OSZTÁLYDIAGRAM

## UML-jelek
**Mit csinál:** Az osztálydiagram kapcsolatai és jelölései.

| Jel | Név (DE) | Jelentés | Java |
|---|---|---|---|
| `——▷` (folytonos, üres háromszög) | Vererbung / Generalisierung | örököl egy osztálytól | `extends` |
| `- - -▷` (szaggatott, üres háromszög) | Realisierung | megvalósít egy interface-t | `implements` |
| `◇——>` (üres rombusz) | Aggregation | „van neki” (pl. listában), a részek nélküle is léteznek | attribútum, pl. `ArrayList` |
| `- - ->` (szaggatott nyíl) | Abhängigkeit | használja / létrehozza | `new` egy metódusban |
| `0..*` | Multiplizität | nulla vagy több | lista elemszáma |
| `«interface»` | Stereotyp | interface jelölése | `interface` |
| aláhúzott | statisch | osztályszintű | `static` |

**Láthatóság:**

| UML | Java |
|---|---|
| `+` | `public` |
| `#` | `protected` |
| `~` | package-private (nincs kulcsszó) |
| `-` | `private` |
---

## Observer osztálydiagram (UhrTicker, tankönyvi változat)
```
   «interface» Subject                      «interface» UhrViewer
   +attach(UhrViewer v)                     +zeigeUhrzeit(String uzeit)
   +detach(UhrViewer v)                        △               △
   +notifyViewers()                            ┆               ┆ (implements)
          △                                    ┆               ┆
          ┆ (implements)                 KonsoleViewer     UhrFenster ──▷ JFrame
          ┆
   UhrTicker ──▷ Thread
   -uListe: ArrayList<UhrViewer>
   -ticke()
   +run()
   UhrTicker ◇──────────────────────► UhrViewer
                         0..*  (uListe)

   StartKlasse - - -> UhrTicker, KonsoleViewer, UhrFenster   (erzeugt)
```
Jelmagyarázat: `△ ┆` = implements (Realisierung), `──▷` = extends (Vererbung), `◇──►` = Aggregation.
---
