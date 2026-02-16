# Epic: Migrate Hystrix from Java 8 to Java 11

## Epic Description

Upgrade the entire Hystrix codebase, build system, CI/CD pipelines, and documentation from Java 8 to Java 11. This migration ensures continued support on a modern, long-term-support (LTS) JDK, enables access to Java 9+ language features and APIs, and addresses dependency compatibility requirements for Java 11 bytecode support.

**Repository:** `elliotthomson/Hystrix`
**Epic Owner:** TBD
**Target Completion:** TBD
**Priority:** High

### Scope

- All GitHub Actions CI/CD workflows
- Gradle build configuration across all modules
- Critical dependency upgrades (AspectJ, Spring, Clojure, ASM, etc.)
- Verification of all remaining dependencies against Java 11
- Full test suite validation including AspectJ weaving tests
- Documentation updates

### Success Criteria

- All modules compile and pass tests on Java 11
- CI pipelines build and test against Java 11
- Publish workflows produce artifacts built with Java 11
- No regressions in existing functionality
- All documentation reflects the new Java 11 requirement

### Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| AspectJ weaving breaks on Java 11 bytecode | High | Critical | Upgrade AspectJ to 1.9.7+ before any JDK changes |
| Spring 4.x incompatible with Java 11 | High | High | Upgrade to Spring 5.x in same phase as AspectJ |
| RxJava 1.x issues with Java 11 module system | Medium | High | Test thoroughly; add `--add-opens` JVM args if needed |
| Nebula/Gradle plugin incompatibilities | Medium | Medium | Test build pipeline early; upgrade plugins as needed |
| Clojure tooling breaks on Java 11 | Medium | Medium | Upgrade Clojure and Nebula Clojure plugin |
| Illegal reflective access warnings at runtime | High | Low | Suppress initially; address root causes in later phases |

---

## Critical Path

The following tasks **must** be completed in sequence:

```
Phase 1 (Dependencies) → Phase 2 (Build Config) → Phase 3 (CI/CD) → Phase 4 (Testing) → Phase 5 (Documentation) → Phase 6 (Release)
```

Within each phase, tasks marked **"parallelizable"** can be worked on simultaneously.

---

## Phase 1: Critical Dependency Upgrades

> **Goal:** Upgrade all dependencies that are known to be incompatible with Java 11 bytecode, while still building on Java 8. This ensures a safe, incremental migration.

---

### Task 1.1: Upgrade AspectJ to 1.9.7+

**Priority:** Critical
**Parallelizable:** No (must be completed first in Phase 1)
**Estimated Effort:** 1 day

#### Description

AspectJ 1.8.6 does not support Java 11 bytecode. The AspectJ compiler (ajc) and weaver must be upgraded to at least 1.9.2 (Java 11 bytecode support), with 1.9.7+ recommended for stability and bug fixes.

#### Files to Modify

| File | Line(s) | Current Value | Target Value |
|------|---------|---------------|--------------|
| `hystrix-contrib/hystrix-javanica/build.gradle` | 76 | `aspectjVersion = '1.8.6'` | `aspectjVersion = '1.9.7'` |

#### Affected Dependencies (lines 91-94 of same file)

- `org.aspectj:aspectjtools:$aspectjVersion` (line 91)
- `org.aspectj:aspectjrt:$aspectjVersion` (lines 92, 94)
- `org.aspectj:aspectjweaver:$aspectjVersion` (line 93)

#### Acceptance Criteria

- [ ] `aspectjVersion` updated to `1.9.7` (or latest stable 1.9.x) in `hystrix-contrib/hystrix-javanica/build.gradle`
- [ ] `compileAjcTestJava` task compiles successfully
- [ ] `compileAjcJava` task compiles successfully
- [ ] `ajcTest` test task passes (lines 63-72 of `hystrix-contrib/hystrix-javanica/build.gradle`)
- [ ] Standard `test` task in `hystrix-javanica` still passes
- [ ] No new deprecation warnings introduced by AspectJ upgrade

---

### Task 1.2: Upgrade ASM to 7.0+

**Priority:** Critical
**Parallelizable:** Yes (with Task 1.3, 1.4, 1.5 — after Task 1.1)
**Estimated Effort:** 0.5 days

#### Description

ASM 5.0.4 does not support Java 11 class file format (version 55). ASM 7.0+ is required for Java 11 bytecode reading/writing. Note: AspectJ 1.9.7 bundles a compatible ASM version internally, but the explicit ASM dependency in `hystrix-javanica` must also be upgraded.

#### Files to Modify

| File | Line | Current Value | Target Value |
|------|------|---------------|--------------|
| `hystrix-contrib/hystrix-javanica/build.gradle` | 99 | `implementation 'org.ow2.asm:asm:5.0.4'` | `implementation 'org.ow2.asm:asm:9.4'` |

#### Acceptance Criteria

- [ ] ASM dependency updated to `9.4` (or latest stable) in `hystrix-contrib/hystrix-javanica/build.gradle`
- [ ] No class file version conflicts between ASM and AspectJ
- [ ] `hystrix-javanica` compiles and tests pass

---

### Task 1.3: Upgrade Spring Framework to 5.3.x (Test Dependency)

**Priority:** High
**Parallelizable:** Yes (with Task 1.2, 1.4, 1.5 — after Task 1.1)
**Estimated Effort:** 1 day

#### Description

Spring Framework 4.3.2.RELEASE does not officially support Java 11. Spring 5.x is required for full Java 11 compatibility. Since Spring is used only as a **test dependency** in `hystrix-javanica`, the blast radius is limited to the test suite. Spring 5.3.x is the latest 5.x line and provides Java 11+ support.

#### Files to Modify

| File | Line | Current Value | Target Value |
|------|------|---------------|--------------|
| `hystrix-contrib/hystrix-javanica/build.gradle` | 77 | `springframeworkVesion = '4.3.2.RELEASE'` | `springframeworkVesion = '5.3.30'` |

Note: the variable name contains a typo (`Vesion` instead of `Version`). Preserving the existing name to minimize diff unless a separate cleanup task is created.

#### Affected Dependencies (lines 103-106 of same file)

- `org.springframework:spring-core:$springframeworkVesion` (line 103)
- `org.springframework:spring-context:$springframeworkVesion` (line 104)
- `org.springframework:spring-aop:$springframeworkVesion` (line 105)
- `org.springframework:spring-test:$springframeworkVesion` (line 106)

#### Acceptance Criteria

- [ ] Spring version updated to `5.3.30` (or latest 5.3.x) in `hystrix-contrib/hystrix-javanica/build.gradle`
- [ ] All `hystrix-javanica` unit tests pass
- [ ] All `hystrix-javanica` `ajcTest` tests pass
- [ ] No incompatibilities between Spring 5.3.x and the AspectJ version used
- [ ] Verify `cglib:cglib:3.1` (line 107) is still compatible or upgrade if needed

---

### Task 1.4: Upgrade Clojure to 1.10.x

**Priority:** Medium
**Parallelizable:** Yes (with Task 1.2, 1.3, 1.5 — after Task 1.1)
**Estimated Effort:** 0.5 days

#### Description

Clojure 1.7.0 has known issues with Java 11 module system changes. Clojure 1.10.x provides full Java 11 support and is the recommended minimum for Java 11+ environments. Also evaluate whether the Nebula Clojure plugin (`13.0.1`) needs an upgrade.

#### Files to Modify

| File | Line | Current Value | Target Value |
|------|------|---------------|--------------|
| `hystrix-contrib/hystrix-clj/build.gradle` | 22 | `implementation 'org.clojure:clojure:1.7.0'` | `implementation 'org.clojure:clojure:1.10.3'` |

#### Additional Investigation

- Nebula Clojure Plugin `13.0.1` (line 10 of same file): verify compatibility with Java 11 and Clojure 1.10.x. Upgrade if necessary.
- `org.clojure:tools.nrepl:0.2.1` (line 42): verify still works or replace with `nrepl/nrepl` (the tools.nrepl artifact was moved).

#### Acceptance Criteria

- [ ] Clojure version updated to `1.10.3` in `hystrix-contrib/hystrix-clj/build.gradle`
- [ ] `hystrix-clj` module compiles successfully
- [ ] `hystrix-clj` module tests pass
- [ ] Nebula Clojure plugin compatibility verified (or upgraded)
- [ ] nREPL task still functions (or documented as deprecated)

---

### Task 1.5: Verify and Upgrade Remaining Dependencies

**Priority:** Low
**Parallelizable:** Yes (with Task 1.2, 1.3, 1.4 — after Task 1.1)
**Estimated Effort:** 1 day

#### Description

Verify that all remaining dependencies work on Java 11. Upgrade where necessary.

#### Dependencies to Verify

| Dependency | Current Version | File | Line | Java 11 Compatible? | Recommended Action |
|------------|----------------|------|------|---------------------|-------------------|
| `com.netflix.archaius:archaius-core` | `0.4.1` | `hystrix-core/build.gradle` | 4 | Likely — verify | Test; upgrade if issues found |
| `io.reactivex:rxjava` | `1.2.0` | `hystrix-core/build.gradle` | 5 | Yes (1.2.0+ works) | Test; consider `1.3.8` (latest 1.x) |
| `org.slf4j:slf4j-api` | `1.7.0` | `hystrix-core/build.gradle` | 6 | Yes | Optionally upgrade to `1.7.36` |
| `org.hdrhistogram:HdrHistogram` | `2.1.9` | `hystrix-core/build.gradle` | 7 | Yes | Optionally upgrade to `2.1.12` |
| `com.google.guava:guava` | `15.0` | `hystrix-contrib/hystrix-javanica/build.gradle` | 96 | Likely — verify | Test; consider upgrading to `31.1-jre` |
| `org.apache.commons:commons-lang3` | `3.1` | `hystrix-contrib/hystrix-javanica/build.gradle` | 97 | Yes | Optionally upgrade to `3.12.0` |
| `com.google.code.findbugs:jsr305` | `2.0.0` | `hystrix-contrib/hystrix-javanica/build.gradle` | 98 | Yes | No action needed |
| `me.champeau.jmh` (plugin) | `0.7.1` | `build.gradle` | 3 | Yes | No action needed |
| JMH | `1.15` | `hystrix-core/build.gradle` | 30 | Likely — verify | Consider upgrading to `1.36` |

#### Acceptance Criteria

- [ ] Each dependency in the table above verified against Java 11
- [ ] Dependencies with known issues upgraded to compatible versions
- [ ] Full build (`./gradlew build`) passes with all dependency changes
- [ ] No new runtime warnings related to dependencies
- [ ] Findings documented in a compatibility matrix (can be a comment on this Jira ticket)

---

## Phase 2: Build Configuration Updates

> **Goal:** Update all Gradle build files and configurations to target Java 11 compilation and bytecode.

---

### Task 2.1: Update Root Build Configuration for Java 11

**Priority:** Critical
**Parallelizable:** No (must be completed before Task 2.2 and 2.3)
**Estimated Effort:** 0.5 days

#### Description

Configure the root `build.gradle` to set Java 11 as the source and target compatibility for all subprojects.

#### Files to Modify

| File | Line(s) | Change |
|------|---------|--------|
| `build.gradle` | 12-21 (subprojects block) | Add `sourceCompatibility = JavaVersion.VERSION_11` and `targetCompatibility = JavaVersion.VERSION_11` |

#### Suggested Change

```groovy
subprojects {
    apply plugin: 'nebula.netflixoss'
    apply plugin: 'java-library'

    group = "com.netflix.hystrix"

    sourceCompatibility = JavaVersion.VERSION_11
    targetCompatibility = JavaVersion.VERSION_11

    tasks.withType(Javadoc).configureEach {
        failOnError = false
    }
}
```

#### Acceptance Criteria

- [ ] `sourceCompatibility` set to `JavaVersion.VERSION_11` in root `build.gradle`
- [ ] `targetCompatibility` set to `JavaVersion.VERSION_11` in root `build.gradle`
- [ ] `./gradlew compileJava` succeeds for all modules
- [ ] Generated `.class` files have major version 55 (Java 11)

---

### Task 2.2: Update AspectJ Compile Tasks for Java 11

**Priority:** Critical
**Parallelizable:** Yes (with Task 2.3 — after Task 2.1)
**Estimated Effort:** 0.5 days

#### Description

Update the hardcoded `source` and `target` values in the AspectJ `iajc` Ant tasks in `hystrix-javanica`.

#### Files to Modify

| File | Line | Current Value | Target Value |
|------|------|---------------|--------------|
| `hystrix-contrib/hystrix-javanica/build.gradle` | 30 | `ant.iajc(source: "1.8", target: "1.8",` | `ant.iajc(source: "11", target: "11",` |
| `hystrix-contrib/hystrix-javanica/build.gradle` | 48 | `ant.iajc(source: "${sourceCompatibility}", target: "${targetCompatibility}",` | No change needed (inherits from root) |

Line 48 uses `${sourceCompatibility}` and `${targetCompatibility}`, which will automatically pick up the values from Task 2.1. However, line 30 has **hardcoded** values that must be changed manually.

#### Acceptance Criteria

- [ ] Line 30 of `hystrix-contrib/hystrix-javanica/build.gradle` updated to `source: "11", target: "11"`
- [ ] `compileAjcTestJava` task succeeds with Java 11 source/target
- [ ] `compileAjcJava` task succeeds with Java 11 source/target
- [ ] AspectJ weaving produces valid Java 11 bytecode
- [ ] `ajcTest` tests pass

---

### Task 2.3: Add JVM Arguments for Java 11 Module System

**Priority:** High
**Parallelizable:** Yes (with Task 2.2 — after Task 2.1)
**Estimated Effort:** 0.5 days

#### Description

Java 11 enforces the module system more strictly than Java 8. Some libraries (especially those using reflection) may need `--add-opens` or `--add-exports` JVM arguments. These need to be added to test tasks and potentially the JMH benchmark configuration.

#### Files to Potentially Modify

| File | Section | Potential Change |
|------|---------|-----------------|
| `build.gradle` | `subprojects` block | Add `--add-opens` args to all `Test` tasks |
| `hystrix-core/build.gradle` | `jmh` block (lines 27-36) | Add JVM args for JMH benchmarks |
| `hystrix-contrib/hystrix-javanica/build.gradle` | `ajcTest` task (lines 63-70) | Add `--add-opens` args |

#### Investigation Steps

1. Run `./gradlew build` on Java 11 and collect all `InaccessibleObjectException` and illegal reflective access warnings
2. Determine which `--add-opens` directives are needed
3. Add them to the appropriate tasks

#### Suggested Pattern

```groovy
// In root build.gradle, subprojects block:
tasks.withType(Test).configureEach {
    jvmArgs += [
        '--add-opens', 'java.base/java.lang=ALL-UNNAMED',
        '--add-opens', 'java.base/java.lang.reflect=ALL-UNNAMED',
        // Add more as discovered during testing
    ]
}
```

#### Acceptance Criteria

- [ ] All illegal reflective access errors resolved
- [ ] All test tasks pass without `InaccessibleObjectException`
- [ ] JMH benchmarks run without module system errors
- [ ] JVM arguments documented in comments for maintainability
- [ ] No unnecessary `--add-opens` directives (only add what is needed)

---

## Phase 3: CI/CD Pipeline Updates

> **Goal:** Update all GitHub Actions workflows to build, test, and publish with Java 11.

---

### Task 3.1: Update CI Workflow (nebula-ci.yml)

**Priority:** Critical
**Parallelizable:** Yes (with Task 3.2 and 3.3)
**Estimated Effort:** 0.5 days

#### Description

Update the CI workflow to test against Java 11 instead of Java 8.

#### Files to Modify

| File | Line(s) | Current Value | Target Value |
|------|---------|---------------|--------------|
| `.github/workflows/nebula-ci.yml` | 15 | `# test against JDK 8` | `# test against JDK 11` |
| `.github/workflows/nebula-ci.yml` | 16 | `java: [ 8  ]` | `java: [ 11 ]` |

#### Acceptance Criteria

- [ ] CI workflow matrix updated to Java 11
- [ ] CI build passes on Java 11
- [ ] All test suites pass in CI environment
- [ ] Build cache keys remain valid (no changes needed to cache configuration)

---

### Task 3.2: Update Publish Workflow (nebula-publish.yml)

**Priority:** Critical
**Parallelizable:** Yes (with Task 3.1 and 3.3)
**Estimated Effort:** 0.5 days

#### Description

Update the release/candidate publish workflow to build with Java 11.

#### Files to Modify

| File | Line(s) | Current Value | Target Value |
|------|---------|---------------|--------------|
| `.github/workflows/nebula-publish.yml` | 17 | `- name: Setup jdk 8` | `- name: Setup jdk 11` |
| `.github/workflows/nebula-publish.yml` | 20 | `java-version: 1.8` | `java-version: 11` |

#### Acceptance Criteria

- [ ] Publish workflow updated to Java 11
- [ ] Workflow syntax validates (via `act` or GitHub Actions linter)
- [ ] Published artifacts target Java 11 bytecode
- [ ] Signing and publishing steps unaffected by JDK change

---

### Task 3.3: Update Snapshot Workflow (nebula-snapshot.yml)

**Priority:** Critical
**Parallelizable:** Yes (with Task 3.1 and 3.2)
**Estimated Effort:** 0.5 days

#### Description

Update the snapshot publish workflow to build with Java 11.

#### Files to Modify

| File | Line(s) | Current Value | Target Value |
|------|---------|---------------|--------------|
| `.github/workflows/nebula-snapshot.yml` | 16 | `- name: Set up JDK` | `- name: Set up JDK 11` (optional rename) |
| `.github/workflows/nebula-snapshot.yml` | 19 | `java-version: 8` | `java-version: 11` |

#### Acceptance Criteria

- [ ] Snapshot workflow updated to Java 11
- [ ] Snapshot build and publish succeeds on Java 11
- [ ] Workflow syntax validates

---

## Phase 4: Comprehensive Testing

> **Goal:** Validate the entire migration through systematic testing across all modules.

---

### Task 4.1: Run Full Test Suite on Java 11

**Priority:** Critical
**Parallelizable:** No (depends on Phases 1-3 completion)
**Estimated Effort:** 1-2 days

#### Description

Execute the full test suite across all modules on Java 11 and fix any failures.

#### Test Commands

```bash
# Full build with tests
./gradlew clean build

# Individual module tests if debugging failures
./gradlew :hystrix-core:test
./gradlew :hystrix-javanica:test
./gradlew :hystrix-javanica:ajcTest
./gradlew :hystrix-clj:test
./gradlew :hystrix-examples:build
./gradlew :hystrix-serialization:test
```

#### Key Areas to Verify

1. **hystrix-core**: Core command execution, circuit breaker, thread pool isolation, request collapsing, metrics
2. **hystrix-javanica**: Annotation processing, AspectJ weaving (both `test` and `ajcTest` tasks), Spring integration tests
3. **hystrix-clj**: Clojure wrapper functionality
4. **hystrix-serialization**: Serialization/deserialization
5. **All contrib modules**: Metrics publishers, event streams, servlet integration

#### Acceptance Criteria

- [ ] `./gradlew clean build` passes with zero test failures
- [ ] `ajcTest` task passes (AspectJ compile-time weaving tests)
- [ ] No illegal reflective access warnings in test output (or all documented and suppressed)
- [ ] Test execution times are within expected ranges (no significant regressions)
- [ ] All modules listed in `settings.gradle` build successfully

---

### Task 4.2: Validate AspectJ Weaving Behavior

**Priority:** High
**Parallelizable:** Yes (with Task 4.3 — after Task 4.1)
**Estimated Effort:** 1 day

#### Description

AspectJ weaving is the highest-risk area of this migration. Perform focused validation of both compile-time weaving (CTW) and the AspectJ test suite.

#### Files of Interest

| File | Description |
|------|-------------|
| `hystrix-contrib/hystrix-javanica/build.gradle` lines 26-41 | `compileAjcTestJava` task — CTW for test sources |
| `hystrix-contrib/hystrix-javanica/build.gradle` lines 44-61 | `compileAjcJava` task — CTW for main sources |
| `hystrix-contrib/hystrix-javanica/build.gradle` lines 63-70 | `ajcTest` task definition |
| `hystrix-contrib/hystrix-javanica/build.gradle` lines 81-84 | `ajcJar` task — CTW JAR artifact |
| `hystrix-contrib/hystrix-javanica/src/ajcTest/` | AspectJ-specific test sources |

#### Validation Steps

1. Run `./gradlew :hystrix-javanica:compileAjcJava` and verify the CTW JAR is produced
2. Run `./gradlew :hystrix-javanica:compileAjcTestJava` and verify no weaving errors
3. Run `./gradlew :hystrix-javanica:ajcTest` and verify all tests pass
4. Inspect woven bytecode with `javap` to confirm Java 11 class version (55.0)
5. Verify that `@HystrixCommand` annotation interception works correctly

#### Acceptance Criteria

- [ ] `compileAjcJava` produces a valid CTW JAR
- [ ] `compileAjcTestJava` completes without errors
- [ ] All `ajcTest` tests pass
- [ ] Woven class files target Java 11 (class file version 55)
- [ ] No AspectJ-related runtime errors

---

### Task 4.3: Validate JMH Benchmarks

**Priority:** Medium
**Parallelizable:** Yes (with Task 4.2 — after Task 4.1)
**Estimated Effort:** 0.5 days

#### Description

Ensure JMH benchmarks in `hystrix-core` compile and run on Java 11.

#### Files of Interest

| File | Line(s) | Description |
|------|---------|-------------|
| `hystrix-core/build.gradle` | 27-36 | JMH configuration |
| `build.gradle` | 3 | JMH plugin (`me.champeau.jmh` version `0.7.1`) |

#### Acceptance Criteria

- [ ] `./gradlew :hystrix-core:jmh` runs without errors on Java 11
- [ ] Benchmark results are reasonable (no anomalies indicating JVM issues)
- [ ] JMH forked processes use Java 11

---

## Phase 5: Documentation Updates

> **Goal:** Update all documentation to reflect the new Java 11 requirement.

---

### Task 5.1: Update README.md Java Version Requirement

**Priority:** Medium
**Parallelizable:** Yes (with Task 5.2)
**Estimated Effort:** 15 minutes

#### Description

Update the minimum Java version referenced in the root README.

#### Files to Modify

| File | Line | Current Value | Target Value |
|------|------|---------------|--------------|
| `README.md` | 140 | `You need Java 6 or later.` | `You need Java 11 or later.` |

#### Acceptance Criteria

- [ ] README.md line 140 updated to reference Java 11
- [ ] No other stale Java version references in README.md

---

### Task 5.2: Update Javanica README Examples

**Priority:** Low
**Parallelizable:** Yes (with Task 5.1)
**Estimated Effort:** 15 minutes

#### Description

The `hystrix-contrib/hystrix-javanica/README.md` contains Java 8 lambda/stream examples with comments labelled `// hava 8`. These should be reviewed and updated.

#### Files of Interest

| File | Line(s) | Description |
|------|---------|-------------|
| `hystrix-contrib/hystrix-javanica/README.md` | 755 | Comment `// hava 8` (typo for "java 8") |
| `hystrix-contrib/hystrix-javanica/README.md` | 771 | Comment `// hava 8` (typo for "java 8") |

#### Acceptance Criteria

- [ ] Java 8 specific comments reviewed; update or remove version-specific labels
- [ ] Code examples remain valid Java 11 code (Java 8 lambdas/streams are fully compatible with Java 11)
- [ ] Fix typo `hava` → `java` if modifying these lines

---

## Phase 6: Release Preparation

> **Goal:** Finalize the migration and prepare for a release.

---

### Task 6.1: Update CHANGELOG.md

**Priority:** Medium
**Parallelizable:** Yes (with Task 6.2)
**Estimated Effort:** 15 minutes

#### Description

Add an entry to `CHANGELOG.md` documenting the Java 8 → Java 11 migration, including all dependency upgrades and breaking changes.

#### Key Points to Document

- Minimum Java version changed from 8 to 11
- AspectJ upgraded from 1.8.6 to 1.9.7+
- Spring test dependency upgraded from 4.3.2 to 5.3.x
- Clojure upgraded from 1.7.0 to 1.10.x
- ASM upgraded from 5.0.4 to 9.4
- Any `--add-opens` JVM arguments required for testing/runtime
- List of all other dependency version changes

#### Acceptance Criteria

- [ ] CHANGELOG.md updated with a new section for the Java 11 migration
- [ ] All dependency version changes listed
- [ ] Breaking changes clearly documented
- [ ] Migration notes for downstream consumers included

---

### Task 6.2: Perform Final Integration Validation

**Priority:** Critical
**Parallelizable:** Yes (with Task 6.1)
**Estimated Effort:** 1 day

#### Description

Perform a final end-to-end validation by running the complete build, test suite, example application, and a dry-run of the publish process.

#### Validation Steps

```bash
# Clean build with all tests
./gradlew clean build

# Run the demo application
./gradlew runDemo

# Run the examples webapp (if applicable)
cd hystrix-examples-webapp && ../gradlew appRun

# Dry-run publish (verify artifacts are correct)
./gradlew publishToMavenLocal
```

#### Acceptance Criteria

- [ ] `./gradlew clean build` passes
- [ ] Demo application runs correctly
- [ ] Published artifacts in local Maven repo target Java 11
- [ ] JAR manifests contain correct metadata
- [ ] No unexpected files or changes in the build output

---

## Summary: Task Dependency Graph

```
Phase 1 (Dependencies)
├── Task 1.1: Upgrade AspectJ [CRITICAL] ──────────────────────┐
│   ├── Task 1.2: Upgrade ASM [CRITICAL] (parallel)           │
│   ├── Task 1.3: Upgrade Spring [HIGH] (parallel)            │
│   ├── Task 1.4: Upgrade Clojure [MEDIUM] (parallel)         │
│   └── Task 1.5: Verify other deps [LOW] (parallel)          │
│                                                               │
Phase 2 (Build Configuration) ◄─────────────────────────────────┘
├── Task 2.1: Root build config [CRITICAL] ────────────────────┐
│   ├── Task 2.2: AspectJ compile tasks [CRITICAL] (parallel) │
│   └── Task 2.3: JVM module args [HIGH] (parallel)           │
│                                                               │
Phase 3 (CI/CD) ◄──────────────────────────────────────────────┘
├── Task 3.1: Update CI workflow [CRITICAL] (parallel)
├── Task 3.2: Update publish workflow [CRITICAL] (parallel)
└── Task 3.3: Update snapshot workflow [CRITICAL] (parallel)
│
Phase 4 (Testing) ◄────────────────────────────────────────────
├── Task 4.1: Full test suite [CRITICAL]
│   ├── Task 4.2: AspectJ weaving validation [HIGH] (parallel)
│   └── Task 4.3: JMH benchmark validation [MEDIUM] (parallel)
│
Phase 5 (Documentation) ◄──────────────────────────────────────
├── Task 5.1: Update README [MEDIUM] (parallel)
└── Task 5.2: Update Javanica README [LOW] (parallel)
│
Phase 6 (Release) ◄────────────────────────────────────────────
├── Task 6.1: Update CHANGELOG [MEDIUM] (parallel)
└── Task 6.2: Final integration validation [CRITICAL] (parallel)
```

---

## Quick Reference: All Files Requiring Changes

| File | Changes Required |
|------|-----------------|
| `build.gradle` | Add `sourceCompatibility`/`targetCompatibility` Java 11 |
| `hystrix-core/build.gradle` | Verify/upgrade dependency versions |
| `hystrix-contrib/hystrix-javanica/build.gradle` | AspectJ, ASM, Spring, Guava upgrades; update `iajc` source/target |
| `hystrix-contrib/hystrix-clj/build.gradle` | Clojure upgrade; verify Nebula plugin |
| `.github/workflows/nebula-ci.yml` | Java 8 → 11 in matrix |
| `.github/workflows/nebula-publish.yml` | Java 1.8 → 11 |
| `.github/workflows/nebula-snapshot.yml` | Java 8 → 11 |
| `README.md` | Java 6 → Java 11 requirement |
| `hystrix-contrib/hystrix-javanica/README.md` | Update Java 8 example comments |
| `CHANGELOG.md` | Add migration entry |

---

## Jira Import Notes

- **Epic:** "Migrate Hystrix from Java 8 to Java 11"
- **Each Task (1.1 through 6.2)** should be created as a separate Jira Story/Task under the epic
- **Labels:** `java-migration`, `java-11`, `tech-debt`
- **Components:** Tag each task with the relevant module(s): `hystrix-core`, `hystrix-javanica`, `hystrix-clj`, `ci-cd`, `documentation`
- **Links:** Tasks within the same phase that are parallelizable should be linked as "related to" each other; sequential dependencies should use "is blocked by" links
- **Story Points (suggested):**
  - 0.5 day tasks: 1-2 points
  - 1 day tasks: 3 points
  - 1-2 day tasks: 5 points
  - 15-minute tasks: 0.5 points
- **Total estimated effort:** ~10-12 developer days
