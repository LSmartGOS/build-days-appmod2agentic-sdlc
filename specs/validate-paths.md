# Change specification: Validate local-storage object keys

## Metadata

- **Owner:** @LSmartGOS
- **Status:** Approved
- **Source root:** `legacy-src/java-asset-manager`
- **Related issue:** not created

## Problem and outcomes

### Problem

Under the `dev` Spring profile the Java asset manager stores uploads on the
local file system. `LocalFileStorageService` does not reliably keep object keys
inside its storage directory.

- `uploadObject` runs `StringUtils.cleanPath(file.getOriginalFilename())` and
  rejects only names containing `..`. It does not reject absolute paths, drive
  paths, or names with separators, and it does not check that the resolved
  target is inside `rootLocation`. A multipart request with a crafted
  `filename` such as `/tmp/x` or `C:/x` is therefore a suspected way to write
  outside the storage directory (to be confirmed by AC-4).
- `getObject` and `deleteObject` call `rootLocation.resolve(key)` with no
  containment check, so keys such as `..` or an absolute path are not rejected
  by the service itself.
- A null original filename passes `cleanPath` and then fails at
  `filename.contains("..")` (cause to be confirmed; see Unresolved questions).

Affected users are developers and workshop participants who run the `dev`
profile. Whether other machines can reach a dev instance is unknown (see
Assumptions). The non-`dev` `AwsS3Service` is not affected by this problem.

HTTP exposure is partly mitigated: Spring path variables do not match `/`, and
Tomcat rejects encoded slashes by default. This is unverified, so the service
must enforce the rules itself.

### Desired outcomes

1. Unsafe keys and filenames are rejected before any file read, write, or
   delete, with a short plain-language reason.
2. Viewing a rejected key returns the existing 404 response. Delete and upload
   failures show the existing "Failed to ..." flash message followed by the
   reason.
3. Valid single-filename keys behave exactly as before.

### Success measures

- All new unit tests pass in the `web` module.
- No file is created, read, or deleted outside the test's temporary storage
  directory in any test.
- The worker module and its test results are unchanged.

## Functional requirements

| ID | Requirement | Evidence or rationale |
| --- | --- | --- |
| FR-1 | The service shall reject, with an `IOException`, any key or upload filename that is null, empty, or blank. | `LocalFileStorageService.uploadObject`, `getObject`, `deleteObject` in `legacy-src/java-asset-manager/web/src/main/java/com/microsoft/migration/assets/service/LocalFileStorageService.java` |
| FR-2 | The service shall reject any key or upload filename containing `..`, `/`, or `\`. It shall treat all other characters, including `%`, as ordinary filename characters and shall not URL-encode or decode keys. | Owner decision: disallow `..`; the owner clarified that percent-encoded sequences need not be rejected. Keys are flat names: `listObjects` uses `Files.walk(rootLocation, 1)`. Spring decodes `{key}` once before the service sees it, so the service always works on decoded names (see URL encoding finding). |
| FR-3 | The service shall reject any key or filename that is an absolute path or drive-qualified path, or whose resolved, normalized path is not a direct child of `rootLocation`. | `rootLocation.resolve(key)` returns the argument when it is absolute. |
| FR-4 | A rejected upload shall not write a file and shall not call `RabbitTemplate.convertAndSend`. | `LocalFileStorageService.uploadObject` sends the queue message after copying. |
| FR-5 | A rejected key in `deleteObject` shall not delete the original file or any thumbnail. | `LocalFileStorageService.deleteObject` |
| FR-6 | Rejections shall be thrown as `IOException` so existing controller handling is reused: 404 for view, flash text `"Failed to delete file: " + message` for delete, `"Failed to upload file: " + message` for upload. | `S3Controller.viewObject`, `deleteObject`, `uploadObject` in `.../controller/S3Controller.java` |
| FR-7 | The rejection message shall use plain language, shall state why the name is not allowed, and shall not contain the submitted key, the filename, or any file-system path. | Owner decision: explain why in low-tech terms. The key is client-controlled and the flash is user-visible. |
| FR-8 | A safe, existing key shall be read, deleted, and uploaded as before, including deleting its `_thumbnail` companion. | `StorageService.getThumbnailKey`, `LocalFileStorageService.deleteObject` |
| FR-9 | A safe but missing key shall keep the existing `FileNotFoundException("File not found: ...")` behavior. | `LocalFileStorageService.getObject`, `deleteObject` |

## Constraints and out of scope

### Constraints

- **Allowed paths:**
  - `legacy-src/java-asset-manager/web/src/main/java/com/microsoft/migration/assets/service/LocalFileStorageService.java`
  - `legacy-src/java-asset-manager/web/src/test/java/com/microsoft/migration/assets/service/LocalFileStorageServiceTest.java` (new)
- **Primary validation command:** `.\mvnw.cmd -B -ntp -pl web test`, run from `legacy-src/java-asset-manager` (on macOS/Linux use `./mvnw` with the same arguments). It limits the run to the `web` module so the known worker test defect cannot hide a real failure. Whether `-pl web` works without `-am` is unverified and must be confirmed at baseline.
- **Compatibility:** Java 8 syntax, Spring Boot 2.7, and existing public signatures stay unchanged (`legacy-src/java-asset-manager/AGENTS.md`, `.github/instructions/java.instructions.md`). Existing response codes and flash message prefixes are preserved.
- **Dependencies:** No `pom.xml` change. `web/pom.xml` already declares `spring-boot-starter-test` with test scope. No new libraries.
- **Data and security:** Tests use only visibly synthetic filenames in a JUnit `@TempDir`. Tests must never touch `../storage` or any real path. No secrets or credentials are added to code or `application.properties`.

### Exclusions (out of scope)

- Any change to the worker module, including `LocalFileProcessingService` (owner decision: keep the change small and the modules separate).
- Any change to `AwsS3Service`, `S3Controller`, templates, `application.properties`, or any `pom.xml`.
- A controller or MockMvc test (owner decision).
- Symlink resolution (`toRealPath`) and Windows-specific hardening (alternate data streams such as `file.txt:stream`, reserved names such as `CON`, trailing dots or spaces, case-insensitive file systems). Owner decision: out of scope.
- URL-encoding of keys in templates (`list.html`, `view.html`) and in `StorageService.generateUrl`. These are outside the allowed paths; see the URL encoding finding and Unresolved questions.
- Subdirectory support, Java or framework upgrades, refactoring, and cleanup.

## Edge cases

| Type | Condition | Expected behavior |
| --- | --- | --- |
| Normal | Upload, read, and delete of `photo.png` in a temp storage directory. | Works as before. Delete also removes `photo_thumbnail.png` when present. |
| Failure | Key or filename is `..`, `a/b`, `a\b`, `/tmp/x`, `C:/x`, null, or blank. | `IOException` with a safe message. No file operation and no queue message. |
| Failure | Safe key does not exist. | Existing `FileNotFoundException` behavior. |
| Boundary | `a..b.png`, `...`, `.hidden`, a name with spaces. | Any name containing `..` is rejected (including `a..b.png` and `...`). `.hidden` and names with spaces are accepted. |
| Boundary | Name contains a literal `%` sequence such as `100%2Fine.png` (already decoded by Spring). | Accepted and stored literally; no decoding or rejection in the service. Links to it are a template concern (see URL encoding finding). |
| Boundary | Key names a directory inside the root. | Behavior not changed by this work (unverified; see Unresolved questions). |
| Compatibility | Upload filename such as `dir/x.png`, accepted today if `dir` exists. | Now rejected. This is a deliberate behavior change. |
| Compatibility | Existing files with legal names. | Still list, view, and delete as before. |

## Acceptance criteria

### AC-1: Valid upload

- **Given** a `LocalFileStorageService` rooted at a temporary directory and a `MockMultipartFile` named `photo.png` with content
- **When** `uploadObject` is called
- **Then** `photo.png` exists in the root with the same content and `RabbitTemplate.convertAndSend` is called once with `IMAGE_PROCESSING_QUEUE` and a message whose key is `photo.png`
- **Verification:** `LocalFileStorageServiceTest`, run by the primary validation command

### AC-2: Unsafe read keys

- **Given** the same service and an existing file in the root
- **When** `getObject` is called with each of `..`, `a/b`, `a\b`, `a..b.png`, null, and blank
- **Then** an `IOException` is thrown whose message is non-empty and does not contain the submitted key or the root path
- **Verification:** `LocalFileStorageServiceTest`

### AC-3: Unsafe delete keys

- **Given** the same service, a file and thumbnail in the root, and a sentinel file in the root's parent
- **When** `deleteObject` is called with each key from AC-2
- **Then** an `IOException` is thrown and every file, including the sentinel, is unchanged
- **Verification:** `LocalFileStorageServiceTest`

### AC-4: Absolute-path upload filename

- **Given** the same service and a `MockMultipartFile` whose original filename is an absolute path under another temporary directory, and a second one that is drive-qualified (`C:/x`)
- **When** `uploadObject` is called
- **Then** an `IOException` is thrown, no file exists at the absolute target, and `RabbitTemplate` is never called
- **Verification:** `LocalFileStorageServiceTest`. The drive-qualified case is asserted only where it is distinguishable on the current platform (unverified).

### AC-5: Null or blank upload filename

- **Given** the same service and a multipart file with a null original filename, and another with a blank one
- **When** `uploadObject` is called
- **Then** an `IOException` is thrown (not a `NullPointerException`) and `RabbitTemplate` is never called
- **Verification:** `LocalFileStorageServiceTest`

### AC-6: Unsafe upload filenames with separators or `..`

- **Given** the same service
- **When** `uploadObject` is called with filenames `../x.png`, `dir/x.png`, and `dir\x.png`
- **Then** an `IOException` is thrown, no file is written, and `RabbitTemplate` is never called
- **Verification:** `LocalFileStorageServiceTest`

### AC-9: Literal percent sequences are ordinary characters

- **Given** the same service
- **When** `uploadObject` is called with a file named `100%2Fine.png`, then `getObject("100%2Fine.png")` and `deleteObject("100%2Fine.png")` are called
- **Then** the file is stored under that exact literal name, read back with the same content, and deleted, with no `IOException` and no decoding
- **Verification:** `LocalFileStorageServiceTest`

### AC-7: Valid delete with thumbnail

- **Given** the same service with `photo.png` and `photo_thumbnail.png` in the root
- **When** `deleteObject("photo.png")` is called
- **Then** both files are gone
- **Verification:** `LocalFileStorageServiceTest`

### AC-8: Missing safe key

- **Given** the same service and no file named `missing.png`
- **When** `getObject("missing.png")` or `deleteObject("missing.png")` is called
- **Then** a `FileNotFoundException` is thrown
- **Verification:** `LocalFileStorageServiceTest`

## Non-functional requirements

| ID | Quality | Testable requirement | Verification |
| --- | --- | --- | --- |
| NFR-1 | Reliability | Every rejection is an `IOException`. No test observes a `NullPointerException` or other unchecked exception for the inputs in FR-1 to FR-3. | AC-2, AC-3, AC-5 |
| NFR-2 | Security/privacy | Rejection messages never contain the submitted key, the filename, or the root path. | AC-2 assertions on the message text |
| NFR-3 | Maintainability | Only the two allowed files change, with no new dependency. | `git diff --name-only` shows only the allowed paths |
| NFR-4 | Test safety | All tests write only inside `@TempDir`. | Test code review and AC-3 sentinel check |

## Assumptions, dependencies, and risks

| Item | Type | Validation or mitigation |
| --- | --- | --- |
| Spring path variables and Tomcat already block `/` and encoded slashes for view and delete. | Assumption (unverified) | Not relied on. The service enforces FR-2 itself. |
| `StringUtils.cleanPath(null)` returns null in the Spring version managed by Boot 2.7, so the NPE comes from `filename.contains("..")`. | Assumption (unverified) | Confirm the Spring version in the dependency tree before finalizing AC-5. |
| The web module has no tests today, so JUnit 5 and Surefire discovery are unverified. | Risk | Confirm with the baseline run. Stop if `pom.xml` changes are required. |
| Rejecting `\` changes behavior on Linux, where it is a legal filename character. | Risk | Accepted by this specification. |
| Links built by the templates do not URL-encode keys, so files with characters such as `#`, `?`, `%`, `+`, or `\` may produce broken view or delete links (the service itself is unaffected). | Risk (unverified at runtime) | Out of scope for this change. Recorded as a follow-up; see Unresolved questions. |
| A dev instance may be reachable from other machines. | Assumption (unverified) | Bind address and `dev` profile activation were not inspected. |
| `target/` is ignored by git. | Assumption (unverified) | Check with `git status` after the baseline run. |
| Worker `LocalFileProcessingService` trusts message keys without a containment check. | Risk (accepted) | Out of scope by owner decision. Track as a follow-up. |

## Plan-mode review checkpoint

### Evidence

- **Current behavior:**
  - `legacy-src/java-asset-manager/web/src/main/java/com/microsoft/migration/assets/service/LocalFileStorageService.java:uploadObject` — `cleanPath` plus a `contains("..")` check only, then `rootLocation.resolve(filename)` and `Files.copy`.
  - `LocalFileStorageService.getObject` and `deleteObject` — `rootLocation.resolve(key)` with only an existence check.
  - `legacy-src/java-asset-manager/web/src/main/java/com/microsoft/migration/assets/controller/S3Controller.java:viewObject` — catches `IOException` and returns 404.
  - `S3Controller.deleteObject` and `uploadObject` — flash messages `"Failed to delete file: " + e.getMessage()` and `"Failed to upload file: " + e.getMessage()`.
  - `legacy-src/java-asset-manager/web/src/main/java/com/microsoft/migration/assets/service/StorageService.java:getThumbnailKey` — appends `_thumbnail` to the filename.
  - `legacy-src/java-asset-manager/worker/src/main/java/com/microsoft/migration/assets/worker/service/LocalFileProcessingService.java:downloadOriginal` and `uploadThumbnail` — unguarded `rootLocation.resolve(key)` (out of scope).
- **URL encoding finding:**
  - Service: no encoding is required or wanted. Spring decodes the `{key}` path variable once before `S3Controller` passes it to the service, and the service uses it as a raw file name. Encoding or decoding there would double-process names such as `100%2Fine.png`. `/` and `\` cannot be valid keys, so they are rejected (FR-2) rather than encoded.
  - Templates: `legacy-src/java-asset-manager/web/src/main/resources/templates/list.html` lines 10, 20, 21 and `view.html` lines 21, 24, 33 build links as `@{'/storage/view/' + ${object.key}}` (and the `view-page` and `delete` equivalents). This concatenates the raw key into the URL. As I understand Thymeleaf link expressions, only `{name}` placeholders with parameters are encoded, not concatenated strings, so keys containing `#`, `?`, `%`, `+`, spaces, or `\` are likely to produce wrong links. Unverified at runtime.
  - Recommended encoding for a follow-up: use Thymeleaf path-variable syntax such as `@{/storage/view/{key}(key=${object.key})}`, which percent-encodes the segment (for example `%` becomes `%25`, `#` becomes `%23`, `?` becomes `%3F`, `\` becomes `%5C`; a space is encoded as `%20`). Note that Tomcat rejects `%5C` and `%2F` by default, so such names would still not be reachable via the browser, which is another reason for rejecting `/` and `\` in the service.
  - `StorageService.generateUrl` returns `"/storage/view/" + key` unencoded. Its value is stored in `S3StorageItem.url`; the templates do not use that property (`grep` found no template reference), so the practical impact is unconfirmed.
  - `upload.html:125` stores the raw `file.name` in `sessionStorage` and `list.html` compares it with `data-key` attributes. This is plain string comparison, not URL building.
- **Existing tests:** `legacy-src/java-asset-manager/worker/src/test/java/com/microsoft/migration/assets/worker/service/S3FileProcessingServiceTest.java`. No tests exist in the `web` module.
- **Baseline validation evidence:** not yet run. The expected full-reactor result is documented in `legacy-src/java-asset-manager/AGENTS.md`: `.\mvnw.cmd -B -ntp test` fails only on the known worker error (final `software.amazon.awssdk.core.ResponseInputStream` cannot be mocked) with the other three worker tests passing. Record the observed result for both that command and the primary command before editing.

### Unresolved questions

- Resolved by the owner: Windows-specific hardening and symlink handling are out of scope; `%2f` and `%5c` are not rejected by the service; template link encoding is a separate follow-up change, not part of this specification.
- [x] Which Spring version does Boot 2.7 resolve, and does `cleanPath(null)` return null? Owner: implementer, answered during the baseline check.
- [x] Is the `dev` profile reachable beyond localhost (bind address, activation)? Owner: implementer. Affects the problem statement only.
- [x] How should a key that names a directory in the root behave? Default: unchanged. Owner: @LSmartGOS.
- [x] Does `-pl web` run correctly without `-am`? Owner: implementer, answered by the baseline run.

The specification is not ready for approval until the remaining open questions are answered, because they could change the acceptance criteria or validation.

### Boundary and validation review

- [x] Evidence citations were checked against the repository.
- [x] Allowed paths include all anticipated implementation and test changes and
      nothing unrelated.
- [x] The primary validation command is safe, non-interactive, and runnable
      from the stated source root.
- [x] Acceptance criteria cover the required normal, failure, boundary, and
      compatibility behavior.
- [x] Exclusions prevent unrelated modernization or cleanup.

### Stop conditions

Stop and request human guidance if:

- repository evidence contradicts this specification (for example legitimate keys containing `..` or subdirectories turn out to be required);
- an unresolved question could materially change behavior or scope;
- satisfying a criterion requires a path outside the allowed paths, including any controller, worker, `pom.xml`, or `application.properties` change;
- the baseline shows any failure other than the exact known worker error, or the primary validation fails for a reason caused by the proposed change, or the command is missing, destructive, or irreproducible (do not weaken or delete tests to pass);
- a missing prerequisite prevents required evidence from being collected, such as the JDK, the Maven wrapper, or network access for dependency download;
- a secret, credential, real data, production resource, destructive action, dependency change, schema change, or public API change is required but not explicitly approved. Tests must only touch `@TempDir`. Never run delete against the real `../storage`, and never run git history rewrites, `target/` deletion, or Azure actions; or
- implementation or validation cannot satisfy the acceptance criteria without expanding the approved contract, or a failure's cause is not understood after reasoned fixes.

### Human approval

- [x] I reviewed the evidence, unresolved questions, allowed paths, acceptance
      criteria, validation command, exclusions, risks, and stop conditions.
- **Reviewer:** @LSmartGOS
- **Approval date:** 2026-11-23
- **Approved specification commit:** 5ba8653f208cafd952c50406d8b58f5e242baa69

Change **Status** to `Approved` only after the checkbox is selected and the
reviewer, date, and commit fields are complete. Planning approval authorizes
issue preparation, not implementation.
