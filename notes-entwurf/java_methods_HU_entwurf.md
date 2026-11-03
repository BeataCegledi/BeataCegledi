# INTERFACE ÉS ABSZTRAKT OSZTÁLY

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
---
