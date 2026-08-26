# 04 · Security Best Practices

Two problems show up in almost every production service: proving a user is
who they claim to be (authentication) and proving a caller is allowed to do
what they're asking (authorization). This module builds both from JDK
primitives only — password hashing with PBKDF2 and signed tokens with
HMAC — so the mechanics are visible before you reach for a library like
`bcrypt` bindings or a JWT library that does the same thing pre-packaged.

## Never store plain-text passwords

Storing `"hunter2"` directly means a single database leak exposes every
user's real password. The fix is a **salted, slow hash**: PBKDF2 with a
per-user random salt and a high iteration count, built entirely from
`javax.crypto`:

```scala
import java.security.SecureRandom
import javax.crypto.SecretKeyFactory
import javax.crypto.spec.PBEKeySpec
import java.util.Base64

object Passwords:
  private val random = SecureRandom()

  def hash(password: String): String =
    val salt = new Array[Byte](16)
    random.nextBytes(salt)
    val spec = new PBEKeySpec(password.toCharArray, salt, 100000, 256)
    val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
    val hashBytes = factory.generateSecret(spec).getEncoded
    s"${Base64.getEncoder.encodeToString(salt)}:${Base64.getEncoder.encodeToString(hashBytes)}"

  def verify(password: String, stored: String): Boolean =
    val Array(saltB64, hashB64) = stored.split(":")
    val salt = Base64.getDecoder.decode(saltB64)
    val spec = new PBEKeySpec(password.toCharArray, salt, 100000, 256)
    val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
    val candidateHash = Base64.getEncoder.encodeToString(factory.generateSecret(spec).getEncoded)
    java.security.MessageDigest.isEqual(candidateHash.getBytes, hashB64.getBytes)

val hashed = Passwords.hash("hunter2")
println(Passwords.verify("hunter2", hashed))   // true
println(Passwords.verify("wrong", hashed))     // false
```

The salt is stored **alongside** the hash (it doesn't need to be secret,
just unique per password) — its job is making two users with the same
password produce different stored hashes, which defeats precomputed
rainbow-table attacks. The 100,000 iterations are deliberate: PBKDF2 is
designed to be slow, so brute-forcing a leaked hash costs an attacker real
computation per guess, unlike a single fast `SHA-256` of the raw password.

### The trap: comparing hashes with `==`

`verify` uses `MessageDigest.isEqual`, not `hashA == hashB`. A naive
`String` comparison short-circuits at the first mismatched character,
which means the comparison takes measurably longer the more characters
match — a timing side-channel an attacker can exploit to guess a hash
byte-by-byte. `MessageDigest.isEqual` always compares the full length in
constant time regardless of where the strings diverge, specifically to
close that channel. This same "constant-time compare" concern applies to
comparing any other secret (API keys, signatures) — never `==` on secrets
that came from outside the process.

## Signed tokens for stateless authentication

Once a password is verified, the server issues a **token** the client sends
on later requests instead of re-authenticating every time. An HMAC-signed
token (the mechanism a JWT is built on) proves the token wasn't tampered
with, without the server needing to store session state:

```scala
import javax.crypto.Mac
import javax.crypto.spec.SecretKeySpec
import java.nio.charset.StandardCharsets

object Tokens:
  private val secret = "super-secret-key"   // in production: from a secrets manager, never hardcoded

  def sign(payload: String): String =
    val mac = Mac.getInstance("HmacSHA256")
    mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA256"))
    val sig = Base64.getUrlEncoder.withoutPadding.encodeToString(
      mac.doFinal(payload.getBytes(StandardCharsets.UTF_8))
    )
    s"$payload.$sig"

  def verify(token: String): Option[String] =
    token.split("\\.", 2) match
      case Array(payload, sig) if sign(payload) == token => Some(payload)
      case _ => None

val token = Tokens.sign("user=ada;role=admin")
println(token)              // user=ada;role=admin.6poDkG3WB9DhDHRUKpM8L6s9SOqaM1xF-gqgZyPv56E
println(Tokens.verify(token))            // Some(user=ada;role=admin)
println(Tokens.verify(token + "tampered")) // None -- signature no longer matches
```

Anyone can *read* the payload (it's only Base64/plain text, not encrypted)
— the signature only proves it wasn't **modified** since the server signed
it, since recomputing a matching signature requires the secret key. This
is exactly the guarantee a JWT's signature provides; a real JWT adds a
standard JSON structure (header/claims), expiry (`exp`), and usually a
library instead of hand-rolled parsing — but the cryptographic core is the
same `HMAC(secret, payload)`.

### The trap: authentication proves identity, not permission

Verifying a token tells you *who* is calling — it says nothing about
*what they're allowed to do*. Checking `role=admin` requires reading the
verified payload and explicitly gating the operation:

```scala
def requireAdmin(payload: String): Boolean = payload.contains("role=admin")

Tokens.verify(token).foreach { payload =>
  if requireAdmin(payload) then println("access granted")
  else println("403 forbidden")
}
// access granted
```

Skipping this step — treating "the token is valid" as "the request is
allowed" — is how services end up letting any logged-in user perform
admin-only actions.

## Cheat sheet

| Need to... | Use |
|---|---|
| Store a password safely | PBKDF2 (or bcrypt/argon2 in production) with a per-user random salt |
| Compare secrets/hashes | `MessageDigest.isEqual`, never `==` |
| Issue a stateless auth token | HMAC-signed payload (the mechanism behind JWT) |
| Verify a token wasn't tampered with | recompute the signature and compare (constant-time) |
| Gate an action by role/permission | check the verified payload's claims explicitly — authn ≠ authz |

## Exercise

Add an expiry to `Tokens`: change `sign` to embed a Unix timestamp in the
payload (e.g. `s"user=ada;role=admin;exp=${System.currentTimeMillis() +
60000}"`), and change `verify` to return `None` if the signature is valid
but `exp` has already passed. Test that a freshly signed token verifies
successfully, and that a token you construct with an `exp` in the past
(even with a correct signature) is rejected. Then write `hasRole(payload:
String, role: String): Boolean` generically (don't hardcode `"admin"`) and
use it to gate two different pretend endpoints requiring different roles.
