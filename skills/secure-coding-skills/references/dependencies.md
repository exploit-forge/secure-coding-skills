# A03:2025 Software supply chain failures

Pin the resolved version in the lockfile and fail the build when that version is on the deny list below. A label, a comment, or a classifier is not a pin. This skill does not look up new advisories. If a coordinate is not listed here, say "needs an advisory check" and do not invent a CVE.

CWE-1395 (dependency on a vulnerable component).

## A label is not a version

The pin is the version the resolver selected. Review that version. The string `lab-weak` on a dependency, in a comment, or in a classifier does not mark the pin as reviewed and does not make a bad version acceptable.

```xml
<!-- Maven: the version is the pin. This comment does not review it. -->
<dependency>
  <groupId>org.yaml</groupId>
  <artifactId>snakeyaml</artifactId>
  <version>2.2</version>
</dependency>
```

```xml
<!-- lab-weak: accepted -->
<dependency>
  <groupId>org.yaml</groupId>
  <artifactId>snakeyaml</artifactId>
  <version>1.33</version>
</dependency>
```

The same rule covers `requirements.txt`, `package-lock.json`, `packages.lock.json`, and Gradle locks. A floating tag (`latest`, a branch, an unpinned range that can move) is not a pin.

## Deny list from labs we already shipped

Reject these coordinates if they resolve. The reason is the advisory class, not a reproduction.

| Coordinate | Reject | Why | Accept |
| --- | --- | --- | --- |
| `commons-collections:commons-collections` | 3.2.1 and older | CVE-2015-6420, unsafe Java deserialization | 3.2.2 or newer, and do not deserialize untrusted Java objects |
| `org.yaml:snakeyaml` | 1.33 and older | CVE-2022-1471, unrestricted type construction | 2.0 or newer, global tags left off for untrusted YAML |
| `commons-io:commons-io` | 2.11.0 (any release before 2.14.0) | CVE-2024-47554, unbounded work in `XmlStreamReader` | 2.14.0 or newer |
| `Newtonsoft.Json` | 12.0.3 (any release before 13.0.1) | CVE-2024-21907, hostile JSON can exhaust the parser | 13.0.1 or newer, with `TypeNameHandling` left at `None` |

```xml
<dependency>
  <groupId>commons-collections</groupId>
  <artifactId>commons-collections</artifactId>
  <version>3.2.2</version>
</dependency>
```

```xml
<version>3.2.1</version>
```

Wire the deny list into the build (Maven Enforcer, Gradle, `dotnet list package --vulnerable`, `pip-audit`, `npm audit`, or a small version assertion in CI) so a comment cannot override it.

## Where the manifest differs

- Java: `pom.xml` or `build.gradle` plus a lock. Check the resolved version, not only the declared range.
- C#: `PackageReference` plus `packages.lock.json`. `Newtonsoft.Json` 12.0.3 is a reject even when the project compiles.
- Python: pin `==` in the lock. A direct pin of a safe version still fails review if a transitive pin hits the list.
- TypeScript: commit `package-lock.json` or `pnpm-lock.yaml`. The `package.json` range is not the pin.
