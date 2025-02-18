---
id: best
title: "Migliori Pratiche"
---

La guida seguente è un elenco delle migliore pratiche raccolte e che raccomandiamo solitamente a tutti gli utenti. Non considerare questa guida come obbligatoria, puoi selezionare qualcuna di esse a seconda delle tue esigenze.

**Feel free to suggest your best practices to the Verdaccio community**.

## Registro Privato

È possibile aggiungere utenti e gestire quali utenti possono accedere a quali pacchetti.

È raccomandabile definire un prefisso per i tuoi pacchetti privati, per esempio `local-*` o `@my-company/*` con scope, così che tutti i tuoi elementi privati appariranno così: `local-foo`. In questo modo si possono separare chiaramente i pacchetti pubblici da quelli privati.

```yaml
 packages:
   '@my-company/*':
     access: $all
     publish: $authenticated
    'local-*':
     access: $all
     publish: $authenticated
   '@*/*':
     access: $all
     publish: $authenticated
   '**':
     access: $all
     publish: $authenticated
```

Always remember, **the order of packages access is important**, packages are matched always top to bottom.

### Utilizzo dei pacchetti pubblici da npmjs.org

If a package doesn't exist in the storage, the server will try to fetch it from npmjs.org. If npmjs.org is down, it serves packages from the cache pretending that no other packages exist. **Verdaccio will download only what's needed (requested by clients)**, and this information will be cached, so if the client requests the same thing a second time it can be served without asking npmjs.org for it.

**Example:**

If you successfully request `express@4.0.1` from the server once, you'll be able to do it again (with all of it's dependencies) any time, even if npmjs.org is down. Though note that `express@4.0.0` will not be downloaded until it's actually needed by somebody. And if npmjs.org is offline, the server will say that only `express@4.0.1` (what's in the cache) is published, but nothing else.

### Annullare pacchetti pubblici

If you want to use a modified version of some public package `foo`, you can just publish it to your local server, so when your type `npm install foo`, **it'll consider installing your version**.

There's two options here:

1. Desideri creare un **fork** separato e interrompere la sincronizzazione con la versione pubblica.
    
    If you want to do that, you should modify your configuration file so Verdaccio won't make requests regarding this package to npmjs anymore. Aggiungi una voce separata per questo pacchetto a `config.yaml`, rimuovi `npmjs` dalla lista `proxy` e riavvia il server.
    
    ```yaml
    packages:
     "@my-company/*":
       access: $all
       publish: $authenticated
       # comment it out or leave it empty
       # proxy:
    ```
    
    When you publish your package locally, **you should probably start with a version string higher than the existing package** so it won't conflict with that package in the cache.

2. You want to temporarily use your version, but return to the public one as soon as it's updated.
    
    Per evitare conflitti delle versioni, **dovresti usare un suffisso personalizzato rilasciato prima della successiva versione della patch**. Per esempio, se un pacchetto pubblico ha la versione 0.1.2, puoi fare l'upload di `0.1.3-my-temp-fix`.
    
    ```bash
    npm version 0.1.3-my-temp-fix
    npm publish --tag fix --registry http://localhost:4873
    ```
    
    In questo modo il tuo pacchetto verrà utilizzato fino a che il suo manutentore originale aggiorna il suo pacchetto pubblico alla `0.1.3`.

## Sicurezza

Security starts in your environment. For such things we recommend reading:

- **[10 npm Security Best Practices](https://snyk.io/blog/ten-npm-security-best-practices/)** and following the steps outlined there.
- **[Avoiding npm substitution attacks](https://github.blog/2021-02-12-avoiding-npm-substitution-attacks/)**

> Feel free to attach here new useful articles to improve the security.

### Strong package access with `$authenticated`

By default all packages you publish in Verdaccio are accessible for all users. We recommend protecting your registry from external non-authorized users by updating the `access` property of your packages to `$authenticated`.

```yaml
packages:
  "@my-company/*":
    access: $authenticated
    publish: $authenticated
  "@*/*":
    access: $authenticated
    publish: $authenticated
  "**":
    access: $authenticated
    publish: $authenticated
```

That way, **nobody can access your registry unless they are authorized, and private packages won't be displayed in the web interface**.

### Remove `proxy` to increase security at private packages

After a clean installation, by default all packages will be resolved to the default uplink (the public registry `npmjs`).

```yaml
packages:
  "@*/*":
    access: $authenticated
    publish: $authenticated
    proxy: npmjs
  "**":
    access: $authenticated
    publish: $authenticated
    proxy: npmjs
```

This means, if a private packaged eg: `@my-company/auth` is published locally, the registry will look up at the public registry. If your intention is fully protection, remove the `proxy` property from your configuration, for instance:

```yaml
packages:
  "@my-company/*":
    access: $authenticated
    publish: $authenticated
    unpublish: $authenticated
  "@*/*":
    access: $authenticated
    publish: $authenticated
    proxy: npmjs
  "**":
    access: $authenticated
    publish: $authenticated
    proxy: npmjs
```

This configuration will **avoid downloading needlessly to external registries**, merging external metadata and download external tarballs.

## Server

### Secured Connections

Using **HTTPS** is a common recommendation. For this reason we recommend reading the [SSL](ssl.md) section to make Verdaccio secure, or alternatively using an HTTPS [reverse proxy](reverse-proxy.md) on top of Verdaccio.

### Expiring Tokens

Since `verdaccio@3.x` the tokens have no expiration date. For such reason we introduced in the next `verdaccio@4.x` the JWT feature [PR#896](https://github.com/verdaccio/verdaccio/pull/896)

```yaml
security:
  api:
    jwt:
      sign:
        expiresIn: 15d
        notBefore: 0
  web:
    sign:
      expiresIn: 7d
```

**Using this configuration will override the current system and you will be able to control how long the token will live**.

Using JWT also improves the performance with authentication plugins. The old system will perform an unpackage and validate the credentials on every request, while JWT will rely on the token signature instead, avoiding the overhead for the plugin.

As a side note, be aware at **npmjs** and the **legacy** verdaccio token never expires** unless you invalidate manually.