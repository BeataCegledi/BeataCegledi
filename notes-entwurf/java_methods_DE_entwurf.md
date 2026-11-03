# INTERFACE UND ABSTRAKTE KLASSE

## `default`-Methode (Interface)
**Was es macht:** Das Schlüsselwort `default` kennzeichnet im Interface, dass die Methode einen Rumpf hat und dies die Standardimplementierung ist. Überschreibt die implementierende Klasse sie nicht, läuft diese Version. Überschreibt sie sie (`@Override`), läuft ihre eigene Version.
**Beispiel:**
```java
interface WetterList {
    List<WetterChecker> getList();        // abstrakt: muss implementiert werden
    default void bescheid() {             // default: fertig, Überschreiben optional
        for (WetterChecker c : getList()) c.refresche();
    }
}
```

⚠️ **Grund für die Verwechslung: Das Wort `default` hat in Java drei Bedeutungen**

| Bedeutung | Wo | Beispiel |
|---|---|---|
| default-Methode | im Interface | `default void bescheid() { … }` |
| default-Zweig | im `switch` | `default: System.out.println("?");` |
| „default“-Zugriff (package-private) | in Klassen | `void m() { … }` (kein Schlüsselwort!) |
---
