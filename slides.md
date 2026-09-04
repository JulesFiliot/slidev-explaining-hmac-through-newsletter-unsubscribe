---
theme: '@logos-co/slidev-theme-logos'
layout: cover
title: Securing our newsletter unsubscribe links with HMAC
info: |
  ## Securing our newsletter unsubscribe links with HMAC
  Why a public endpoint needs a tamper-proof link, and how HMAC gives us one.
date: September 2026
description: Technical Services / Web
event: Jules Filiot
drawings:
  persist: false
transition: slide-left
duration: 10min
---

---
layout: default
---

# The unsubscribe link

```
https://admin-acid.logos.co/api/newsletters/unsubscribe
  ?email=alice@example.com&typeName=logos&newsletterId=6913441fee2f120001cec90d
```

<div class="mt-6"></div>

```bash
for email in $(cat leaked-list.txt); do
  curl ".../unsubscribe?email=$email&typeName=logos&newsletterId=6913441f..."
done
```

<p class="mt-8">Public endpoint. <span class="highlight">Writes to someone else's subscription.</span> Trusts only the URL.</p>

---
layout: default
class: stacked
---

# There is nobody to authenticate

<div class="section">

### No session

- No account, no cookie, no password
- Nothing to look them up against

</div>

<div class="section">

### And it stays public

- Gmail's one-click button
- Opened five years later

</div>

---
layout: center
---

<div class="text-center">

## We cannot verify who is clicking.

<div class="mt-8"></div>

## We can verify the link is <span class="highlight">ours</span>, and <span class="highlight">unchanged</span>.

</div>

---
layout: default
---

# What we need in the URL

<div class="mt-6"></div>

- Proves we issued it
- Covers <span class="highlight">every</span> parameter the endpoint acts on
- No server-side state
- Never expires

<div class="mt-10"></div>

<p>That is a <b>Message Authentication Code</b>.</p>

---
layout: default
---

# HMAC

```
MAC(key, message) → tag
```

<div class="mt-6"></div>

<p>Without the key, you cannot produce a valid <code>(message, tag)</code> pair. <span class="highlight">That is the whole guarantee.</span></p>

<div class="mt-8"></div>

- **Symmetric** -- one secret, both ends are us
- **Standard** -- HMAC-SHA256, RFC 2104
- **Small** -- 43 characters, nothing stored

---
layout: default
---

# Minting the link

<div class="mint-flow">

<div class="mint-step">
  <div class="mint-head"><span class="mint-num">01</span>The three fields the endpoint acts on</div>
  <div class="mint-value">alice@example.com<span class="mint-sep">\n</span>logos<span class="mint-sep">\n</span>6913441fee2f120001cec90d</div>
</div>

<div class="mint-arrow">&darr;</div>
<div class="mint-step">
  <div class="mint-head"><span class="mint-num">02</span>HMAC-SHA256, keyed with our secret</div>
  <div class="mint-value">3fd68ed8d5fa4b8d284fcdf6c40995d887982cb8632ef3d1c7d8c049b59e7335</div>
</div>

<div class="mint-arrow">&darr;</div>
<div class="mint-step">
  <div class="mint-head"><span class="mint-num">03</span>base64url &mdash; 43 characters, URL-safe</div>
  <div class="mint-value">P9aO2NX6S40oT832xAmV2IeYLLhjLvPRx9jASbWeczU</div>
</div>

<div class="mint-arrow">&darr;</div>
<div class="mint-step mint-out">
  <div class="mint-head"><span class="mint-num">04</span>Appended to the link</div>
  <div class="mint-value">...&newsletterId=6913441fee2f120001cec90d&amp;token=<span class="mint-token">P9aO2NX6S40oT832xAmV2IeYLLhjLvPRx9jASbWeczU</span></div>
</div>

</div>

<style>
.mint-flow {
  margin-top: 0.6rem;
}
.mint-head {
  font-family: var(--font-body);
  font-size: 0.8rem;
  font-weight: 700;
  color: var(--color-black);
  margin-bottom: 0.25rem;
}
.mint-num {
  font-weight: 400;
  color: var(--color-muted);
  margin-right: 0.6rem;
}
.mint-value {
  border: var(--border-dark);
  border-radius: 6px;
  background: var(--color-white);
  padding: 0.5rem 0.9rem;
  font-family: var(--font-code);
  font-size: 0.72rem;
  line-height: 1.5;
  color: var(--color-forest);
  overflow-wrap: anywhere;
}
.mint-sep {
  color: var(--color-coral);
  margin: 0 0.5rem;
}
.mint-arrow {
  color: var(--color-muted);
  font-size: 0.9rem;
  line-height: 1;
  padding: 0.3rem 0 0.3rem 0.55rem;
}
.mint-out .mint-value {
  border-color: var(--color-forest);
  background: var(--color-parchment);
}
.mint-token {
  color: var(--color-coral);
  font-weight: 700;
}
</style>

---
layout: two-cols
---

# Verifying

::left::

```ts
export function verifyUnsubscribeToken(params, token) {
  const secret = process.env.UNSUBSCRIBE_TOKEN_SECRET;
  if (!secret || !token) return false;

  return safeEq(token, sign(params, secret));
}
```

::right::

<div class="verify-flow">
  <div class="verify-row"><b>Request</b><br>email + typeName + newsletterId + token</div>
  <div class="verify-arrow">↓ recompute from the parameters</div>
  <div class="verify-ok"><b>match</b> - unsubscribe</div>
  <div class="verify-bad"><b>no match</b> - 400, invalid link</div>
</div>

<style>
.verify-flow { font-size: 0.85rem; line-height: 1.5; margin-top: 0.5rem; }
.verify-row, .verify-ok, .verify-bad {
  border: var(--border-dark);
  border-radius: 6px;
  padding: 0.6rem 0.9rem;
  background: var(--color-white);
}
.verify-arrow { padding: 0.6rem 0; color: var(--color-muted); }
.verify-ok { margin-bottom: 0.6rem; color: var(--color-sage); }
.verify-bad { color: var(--color-coral); }
</style>

---
layout: default
---

# Change one character

| Newsletter id | Digest |
|---|---|
| `...cec90d` | `3fd68ed8d5fa4b8d284fcdf6c40995d887982cb8632ef3d1c7d8c049b59e7335` |
| `...cec90e` | `4231260c9e74d3d38b723ebefd805650eb738694a5221e800169716cad78f8f9` |

<div class="mt-6"></div>

```
valid     P9aO2NX6S40oT832xAmV2IeYLLhjLvPRx9jASbWeczU
tampered  QjEmDJ5009OLcj6-_YBWUOtzhpSlIh6AAWlxbK14-Pk
```

<p class="mt-8 text-center"><span class="highlight">127 of 256 bits flipped.</span></p>

---
layout: end
---

## Takeaways

<div class="text-left mt-2">

- No session? The URL is the credential
- HMAC proves a link is ours and unchanged. That is enough
- Sign exactly the fields you act on
- MAC failures are the wrong message, not a broken primitive

</div>

<p class="mt-8 belittle">Questions?</p>
