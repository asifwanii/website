---
id: plugin-auth
title: "Authentication Plugin"
---

## What's an Authentication Plugin?

Is a sort plugin that allows to handle who access or publish to a specific package. By default the `htpasswd` is built-in, but can easily be replaced by your own.

 ## Getting Started

The authentication plugins are defined in the `auth:` section, as follows:

```yaml
auth:
  htpasswd:
    file: ./htpasswd
```

also multiple plugins can be chained:

```yaml
auth:
  htpasswd:
    file: ./htpasswd
  anotherAuth:
    foo: bar
    bar: foo
  lastPlugin:
    foo: bar
    bar: foo
```

> If one of the plugin in the chain is able to resolve the request, the next ones will be ignored.

## How do the authentication plugin works?

U suštini treba da vratimo objekat korišćenjem metode zvane `authenticate` koja prima 3 argumenta (`user, password, callback`).

On each request, `authenticate` will be triggered and the plugin should return the credentials, if the `authenticate` fails, it will fallback to the `$anonymous` role by default.

### API

```typescript
  interface IPluginAuth<T> extends IPlugin<T> {
    authenticate(user: string, password: string, cb: AuthCallback): void;
    adduser?(user: string, password: string, cb: AuthCallback): void;
    changePassword?(user: string, password: string, newPassword: string, cb: AuthCallback): void;
    allow_publish?(user: RemoteUser, pkg: AllowAccess & PackageAccess, cb: AuthAccessCallback): void;
    allow_access?(user: RemoteUser, pkg: AllowAccess & PackageAccess, cb: AuthAccessCallback): void;
    allow_unpublish?(user: RemoteUser, pkg: AllowAccess & PackageAccess, cb: AuthAccessCallback): void;
    apiJWTmiddleware?(helpers: any): Function;
  }
```
> Only `adduser`, `allow_access`, `apiJWTmiddleware`, `allow_publish`  and `allow_unpublish` are optional, verdaccio provide a fallback in all those cases.

#### `apiJWTmiddleware` method

Od verzije `v4.0.0`

`apiJWTmiddleware` je uveden od [PR#1227](https://github.com/verdaccio/verdaccio/pull/1227) kako bi se omogućila potpuna kontrola nad upravljanjem tokenima (token handler). Ako biste pregazili taj metod, onemogućili biste `login/adduser` podršku. Preporučujemo Vam da ne koristite navedeni metod, osim ako to nije apsolutno neophodno. Detaljni primer možete pronaći [ovde](https://github.com/verdaccio/verdaccio/pull/1227#issuecomment-463235068).


## What should I return in each of the methods?

Verdaccio relies on `callback` functions at time of this writing. Each method should call the method and what you return is important, let's review how to do it.


### `authentication` callback

Jednom kada se autentifikacija izvrši, na raspolaganju su 2 opcije koje daju odgovor `verdaccio-u`.

##### If the authentication fails

If the auth was unsuccessful, return `false` as the second argument.

```typescript
callback(null, false)
```

##### If the authentication success

Auth je uspešno objavljena.


`groups` čini niz stringova u koji spada korisnik.

```
 callback(null, groups);
```

##### If the authentication produce an error

The authentication service might fails, and you might want to reflect that in the user response, eg: service is unavailable.

```
 import { getInternalError } from '@verdaccio/commons-api';

 callback(getInternalError('something bad message), null);
```

> A failure on login is not the same as service error, if you want to notify user the credentails are wrong, just return `false` instead string of groups. The behaviour mostly depends of you.


### `adduser` callback

##### If adduser success

If the service is able to create an user, return `true` as the second argument.

```typescript
callback(null, true)
```

##### If adduser fails

Any other action different than success must return an error.

```typescript
import { getConflict } from '@verdaccio/commons-api';

const err = getConflict('maximum amount of users reached');

callback(err);
```

### `changePassword` callback

##### If the request is successful

If the service is able to create an user, return `true` as the second argument.

```typescript
const user = serviceUpdatePassword(user, password, newPassword);

callback(null, user)
```

##### If the request fails

Any other action different than success must return an error.

```typescript
import { getNotFound } from '@verdaccio/commons-api';

 const err = getNotFound('user not found');

callback(err);
```

### `allow_access`, `allow_publish`, or `allow_unpublish` callback

These methods aims to allow or deny trigger some actions.

##### If the request success

If the service is able to create an user, return a `true` as the second argument.

```typescript

allow_access(user: RemoteUser, pkg: PackageAccess, cb: Callback): void {
  const isAllowed: boolean = checkAction(user, pkg);

  callback(null, isAllowed)
}
```

##### If the request fails

Any other action different than success must return an error.

```typescript
import { getNotFound } from '@verdaccio/commons-api';

 const err = getForbidden('not allowed to access package');

callback(err);
```

## Generate an authentication plugin

For detailed info check our [plugin generator page](plugin-generator). Run the `yo` command in your terminal and follow the steps.

```
➜ yo verdaccio-plugin

Just found a `.yo-rc.json` in a parent directory.
Setting the project root at: /Users/user/verdaccio_yo_generator

     _-----_     ╭──────────────────────────╮
    |       |    │        Welcome to        │
    |--(o)--|    │ generator-verdaccio-plug │
   `---------´   │   in plugin generator!   │
    ( _´U`_ )    ╰──────────────────────────╯
    /___A___\   /
     |  ~  |
   __'.___.'__
 ´   `  |° ´ Y `

? What is the name of your plugin? service-name
? Select Language typescript
? What kind of plugin you want to create? auth
? Please, describe your plugin awesome auth plugin
? GitHub username or organization myusername
? Author's Name Juan Picado
? Author's Email jotadeveloper@gmail.com
? Key your keywords (comma to split) verdaccio,plugin,auth,awesome,verdaccio-plugin
   create verdaccio-plugin-authservice-name/package.json
   create verdaccio-plugin-authservice-name/.gitignore
   create verdaccio-plugin-authservice-name/.npmignore
   create verdaccio-plugin-authservice-name/jest.config.js
   create verdaccio-plugin-authservice-name/.babelrc
   create verdaccio-plugin-authservice-name/.travis.yml
   create verdaccio-plugin-authservice-name/README.md
   create verdaccio-plugin-authservice-name/.eslintrc
   create verdaccio-plugin-authservice-name/.eslintignore
   create verdaccio-plugin-authservice-name/src/index.ts
   create verdaccio-plugin-authservice-name/index.ts
   create verdaccio-plugin-authservice-name/tsconfig.json
   create verdaccio-plugin-authservice-name/types/index.ts
   create verdaccio-plugin-authservice-name/.editorconfig

I'm all done. Running npm install for you to install the required dependencies. If this fails, try running the command yourself.


⸨ ░░░░░░░░░░░░░░░░░⸩ ⠋ fetchMetadata: sill pacote range manifest for @babel/plugin-syntax-jsx@^7.7.4 fetc
```

After the install finish, access to your project scalfold.

```
➜ cd verdaccio-plugin-service-name
➜ cat package.json

  {
  "name": "verdaccio-plugin-service-name",
  "version": "0.0.1",
  "description": "awesome auth plugin",
  ...
```

## Full implementation ES5 example

```javascript
function Auth(config, stuff) {
  var self = Object.create(Auth.prototype);
  self._users = {};

  // config for this module
  self._config = config;

  // verdaccio logger
  self._logger = stuff.logger;

  // pass verdaccio logger to ldapauth
  self._config.client_options.log = stuff.logger;

  return self;
}

Auth.prototype.authenticate = function (user, password, callback) {
  var LdapClient = new LdapAuth(self._config.client_options);
  ....
  LdapClient.authenticate(user, password, function (err, ldapUser) {
    ...
    var groups;
     ...
    callback(null, groups);
  });
};

module.exports = Auth;
```

I na kraju, konfiguracija izgleda ovako:

```yaml
auth:
  htpasswd:
    file: ./htpasswd
```

Pri čemu je `htpasswd` sufiks za ime plugina. primer: `verdaccio-htpasswd` i ostatak body sekcije će dati parametre za konfigurisanje plugina.

### List Community Authentication Plugins

* [verdaccio-bitbucket](https://github.com/idangozlan/verdaccio-bitbucket): Bitbucket authentication plugin za verdaccio.
* [verdaccio-bitbucket-server](https://github.com/oeph/verdaccio-bitbucket-server): Bitbucket Server authentication plugin za verdaccio.
* [verdaccio-ldap](https://www.npmjs.com/package/verdaccio-ldap): LDAP auth plugin za verdaccio.
* [verdaccio-active-directory](https://github.com/nowhammies/verdaccio-activedirectory): Active Directory authentication plugin za verdaccio
* [verdaccio-gitlab](https://github.com/bufferoverflow/verdaccio-gitlab): koristi GitLab Personal Access Token za autentifikaciju
* [verdaccio-gitlab-ci](https://github.com/lab360-ch/verdaccio-gitlab-ci): Omogućava GitLab CI da authenticate protiv verdaccio.
* [verdaccio-htpasswd](https://github.com/verdaccio/verdaccio-htpasswd): File plugin za Auth based on htpasswd (ugrađen), za verdaccio
* [verdaccio-github-oauth](https://github.com/aroundus-inc/verdaccio-github-oauth): Github oauth authentication plugin za verdaccio.
* [verdaccio-github-oauth-ui](https://github.com/n4bb12/verdaccio-github-oauth-ui): GitHub OAuth plugin za the verdaccio login dugme.
* [verdaccio-groupnames](https://github.com/deinstapel/verdaccio-groupnames): Plugin to handle dynamic group associations utilizing `$group` syntax. Works best with the ldap plugin.
* [verdaccio-sqlite](https://github.com/bchanudet/verdaccio-sqlite): SQLite Authentication plugin for Verdaccio

**Have you developed a new plugin? Add it here !**
