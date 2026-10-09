# Required shared-domain patch for headless 1.2.19

Use this setup when a visitor journey crosses a Replo subdomain and Shopify on the same registered domain, including root, `www`, or sibling subdomains. This affects on-page test attribution across those pages as well as Split URL tests. Installing unpatched `@intelligems/headless@1.2.19` is not sufficient for these journeys.

The SDK reads visitor identity before its downloaded cookie-domain configuration is applied. A Replo-first visit can therefore create a host-only identity that differs from Shopify's shared identity. The tested patch sets the domain before initialization, migrates host cookies when no shared identity exists, preserves an existing shared identity, and synchronizes localStorage identity and first-visit state before context construction.

**Required:** install the exact patched version below and set `cookieDomain`, or use a vendor-fixed version after verifying equivalent behavior with the live audit. Do not assume a newer version contains the fix. This workaround was tested with 1.2.19's React ESM bundle; it is not vendor-approved, does not cover CommonJS, and must not be blindly applied to a different version. Same-host-only journeys do not demonstrate this shared-domain bug; document that scope if the patch is not applicable.

## Install and persist the patch

Run from the site's package directory. Save the script below as `scripts/patch-headless-cookie-domain.mjs` in that site first. Use a fresh temporary edit directory; if replacing an existing patch, inspect it and use pnpm's `--ignore-existing` option deliberately so other fixes are not lost.

```sh
pnpm add --save-exact @intelligems/headless@1.2.19
pnpm patch @intelligems/headless@1.2.19 --edit-dir /tmp/ig-headless-cookie-patch
node scripts/patch-headless-cookie-domain.mjs /tmp/ig-headless-cookie-patch
node --check /tmp/ig-headless-cookie-patch/dist/clientside-DZoD6gVq.mjs
pnpm patch-commit /tmp/ig-headless-cookie-patch
```

Commit the generated patch, pnpm's patch registration (package.json or pnpm-workspace.yaml, depending on pnpm version), and the lockfile with the site. Do not edit the lockfile manually. Verify a clean `pnpm install --frozen-lockfile` applies the patch, then rebuild and publish. Editing an installed `node_modules` bundle alone will not survive the publish/install process.

The patched ESM bundle's SHA-256 must be:

```text
6f642cf48ebed02353938e689bccbba9a372554cf3a166c3bf43f390eb8b9c7b
```

Check with `shasum -a 256` (macOS) or `sha256sum` (Linux). The script refuses a different package version, missing replacement targets or an already-patched bundle. Stop and inspect a mismatch rather than weakening those checks.

## Configure the provider

`cookieDomain` is a **patch-added prop**, not a stock 1.2.19 prop. Set it on every Replo headless provider, before initialization, to the merchant's actual shared cookie domain, with no protocol or path. Do not derive a registrable domain by dropping the first hostname label (that breaks domains such as `brand.co.uk`). Match Shopify's Intelligems domain configuration and verify the delivered CDN config.

```tsx
<IntelligemsNextClientsideAppDirectoryProvider
  organizationId={organizationId}
  storefrontApiToken={storefrontAccessToken}
  activeCurrencyCode={market.currencyCode}
  cookieDomain="brand.com"
>
  {children}
</IntelligemsNextClientsideAppDirectoryProvider>
```

Here `storefrontAccessToken` comes from `ShopifyIntegrationDetailsLoader`; retain the market, consent and tracker setup from the main guide. `brand.com` is an example, not a value to copy. The page hostname must equal that domain or be its subdomain. Use the merchant domain for published QA; localhost and `replosites.com` cannot set a merchant-domain cookie.

## Verification and limits

- **Code audit:** inspect the exact dependency version, committed patch registration, patched bundle hash, provider prop and all used entrypoints. Do not implement ad-hoc cookie manipulation in page components in place of the patch.
- **Live experiment audit:** use fresh Replo-first and Shopify-first visits through root/www/sibling destinations as applicable; check the same visitor and assignment at both ends. Also exercise an existing shared identity with stale host cookies/localStorage and a legacy host-only identity with no shared cookie.
- Published QA on October 6, 2026 verified this patch with regular-cart and Buy Now orders and Intelligems reports across Replo, Shopify root, www and a sibling domain. This is evidence for that exact patch, not certification of arbitrary targeting, subscription pricing, or legacy new/returning-visitor histories. Each merchant setup still needs the [live experiment audit](live-experiment-audit.md).
- This patch fixes identity initialization. It does not implement redirects, server assignment, or route reevaluation. Next.js client navigation still needs the separate redirect checks in the [code audit](code-audit.md).

## Patch script

Copy this exact script into the site file named above. Keep it with the generated pnpm patch so the workaround is reproducible.

```js
import { readFile, writeFile } from "node:fs/promises";
import { join } from "node:path";

const packageRoot = process.argv[2];
if (!packageRoot) throw new Error("Pass the installed @intelligems/headless directory");

const packageJson = JSON.parse(await readFile(join(packageRoot, "package.json"), "utf8"));
if (packageJson.version !== "1.2.19") {
  throw new Error(`Expected @intelligems/headless 1.2.19, got ${packageJson.version}`);
}

const bundlePath = join(packageRoot, "dist/clientside-DZoD6gVq.mjs");
const source = await readFile(bundlePath, "utf8");
const initializer = "let t=ReactClient.init(u),n=new ReactContext(t,";
const deletion =
  "deleteIgCookie(e){Cookies.remove(this.cookieCtxKey(e),{domain:window.location.hostname?`.`+window.location.hostname:void 0})}";
const declarationPaths = [
  "dist/index-BTBlM9P7.d.ts",
  "dist/index-B6Yc7Ybb.d.ts",
  "dist/src/types/core/provider.d.ts",
];
const declarations = await Promise.all(
  declarationPaths.map(async (path) => ({
    path: join(packageRoot, path),
    source: await readFile(join(packageRoot, path), "utf8"),
  })),
);

if (source.startsWith("function prepareSharedIgCookies(")) {
  throw new Error("This Intelligems bundle is already patched");
}
if (!source.includes(initializer) || !source.includes(deletion)) {
  throw new Error("Intelligems bundle changed; inspect it before applying this patch");
}
if (declarations.some(({ source }) => !source.includes("cacheIntervalMinutes?: number;"))) {
  throw new Error("Intelligems provider declaration changed; inspect it before applying this patch");
}

const prepare = `function prepareSharedIgCookies(root) {
  const host = window.location.hostname;
  if (!root || host === root || !host.endsWith("." + root)) return;
  const keys = ["ig-id", "ig-fv", "ig-pv", "ig-vars", "ig-ignored", "ig-preview", "ig-integration", "ig-preview-traffic"];
  const previous = Object.fromEntries(keys.map(key => [key, Cookies.get(key)]));
  for (const key of keys) {
    Cookies.remove(key, { domain: host, path: "/" });
    Cookies.remove(key, { domain: "." + host, path: "/" });
    Cookies.remove(key, { path: "/" });
  }
  if (!Cookies.get("ig-id") && previous["ig-id"]) {
    for (const key of keys) {
      if (previous[key]) Cookies.set(key, previous[key], { domain: "." + root, path: "/", expires: 365 });
    }
  }
  if (Cookies.get("ig-id")) {
    try {
      localStorage.setItem("ig-id", Cookies.get("ig-id"));
      const firstVisit = Cookies.get("ig-fv");
      if (firstVisit) localStorage.setItem("ig-fv", firstVisit);
      else localStorage.removeItem("ig-fv");
    } catch {}
  }
}
`;

const patched =
  prepare +
  source
    .replace(
      initializer,
      "let root=e.cookieDomain||e.config?.options?.domain;prepareSharedIgCookies(root);let t=ReactClient.init(u);if(root)t.domain=root;let n=new ReactContext(t,",
    )
    .replace(
      deletion,
      "deleteIgCookie(e){Cookies.remove(this.cookieCtxKey(e),{domain:this.getCookieDomain()})}",
    );

await writeFile(bundlePath, patched);
for (const declaration of declarations) {
  await writeFile(
    declaration.path,
    declaration.source.replace("cacheIntervalMinutes?: number;", "cacheIntervalMinutes?: number;\n    cookieDomain?: string;"),
  );
}
console.log("Patched @intelligems/headless 1.2.19 cookie domain in installed package");
```
