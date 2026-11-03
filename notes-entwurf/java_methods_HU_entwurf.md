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

    default void bescheid(WetterDaten daten) {          // push: az adatcsomagot küldi
        for (WetterChecker subscriber : getList()) {
            subscriber.refresche(daten);
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

## Lassú Observer = lassú Subject
**Mit csinál:** A `refresche` **a Subject szálán** fut (`run()` → `messen()` → `bescheid()` → `refresche()`). Ha egy Observer blokkol (pl. `JOptionPane`), a Subject nem mér tovább, és a listában utána következő Observerek sem kapnak értesítést.
**Példa:**
```java
public void refresche(WetterDaten d) {
    JOptionPane.showMessageDialog(null, d);                        // ❌ megállítja a stationt, amíg OK-t nem nyomnak
}
public void refresche(WetterDaten d) {
    SwingUtilities.invokeLater(() -> label.setText(d.toString()));  // ✅ átadja a Swing-szálnak, azonnal visszatér
}
```
**Szabály:** az Observer legyen gyors. A lassú munkát adja át egy másik szálnak.
---

## Egy Observer több Subjecten, egy osztály több példányban
**Mit csinál:** Ugyanaz az Observer-**objektum** több Subjectre is feliratkozhat (közös kimenet). Ha minden Subjectnek **saját** Observer kell, ugyanabból az osztályból több objektumot hozol létre.
**Példa:**
```java
WetterChecker konsole = new KonsoleAnzeige();
munich.hinzufuegen(konsole);                                  // egy konzol mindkét stationnek
vs.hinzufuegen(konsole);

munich.hinzufuegen(new WarnungChecker(munich.getName()));     // saját ablak stationönként
vs.hinzufuegen(new WarnungChecker(vs.getName()));             // névtelen objektum: csak a listában él
```
⚠️ A közös Observert több szál hívja egyszerre. Ha egy kiírás több `println`-ből áll, a sorok összekeveredhetnek, ezért egy üzenet = egy `println`.
💡 A név a stationtől jön (`getName()`), így nem kell kétszer leírni, és nem lehet elírni.
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

## `@FunctionalInterface` és lambda
**Mit csinál:** Olyan interface-t jelöl, amelyben pontosan egy absztrakt metódus van. A fordító ellenőrzi ezt. Az ilyen interface lambdával is megvalósítható, külön osztály nélkül.
**Példa:**
```java
@FunctionalInterface
public interface WetterChecker {
    void refresche(WetterDaten daten);
}

WetterChecker kurz = daten -> System.out.println(daten.getTemperatur());   // lambda
station.hinzufuegen(kurz);
```
Ismert funkcionális interface-ek: `Runnable` (`() -> …`), `Comparator`, `ActionListener`.
---

## Kis interface (Interface Segregation)
**Mit csinál:** Egy interface csak azt írja elő, amire a hívónak tényleg szüksége van. Minden további metódust **minden** implementáló osztálynak meg kell írnia.

| Ide tartozik? | Hova való |
|---|---|
| `refresche(WetterDaten)` ✅ | Observer interface |
| `hinzufuegen`, `entfernen`, `bescheid` ❌ | Subject (`WetterList`) |
| adatok (mezők) ❌ | `WetterDaten` / `WetterStation` |
| kiírás, határértékek ❌ | konkrét checker osztályok |

💡 Interface-ben a `public` felesleges (minden metódus automatikusan public), de nem hiba. Az interface-nek magának viszont kell a `public`, különben csak a saját package-ében látható.
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

## `interrupt()` és az `InterruptedException`
**Mit csinál:** Az `interrupt()` jelzi a szálnak, hogy álljon le. Ha a szál éppen `sleep`-ben van, `InterruptedException`-t kap. Ez **nem** azt jelenti, hogy „a szálak zavarták egymást”, hanem leállítási kérés, ezért a helyes reakció a befejezés.
**Példa:**
```java
@Override
public void run() {
    while (true) {
        messen();
        try {
            Thread.sleep(sekunden * 1000L);
        } catch (InterruptedException e) {
            System.out.println(getName() + " wird beendet.");
            return;                         // ✅ kilép a run()-ból → a szál véget ér
        }
    }
}
// main: munich.interrupt();
```
💡 Alternatíva: `while (!isInterrupted()) { … }`. A ciklus magától is ellenőrzi a kérést.
💡 `join()`: a hívó szál megvárja, amíg a másik szál befejeződik (`station.join();`).
---

## Szál neve: `super(name)` és `getName()`
**Mit csinál:** Minden `Thread`-nek van neve. A konstruktorban adható át az ősosztálynak, és `getName()`-mel olvasható ki. A `Thread.getName()` **`final`**, ezért saját `name` mező + `getName()` fordítási hibát okoz.
**Példa:**
```java
private String name;
public String getName() { return name; }    // ❌ final metódust nem lehet felülírni

public WetterStation(String name) {
    super(name);                            // ✅ a Thread tárolja a nevet
}
station.getName();                          // "München"
```
💡 Hibakeresésnél a szál is ezen a néven jelenik meg (`Thread-0` helyett).
---

## Az objektum nem ismeri a változója nevét
**Mit csinál:** A változó neve (`berlin`) csak a forráskódban létezik, futáskor nincs meg. Egy objektumra több változó is mutathat, vagy egy sem. Ha név kell, az legyen adat (mező vagy konstruktorparaméter).
**Példa:**
```java
WetterStation berlin = new WetterStation("Berlin");
WetterStation x = berlin;      // ugyanaz az objektum, két változónév
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

## `final` referencia ≠ megváltoztathatatlan tartalom
**Mit csinál:** Referenciatípusnál a `final` azt jelenti, hogy a változó mindig **ugyanarra** az objektumra mutat. Az objektum tartalma ettől még változhat.
**Példa:**
```java
private static final List<String> alleNamen = new ArrayList<>();
alleNamen.add("Berlin");          // ✅ a tartalom változhat
alleNamen = new ArrayList<>();    // ❌ fordítási hiba: új objektumra nem cserélhető
```
**Hasonlat:** a postaláda a falhoz van csavarozva (`final`), de levelet bármikor bedobhatsz.
---

## `List.copyOf(lista)` (Java 10+)
**Mit csinál:** Megváltoztathatatlan másolatot ad vissza. Ha valaki `add`-ot vagy `clear`-t hív rajta, `UnsupportedOperationException` keletkezik. Getterben arra jó, hogy a belső listát kívülről ne lehessen módosítani.
**Példa:**
```java
public static List<String> getAlleNamen() {
    return List.copyOf(alleNamen);
}
WetterStation.getAlleNamen().clear();   // ❌ kivétel, a belső lista biztonságban van
```
Rokonai: `List.of(...)` (megváltoztathatatlan lista közvetlenül), `Collections.unmodifiableList(l)` (csak olvasható nézet, nem másolat).
---

## `record` (Java 16+)
**Mit csinál:** Egy sorban létrehoz egy immutable adatosztályt (DTO): `private final` mezők, konstruktor, getterek (`temperatur()` alakban), `toString`, `equals`, `hashCode`.
**Példa:**
```java
public record Punkt(int x, int y) {}
Punkt p = new Punkt(3, 4);
p.x();                  // 3
System.out.println(p);  // Punkt[x=3, y=4]
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

# KONSTRUKTOROK ÉS STATIC

## `super(...)`
**Mit csinál:** Az ősosztály konstruktorát hívja. Csak a konstruktor **első** sora lehet.
**Példa:**
```java
public WetterStation(String name) {
    super(name);          // Thread(String name)
}
```
---

## Konstruktor-láncolás `this(...)`
**Mit csinál:** Egy konstruktor egy másik konstruktort hív ugyanabban az osztályban, így az alapértékeket nem kell megismételni. Ez is csak az első sor lehet.
**Példa:**
```java
public WetterStation(String name, int sekunden) { super(name); this.sekunden = sekunden; }
public WetterStation(String name) { this(name, 1); }   // alapérték: 1 másodperc
```
---

## `final` mező: hol kaphat értéket?
**Mit csinál:** A `final` mező pontosan egyszer kap értéket. Hogy hol, az attól függ, hogy példány- vagy statikus mező.

| | `final` (példány) | `static final` |
|---|---|---|
| Hol kap értéket? | deklarációnál vagy **a konstruktorban** | deklarációnál vagy `static {}` blokkban |
| Miért? | objektumonként egyszer fut a konstruktor | az osztály egyszer töltődik be, a konstruktor viszont többször futna |
| Példa | `WetterDaten` mezői, `checkers` | `alleNamen`, `FORMAT` |
**Példa:**
```java
private final int sekunden;                                     // a konstruktor állítja be
private static final List<String> alleNamen = new ArrayList<>(); // deklarációnál
```
`final` mezőhöz nem lehet settert írni, mert a konstruktor után nem változhat.
---

## `static` mező: közös az osztály minden objektumának
**Mit csinál:** Egyetlen példány létezik belőle, az osztályhoz tartozik, nem az objektumhoz. Jó például számlálónak vagy annak nyilvántartására, hogy milyen objektumok jöttek létre.
**Példa:**
```java
private static int anzahl = 0;                                   // hány darab
private static final List<String> alleNamen = new ArrayList<>(); // melyik darabok

public WetterStation(String name) {
    super(name);
    anzahl++;
    alleNamen.add(name);       // minden új objektum beírja magát
}
```
```java
private static String name;    // ❌ minden objektum ugyanazt a nevet kapná: az utolsót
```
---

## `static` metódus
**Mit csinál:** Az osztályhoz tartozik, objektum nélkül, az osztály nevével hívható. Csak statikus mezőket érhet el közvetlenül.
**Példa:**
```java
public static List<String> getAlleNamen() { return List.copyOf(alleNamen); }
WetterStation.getAlleNamen();   // [München, Villingen-Schwenningen]
```
---

# ENUM

## `enum`
**Mit csinál:** Fix számú, névvel ellátott konstans, saját típusként. Az enum-értékek immutable-ek, és csak egy-egy példányuk létezik.
**Példa:**
```java
public enum WindRichtung { NORD, SUED, OST, WEST, SUED_WEST, SUED_OST, NORD_WEST, NORD_OST }
WindRichtung w = WindRichtung.NORD;
```
---

## `==` enumoknál
**Mit csinál:** Mivel minden enum-értékből csak egy objektum létezik, az `==` biztonságos és szokásos. Az `equals` is működik.
**Példa:**
```java
if (daten.getStatus() == Status.GEWITTER) { ... }
```
---

## `values()`, `ordinal()`, `name()`, `valueOf()`
**Mit csinál:** Az enum beépített metódusai.

| Metódus | Mit ad | Példa |
|---|---|---|
| `values()` | az összes értéket tömbként | `WindRichtung.values().length // 8` |
| `ordinal()` | a sorszámot (0-tól) | `WindRichtung.OST.ordinal() // 2` |
| `name()` / `toString()` | a nevet szövegként | `"NORD"` |
| `valueOf(String)` | szövegből enum-érték | `WindRichtung.valueOf("WEST")` |
---

## Véletlen enum-érték
**Mit csinál:** Véletlen indexet választ a `values()` tömbből. A felső határ kizárt, ezért `length`, nem `length + 1`.
**Példa:**
```java
WindRichtung[] alle = WindRichtung.values();
WindRichtung w = alle[ThreadLocalRandom.current().nextInt(0, alle.length)];   // ✅
WindRichtung w = alle[ThreadLocalRandom.current().nextInt(0, alle.length + 1)]; // ❌ ArrayIndexOutOfBoundsException
```
---

## `switch` enummal
**Mit csinál:** Az enum-értékeket közvetlenül, előtag nélkül lehet a `case`-ben használni.
**Példa:**
```java
switch (daten.getStatus()) {
    case GEWITTER -> System.out.println("Achtung!");
    case SONNIG   -> System.out.println("Sonnenbrille!");
    default       -> System.out.println("Normal");
}
```
---

# VÉLETLENSZÁMOK

## Áttekintés: `Math.random`, `Random`, `java.util.random`, `ThreadLocalRandom`, `SecureRandom`
**Mit csinál:** Minden változat véletlen számot ad, de különböző célra valók.

| | Mi ez? | Mikor | Import |
|---|---|---|---|
| `Math.random()` | statikus, `double` `[0, 1)` | gyors, egyszeri | nem kell |
| `Random` | osztály, objektum kell | sokszor, kényelmes metódusok, seed | `java.util.Random` |
| `RandomGenerator` (Java 17+) | interface + algoritmusok | haladó | `java.util.random.RandomGenerator` |
| `ThreadLocalRandom` | szálanként saját generátor | **több szálas** programban | `java.util.concurrent.ThreadLocalRandom` |
| `SecureRandom` | kriptográfiailag biztonságos | jelszó, token, kulcs | `java.security.SecureRandom` |

```
RandomGenerator (interface)  ◁┄┄  Random  ◁──  ThreadLocalRandom, SecureRandom
```
**Példa (dobókocka 1–6):**
```java
int a = (int)(Math.random() * 6) + 1;
int b = new Random().nextInt(6) + 1;
int c = new Random().nextInt(1, 7);                    // Java 17+: [1, 7)
int d = ThreadLocalRandom.current().nextInt(1, 7);
double t = ThreadLocalRandom.current().nextDouble(-15, 35);
```
⚠️ A felső határ **mindig kizárt**.
❌ Ne hozz létre `new Random()`-ot minden hívásnál. Egy mező vagy a `ThreadLocalRandom.current()` elég.
---

## `new Random(seed)`
**Mit csinál:** Ugyanazzal a seeddel mindig ugyanazt a sorozatot adja. Teszteléshez hasznos.
**Példa:**
```java
new Random(42).nextInt(100);   // mindig ugyanaz a szám
```
---

# IMPORT ÉS PACKAGE

## `java.lang`: automatikus import
**Mit csinál:** Az egyetlen package, amit minden Java-fájl automatikusan importál.
**Példa:**
```java
String, Math, System, Thread, Integer, Double, Object, Runnable   // import nélkül
```
---

## Package-private (kulcsszó nélkül)
**Mit csinál:** Ha nincs hozzáférés-módosító, az osztály, interface vagy tag csak a saját package-én belül látható. UML-ben: `~`.
**Példa:**
```java
interface WetterChecker { ... }          // csak a wetterStationObserver package-ben
public interface WetterChecker { ... }   // mindenhol
```
---

# SZÖVEGFORMÁZÁS ÉS DÁTUM

## `toString()` felülírása
**Mit csinál:** Az `Object`-től örökölt metódus. A `println` és a String-összefűzés automatikusan meghívja. Felülírás nélkül csak `Osztaly@hash` jelenik meg.
**Példa:**
```java
System.out.println(daten);   // toString nélkül: wetterStationObserver.WetterDaten@1b6d3586

@Override
public String toString() {
    return "Wetter in " + name + ": " + temperatur + " °C";
}
```
---

## `String.format(...)`
**Mit csinál:** Helyőrzőkkel formázott szöveget ad vissza. A `printf` ugyanígy működik, csak rögtön ki is írja.

| Jel | Jelentés | Példa → eredmény |
|---|---|---|
| `%d` | egész szám | `60` |
| `%.1f` | tört, 1 tizedesjegy | `21.4839` → `21.5` (német gépen `21,5`) |
| `%s` | szöveg / bármi (`toString`) | `NORD` |
| `%%` | egy `%` jel | `60 %` |
| `%n` | sortörés | |
**Példa:**
```java
String.format("%.1f °C, %d %%, %s", 21.48, 60, WindRichtung.NORD)   // "21.5 °C, 60 %, NORD"
```
💡 A tizedesjel a rendszer nyelvétől függ: `String.format(Locale.US, ...)` → mindig pont.
---

## `"x".repeat(n)` (Java 11+)
**Mit csinál:** A szöveget n-szer ismételve adja vissza.
**Példa:**
```java
"*".repeat(60)   // 60 csillag
```
---

## `StringBuilder`
**Mit csinál:** Hatékonyan fűz össze szöveget lépésenként, pl. ciklusban vagy feltételek alapján.
**Példa:**
```java
StringBuilder sb = new StringBuilder();
if (temp > 30) sb.append("Hitze\n");
if (sturm)     sb.append("Sturm\n");
if (sb.length() > 0) System.out.println(sb.toString());
```
---

## `LocalDateTime` és `LocalTime`
**Mit csinál:** Dátum + idő, illetve csak idő, időzóna nélkül. Immutable.
**Példa:**
```java
LocalDateTime jetzt = LocalDateTime.now();
LocalTime.now().withNano(0).toString();   // "14:37:12" (nanoszekundumok nélkül)
```
💡 Egy DTO-ban az időpont `LocalDateTime` legyen, ne `String`: az adat és a megjelenítés külön marad, a formázás a `toString()`-be kerül.
---

## `DateTimeFormatter.ofPattern(...)`
**Mit csinál:** Dátumot vagy időt formáz egy minta szerint. A mintában **minden betű** formázó jel, a szó szerinti szöveg aposztrófok közé kerül.
**Példa:**
```java
private static final DateTimeFormatter FORMAT =
        DateTimeFormatter.ofPattern("yyyy. MMMM dd. E 'um' HH:mm:ss", Locale.GERMAN);
zeit.format(FORMAT);   // "2026. November 03. Di. um 14:37:12"

"... E um HH:mm"       // ❌ u = év, m = perc → "202637"
"... E 'um' HH:mm"     // ✅
```

| Jel | Jelentés | Jel | Jelentés |
|---|---|---|---|
| `yyyy` | év | `HH` | óra (0–23) |
| `MM` / `MMMM` | hónap száma / neve | `mm` | perc |
| `dd` | nap | `ss` | másodperc |
| `E` / `EEEE` | nap neve röviden / hosszan | `'szöveg'` | szó szerint |

A hónap- és napnevek a `Locale`-tól függenek (`import java.util.Locale;`). A formattert érdemes `static final` konstansként tárolni, így nem jön létre minden híváskor újra.
---

# SWING (GUI)

## `JFrame` + `JLabel` / `JButton`
**Mit csinál:** A `JFrame` egy ablak, a `JLabel` egy felirat, a `JButton` egy gomb. Ha az osztály `extends JFrame`, ő maga az ablak.
**Példa:**
```java
public class WarnungChecker extends JFrame implements WetterChecker {
    private static final long serialVersionUID = 1L;
    private final JLabel lbl = new JLabel("Warte …", SwingConstants.CENTER);

    public WarnungChecker(String name) {
        super("Wetterwarnung " + name);              // ablakcím
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setSize(600, 300);
        setLocationByPlatform(true);                 // a rendszer helyezi el, nem fedik egymást
        lbl.setFont(new Font("Tahoma", Font.BOLD, 22));
        lbl.setOpaque(true);
        add(lbl, BorderLayout.CENTER);
        setVisible(true);
    }
}
```
A komponens legyen **mező** (nem csak a konstruktorban helyi változó), hogy a `refresche`-ből is elérhető legyen.
A `serialVersionUID` azért kell, mert a `JFrame` szerializálható, és nélküle az Eclipse figyelmeztet.
---

## Ablakméret és pozíció
**Mit csinál:**

| Metódus | Mit állít |
|---|---|
| `setSize(w, h)` | csak a méretet |
| `setBounds(x, y, w, h)` | pozíciót és méretet |
| `setLocation(x, y)` | csak a pozíciót |
| `setLocationByPlatform(true)` | a pozíciót az operációs rendszer választja |
| `setLocationRelativeTo(null)` | a képernyő közepére teszi |
---

## `setDefaultCloseOperation`
**Mit csinál:** Megmondja, mi történjen, ha bezárják az ablakot.

| Érték | Hatás |
|---|---|
| `EXIT_ON_CLOSE` | az **egész program** leáll (minden ablak és szál) |
| `DISPOSE_ON_CLOSE` | csak ez az ablak záródik be |
| `HIDE_ON_CLOSE` | csak elrejti (alapértelmezett) |
| `DO_NOTHING_ON_CLOSE` | semmi |
💡 `DISPOSE_ON_CLOSE` esetén a checkert le is kell iratkoztatni (`entfernen`), különben egy láthatatlan ablakot frissítene tovább.
---

## `setOpaque(true)`
**Mit csinál:** A `JLabel` alapból átlátszó. A `setBackground` csak akkor látszik, ha ez be van kapcsolva.
**Példa:**
```java
lbl.setOpaque(true);
lbl.setBackground(Color.RED);
```
---

## `JLabel` HTML-lel
**Mit csinál:** Ha a szöveg `<html>`-lel kezdődik, a `JLabel` HTML-ként jeleníti meg: több sor, félkövér, középre igazítás.
**Példa:**
```java
lbl.setText("<html><center><b>GEFAHR!</b><br>" + daten.toString().replace("\n", "<br>") + "</center></html>");
```
---

## `JOptionPane.showMessageDialog(...)`
**Mit csinál:** Felugró üzenetablak. **Modális:** a hívó szál vár, amíg be nem zárják.
**Példa:**
```java
JOptionPane.showMessageDialog(null, "Text", "Titel", JOptionPane.WARNING_MESSAGE);
```
Típusok: `INFORMATION_MESSAGE`, `WARNING_MESSAGE`, `ERROR_MESSAGE`, `PLAIN_MESSAGE`.
⚠️ Másodpercenként hívva ablakáradat lesz. Figyelmeztetéshez jobb egy frissülő ablak, vagy csak változáskor szólni.
---

## EDT és `SwingUtilities.invokeLater(...)`
**Mit csinál:** A Swingnek saját szála van, az **EDT** (Event Dispatch Thread). Szabály: minden GUI-művelet ezen fusson. Az `invokeLater` egy feladatot tesz az EDT sorába, és **azonnal visszatér**, így a hívó szál (pl. a station) nem blokkolódik.
**Példa:**
```java
// 1) kiértékelés a station szálán (gyors, GUI nélkül)
String titel = temp > 30 ? "HITZEWELLE" : "Alles in Ordnung";
// 2) megjelenítés az EDT-n
SwingUtilities.invokeLater(() -> {
    lbl.setText(titel);
    lbl.setBackground(Color.RED);
});
```
```
Station-szál:  messen → bescheid → refresche → invokeLater(...) → azonnal tovább
EDT:                                               └──► setText / setBackground
```
💡 Szigorúan véve az ablakot is az EDT-n illik létrehozni: `SwingUtilities.invokeLater(() -> new WarnungChecker("X"));`
---

## Effectively final (lambdában)
**Mit csinál:** Egy lambda csak olyan helyi változót használhat, amely csak egyszer kap értéket (akkor is, ha nincs kiírva a `final`). Ha minden `if`/`else` ág pontosan egyszer ad értéket, az is rendben van.
**Példa:**
```java
String titel;
if (temp > 30) titel = "Hitze"; else titel = "OK";      // ✅ egyszer kap értéket
SwingUtilities.invokeLater(() -> lbl.setText(titel));

String t = "A"; t = "B";
SwingUtilities.invokeLater(() -> lbl.setText(t));        // ❌ fordítási hiba
```
---

# PROJEKT: WETTERSTATION (TELJES KÓD)

Observer push-változat DTO-val, szálakkal és Swing-ablakkal. Package: `wetterStationObserver`.

| Szerep | Osztály |
|---|---|
| Subject (interface) | `WetterList` |
| ConcreteSubject | `WetterStation` (`extends Thread`) |
| Observer (interface) | `WetterChecker` |
| ConcreteObserver | `KonsoleAnzeige`, `WarnungChecker` |
| DTO | `WetterDaten` |
| Client | `StarteWetterbericht` |

**`Status.java`, `WindRichtung.java`**
```java
public enum Status { BEWOELKT, SONNIG, REGEN, NEBEL, GEMISCHT, GEWITTER, SCHNEE }
public enum WindRichtung { NORD, SUED, OST, WEST, SUED_WEST, SUED_OST, NORD_WEST, NORD_OST }
```

**`WetterChecker.java`**
```java
@FunctionalInterface
public interface WetterChecker {
    void refresche(WetterDaten daten);
}
```

**`WetterList.java`**
```java
public interface WetterList {
    List<WetterChecker> getList();

    default void hinzufuegen(WetterChecker subscriber) { getList().add(subscriber); }
    default void entfernen(WetterChecker subscriber)   { getList().remove(subscriber); }
    default void bescheid(WetterDaten daten) {
        for (WetterChecker subscriber : getList()) {
            subscriber.refresche(daten);
        }
    }
}
```

**`WetterDaten.java`**
```java
public class WetterDaten {
    private static final DateTimeFormatter FORMAT =
            DateTimeFormatter.ofPattern("yyyy. MMMM dd. E 'um' HH:mm:ss", Locale.GERMAN);

    private final String name;
    private final LocalDateTime zeit;
    private final double temperatur;
    private final int luftFeuchtigkeit;
    private final WindRichtung windRichtung;
    private final int windStaerke;
    private final Status status;

    public WetterDaten(String name, LocalDateTime zeit, double temperatur, int luftFeuchtigkeit,
                       WindRichtung windRichtung, int windStaerke, Status status) {
        this.name = name;
        this.zeit = zeit;
        this.temperatur = temperatur;
        this.luftFeuchtigkeit = luftFeuchtigkeit;
        this.windRichtung = windRichtung;
        this.windStaerke = windStaerke;
        this.status = status;
    }

    public String getName() { return name; }
    public LocalDateTime getZeit() { return zeit; }
    public double getTemperatur() { return temperatur; }
    public int getLuftFeuchtigkeit() { return luftFeuchtigkeit; }
    public WindRichtung getWindRichtung() { return windRichtung; }
    public int getWindStaerke() { return windStaerke; }
    public Status getStatus() { return status; }

    @Override
    public String toString() {
        return "Wetter in " + name + ":\n"
             + zeit.format(FORMAT) + "\n"
             + String.format("%.1f °C, %d %%, %s %d Bft, %s",
                             temperatur, luftFeuchtigkeit, windRichtung, windStaerke, status);
    }
}
```

**`WetterStation.java`**
```java
public class WetterStation extends Thread implements WetterList {
    private final List<WetterChecker> checkers;
    private static final List<String> alleNamen = new ArrayList<>();
    private final int sekunden;

    public WetterStation(String name, int sekunden) {
        super(name);
        alleNamen.add(name);
        this.checkers = new ArrayList<>();
        this.sekunden = sekunden;
    }

    public WetterStation(String name) { this(name, 1); }

    @Override
    public List<WetterChecker> getList() { return checkers; }

    public static List<String> getAlleNamen() { return List.copyOf(alleNamen); }

    public void messen() {
        ThreadLocalRandom rnd = ThreadLocalRandom.current();
        double t = rnd.nextDouble(-15, 35);
        int f = rnd.nextInt(0, 101);
        WindRichtung[] alleWR = WindRichtung.values();
        WindRichtung wr = alleWR[rnd.nextInt(0, alleWR.length)];
        int ws = rnd.nextInt(0, 11);
        Status[] alleS = Status.values();
        Status s = alleS[rnd.nextInt(0, alleS.length)];
        bescheid(new WetterDaten(getName(), LocalDateTime.now(), t, f, wr, ws, s));
    }

    @Override
    public void run() {
        while (true) {
            messen();
            try {
                Thread.sleep(sekunden * 1000L);
            } catch (InterruptedException e) {
                System.out.println(getName() + " wird beendet.");
                return;
            }
        }
    }
}
```

**`KonsoleAnzeige.java`**
```java
public class KonsoleAnzeige implements WetterChecker {
    @Override
    public void refresche(WetterDaten daten) {
        System.out.println("WETTERBERICHT\n" + daten + "\n" + "*".repeat(60) + "\n");   // egy println = szálbiztos kiírás
    }
}
```

**`WarnungChecker.java`**
```java
public class WarnungChecker extends JFrame implements WetterChecker {
    private static final long serialVersionUID = 1L;
    private final JLabel lblWarnung = new JLabel("Warte auf Daten …", SwingConstants.CENTER);

    public WarnungChecker(String name) {
        super("Wetterwarnung " + name);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setSize(600, 300);
        setLocationByPlatform(true);
        lblWarnung.setFont(new Font("Tahoma", Font.BOLD, 22));
        lblWarnung.setOpaque(true);
        add(lblWarnung, BorderLayout.CENTER);
        setVisible(true);
    }

    @Override
    public void refresche(WetterDaten daten) {
        String titel;
        Color farbe;
        if (daten.getTemperatur() > 30)               { titel = "GEFAHR! HITZEWELLE"; farbe = Color.RED; }
        else if (daten.getTemperatur() < 0)           { titel = "GEFAHR! FROST";      farbe = Color.CYAN; }
        else if (daten.getStatus() == Status.GEWITTER){ titel = "STARKES GEWITTER";   farbe = Color.ORANGE; }
        else                                          { titel = "Alles in Ordnung";   farbe = Color.GREEN; }

        String text = "<html><center><b>" + titel + "</b><br><br>"
                    + daten.toString().replace("\n", "<br>") + "</center></html>";

        SwingUtilities.invokeLater(() -> {
            lblWarnung.setText(text);
            lblWarnung.setBackground(farbe);
        });
    }
}
```

**`StarteWetterbericht.java`**
```java
public class StarteWetterbericht {
    public static void main(String[] args) {
        WetterStation munich = new WetterStation("München", 2);
        WetterStation vs = new WetterStation("Villingen-Schwenningen", 3);
        WetterChecker konsole = new KonsoleAnzeige();

        munich.hinzufuegen(konsole);                                  // közös konzol
        vs.hinzufuegen(konsole);
        munich.hinzufuegen(new WarnungChecker(munich.getName()));     // saját ablak
        vs.hinzufuegen(new WarnungChecker(vs.getName()));

        System.out.println(WetterStation.getAlleNamen());

        munich.start();
        vs.start();
    }
}
```
💡 Ha a konzolon `?` jelenik meg a `°` helyett: Eclipse → Run Configurations → Common → Encoding: UTF-8.
---
