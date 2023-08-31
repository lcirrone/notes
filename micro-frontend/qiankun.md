[← back](..\README.md)

# Qiankun

### Fonti utilizzate
- https://medium.com/@fibonalabsdigital/how-to-implement-micro-frontends-using-qiankun-9f308eddc5f4
- https://qiankun.umijs.org/guide/tutorial

## Creazione progetto "body" (React)
- `npx create-react-app body`
- aggiungere al package.json la dipendenza dev "@babel/plugin-proposal-private-property-in-object" per evitare il warning
  ``` json
  "devDependencies": {
    "@babel/plugin-proposal-private-property-in-object": "^7.21.0"
  },
  ```
- modificare lo script "start" all'interno del package.json perché l'app usi una porta diversa da quella di default (la 3000, già usata dal progetto container)
  ``` json
  "scripts": {
    "start": "npx cross-env PORT=3001 react-scripts start",
    "...": "..."
  }
  ```
- ogni micro app deve esportare i tre hook/metodi dei cicli di vita (bootstrap, mount e unmount) dal proprio file entry principale, nel caso di React src/index.js:
  ``` jsx
  import "./public-path";
  import React from "react";
  import ReactDOM from "react-dom/client";
  import App from "./App";

  /**
   * The bootstrap will only be called once when the child application is initialized.
   * The next time the child application re-enters, the mount hook will be called directly, and bootstrap will not be triggered repeatedly.
   * Usually we can do some initialization of global variables here,
   * such as application-level caches that will not be destroyed during the unmount phase.
   */
  export async function bootstrap() {
    console.log("react app bootstraped");
  }
  
  /**
   * The mount method is called every time the application enters,
   * usually we trigger the application's rendering method here.
   */
  export async function mount(props) {
    ReactDOM.createRoot(
      props.container
        ? props.container.querySelector("#root")
        : document.getElementById("root")
    ).render(<App />);
  }
  
  /**
   * Methods that are called each time the application is switched/unloaded,
   * usually in this case we uninstall the application instance of the subapplication.
   */
  export async function unmount(props) {
    ReactDOM.unmountComponentAtNode(
      props.container
        ? props.container.querySelector("#root")
        : document.getElementById("root")
    );
  }
  ```
- creare un file public-path.js all'interno della cartella "src" che verrà usato per modificare il path pubblico a runtime:
  ``` js
  if (window.__POWERED_BY_QIANKUN__) {
  // eslint-disable-next-line
    __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__;
  }
  ```
  e importarlo all'interno di index.js → `import "./public-path";`
  
  **N.B.** la riga `// eslint-disable-next-line` è necessaria per evitare l'errore `__webpack_public_path__ is not defined`

## Configurazione Webpack
Dato che l'applicazione container e le micro app girano su porte diverse, bisogna sovrascrivere la configurazione del dev server perché i browser non permettono l'accesso da una porta all'altra. Per farlo si possono utilizzare librerie come "react-app-rewired" o "@rescripts/cli".
- `npm i react-app-rewired -S`
- nella root creare un file config-overrides.js con la seguente configurazione:
  ``` js
  const { name } = require("./package");

  module.exports = {
    webpack: (config) => {
      config.output.library = `${name}-[name]`;
      config.output.libraryTarget = "umd";
      // config.output.jsonpFunction = `webpackJsonp_${name}`;, → deprecated in Webpack 5
      config.output.chunkLoadingGlobal = `jsonp_${name}`;
      config.output.globalObject = "window";
  
      return config;
    },
  
    devServer: (configFunction) => {
      return function (proxy, allowedHost) {
        const config = configFunction(proxy, allowedHost);
        config.open = false;
        config.hot = false;
        config.headers = {
          "Access-Control-Allow-Origin": "*",
        };
        return config;
      };
    },
  };
  ```
- modificare gli script di start, build e test del package.json sostituendo a "react-scripts" "react-app-rewired":
  ``` json
  "scripts": {
    "start": "react-app-rewired start",
    "build": "react-app-rewired build",
    "test": "react-app-rewired test",
    "eject": "react-scripts eject"
  }
  ```
- creare un file .env per ogni micro app e inserire il numero di porta prescelto per quel server:
  ``` .env
  SKIP_PREFLIGHT_CHECK=true
  PORT=3001
  ```

## Creazione progetto "container" (React)
- `npx create-react-app container`
- aggiungere al package.json la dipendenza dev "@babel/plugin-proposal-private-property-in-object" per evitare il warning
  ``` json
  "devDependencies": {
    "@babel/plugin-proposal-private-property-in-object": "^7.21.0"
  },
  ```
- installare la dipendenza Qiankun, necessaria solo nel progetto container → `npm i qiankun -S`
- modificare l'index.js nel seguente modo:
  ``` js
  import { registerMicroApps, start, setDefaultMountApp } from "qiankun";

  registerMicroApps([
    {
      name: "body", // app name registered should match name in package.json of micro app 1
      entry: "//localhost:3001", // where our micro app 1 exists
      container: "#root", // div which exists in main app with id root, so inside this div our micro app will render
      activeRule: "/body", // our micro app will be visible in main app under this path
      props: { Routerbase: "/body" }, // used by qiankun for routing purpose
    },
  ]);
  setDefaultMountApp("/body"); // optional
  // by default, if you want to display a micro app you can use this
  /// in the above case by default, if we enter it will route localhost:3000 to localhost:3000/body
  
  // start qiankun
  start();
  ```
- rimuovere i file inutilizzati: App.js, App.css, App.test.js e index.css

## Creazione progetto "footer" (Vue)
- installare Vue CLI (deprecato!) se non ancora presente nel sistema → `npm install -g @vue/cli`
- creare una micro app Vue con il comando `vue create footer` scegliendo come configurazione "Default ([Vue 3] babel, eslint)"
- creare il file public-path.js all'interno della cartella "src":
  ``` js
  if (window.__POWERED_BY_QIANKUN__) {
    // eslint-disable-next-line
    __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__;
  }
  ```
  
  **N.B.** come nel caso di React, la riga `// eslint-disable-next-line` è necessaria per evitare l'errore `__webpack_public_path__ is not defined`
- installare "vue-router" → `npm i vue-router -S`
- importare il file "public-path.js" ed esportare i tre metodi dei cicli di vita (bootstrap, mount e unmount) dal proprio file entry principale, nel caso di Vue src/main.js:
  ``` js
  import "./public-path";
  import { createApp } from "vue";
  import { createRouter, createWebHistory } from "vue-router";
  import App from "./App.vue";
  import routes from "./router";
  // import store from "./store";

  let router = null;
  let instance = null;
  let history = null;

  function render(props = {}) {
    const { container } = props;
    history = createWebHistory(window.__POWERED_BY_QIANKUN__ ? "/footer" : "/");
    router = createRouter({
      history,
      routes,
    });

    instance = createApp(App);
    instance.use(router);
    // instance.use(store);
    instance.mount(container ? container.querySelector("#app") : "#app");
  }

  // when run independently
  if (!window.__POWERED_BY_QIANKUN__) {
    render();
  }

  export async function bootstrap() {
    console.log("%c%s", "color: green;", "vue3.0 app bootstraped");
  }
  export async function mount(props) {
    render(props);
  }
  export async function unmount() {
    instance.unmount();
    instance._container.innerHTML = "";
    instance = null;
    router = null;
    history.destroy();
  }
  ```
  **N.B.** il codice fornito nella guida ufficiale non è aggiornato, basarsi sull'esempio in Vue della [repository](https://github.com/umijs/qiankun/blob/master/examples/vue3/src/main.js) 
- file router.js:
  ``` js
  const routes = [
  { path: '/', component: () => import(/* webpackChunkName: "HelloWorld" */ '@/components/HelloWorld'), props: {currentSection: "microsites"} },
  //   { path: '/microsites', component: footerSite, props: {currentSection: "microsites"} },
  //   { path: '/ai', component: footerSite, props: {currentSection: "ai"} },
  //   { path: '/ar', component: footerSite, props: {currentSection: "ar"} },
  //   { path: '/nft-blockchain', component: footerSite, props: {currentSection: "nft-blockchain"} },
  //   { path: '/cloud-aws', component: footerSite, props: {currentSection: "cloud-aws"} },
  ]

  export default routes;
  ```
- modificare la configurazione di Webpack presente nel file vue.config.js:
  ``` js
  const { name } = require("./package");
  module.exports = {
    devServer: {
      headers: {
        "Access-Control-Allow-Origin": "*",
      },
    },
    configureWebpack: {
      output: {
        library: `${name}-[name]`,
        libraryTarget: "umd", // bundle the micro app into umd library format
        // jsonpFunction: `webpackJsonp_${name}`, → deprecated in Webpack 5
        chunkLoadingGlobal: `jsonp_${name}`,
      },
    },
  };
  ```
- registrare la micro app nell'index.js dell'app container:
  ``` js
  registerMicroApps([
    // ... other routes
    {
      name: "footer",
      entry: "//localhost:8080",
      container: "#root",
      activeRule: "/footer",
      props: { Routerbase: "/footer" },
    }
  ]);
  ```

## Creazione progetto "header" (Angular)
- creare una micro app Angular con il comando `npx ng new header` che utilizzi il routing e che usi CSS come formato per i fogli di stile
- creare il file public-path.js all'interno della cartella "src":
  ``` js
  if (window.__POWERED_BY_QIANKUN__) {
    __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__;
  }
  ```
- settare il path base per la modalità di routing *history* modificando il file src/app/app-routing.module.ts:
  ``` ts
  import { APP_BASE_HREF } from '@angular/common';
  import { NgModule } from '@angular/core';
  import { RouterModule, Routes } from '@angular/router';

  const routes: Routes = [];

  @NgModule({
    imports: [RouterModule.forRoot(routes)],
    exports: [RouterModule],
    // @ts-ignore
    providers: [{ provide: APP_BASE_HREF, useValue: window.__POWERED_BY_QIANKUN__ ? '/header' : '/' }]
  })
  export class AppRoutingModule { }
  ```
- importare il file "public-path.js" ed esportare i tre metodi dei cicli di vita (bootstrap, mount e unmount) dal proprio file entry principale, nel caso di Angular src/main.ts:
  ``` ts
  import './public-path';
  import { /* enableProdMode, */ NgModuleRef } from '@angular/core';
  import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
  import { AppModule } from './app/app.module';
  // import { environment } from './environments/environment';

  // if (environment.production) {
  //   enableProdMode();
  // }

  let app: void | NgModuleRef<AppModule>;
  async function render() {
    app = await platformBrowserDynamic()
      .bootstrapModule(AppModule)
      .catch((err) => console.error(err));
  }
  if (!(window as any).__POWERED_BY_QIANKUN__) {
    render();
  }

  export async function bootstrap(props: Object) {
    console.log(props);
  }

  export async function mount(props: Object) {
    render();
  }

  export async function unmount(props: Object) {
    console.log(props);
    // @ts-ignore
    app.destroy();
  }
  ```
- installare il plugin per configurare Webpack → `npm i @angular-builders/custom-webpack -D`
- creare un file custom-webpack.config.js nella root:
  ``` js
  const appName = require("./package.json").name;
  module.exports = {
    devServer: {
      headers: {
        "Access-Control-Allow-Origin": "*",
      },
    },
    output: {
      library: `${appName}-[name]`,
      libraryTarget: "umd",
      // jsonpFunction: `webpackJsonp_${appName}`, → deprecated in Webpack 5
      chunkLoadingGlobal: `jsonp_${appName}`,
    },
  };
  ```
- modificare il file angular.json cambiando i valori di `[packageName] > architect > build > builder` e `[packageName] > architect > serve > builder` in modo che utilizzino i plugin appena installati e aggiungere il nostro file di configurazione Webpack all'interno di `[packageName] > architect > build > options`.
  ```diff
    "build": {
  -   "builder": "@angular-devkit/build-angular:browser",
  +   "builder": "@angular-builders/custom-webpack:browser",
      "options": {
  +     "customWebpackConfig": {
  +     "path": "./custom-webpack.config.js"
  +     }
      }
    },
    "serve": {
  -   "builder": "@angular-devkit/build-angular:dev-server",
  +   "builder": "@angular-builders/custom-webpack:dev-server",
    }
  ```
- registrare la micro app nell'index.js dell'app container:
  ``` js
  registerMicroApps([
    // ... other routes
    {
      name: "header",
      entry: "//localhost:4200",
      container: "#root",
      activeRule: "/header",
      props: { Routerbase: "/header" },
    }
  ]);
  ```
- per prevenire il conflitto di `<app-root></app-root>` quando anche l'app container o altre micro app sono in Angular, è consigliato aggiungere un id unico ad `<app-root>`:
  - src/index.html
    ``` diff
    - <app-root></app-root>
    + <app-root id="header"></app-root>
    ```
  - src/app/app.component.ts
    ``` diff
    - selector: 'app-root',
    + selector: '#header app-root',
    ```

### Problema *zone.js* 
Nel caso la rotta che renderizza la micro app in Angular avesse dei problemi ("la pagina non risponde"), eliminare *zone.js* e spostarlo nell'app container.
Nello specifico:
- nel progetto Angular, eliminare *zone.js* dai polyfill dell'angular.json, all'interno di `[ packageName] > architect > build > options` e `[ packageName] > architect > test > options`:
  ``` diff
    "build": {
      "options": {
  -     "polyfills": [
  -       "zone.js"
  -     ]
      }
    },
    "test": {
      "options": {
  -     "polyfills": [
  -       "zone.js",
  -       "zone.js/testing"
  -     ]
      }
    }
  ```
- nel progetto Angular, eliminare *zone.js* dalle dipendenze del package.json:
  ```diff
  - "zone.js": "~0.13.0"
  ```
- nel progetto Angular, aggiungere *zone.js* all'interno del tag `<head>` del file src/index.html, affinché venga utilizzato quando si accede alla micro app in maniera indipendente:
  ```diff
  + <!-- Other CDN/local packages can also be used -->
  + <script src="https://unpkg.com/zone.js" ignore></script>
  ```
- nel progetto container, installare *zone.js* → `npm i zone.js -S`
  ```diff
  + "zone.js": "^0.13.1"
  ```

## Server
- http://localhost:3001/ → body in React, se raggiunto così dà pagina bianca, ma su http://localhost:3000/body è renderizzato correttamente
- http://localhost:8080/ oppure http://localhost:3000/footer/ → footer in Vue
- http://localhost:4200/ oppure http://localhost:3000/header/ → header in Angular

## Renderizzazione contemporanea di più micro app
Allo stato attuale, le micro app vengono renderizzate una alla volta e a ciascuna è assegnata una rotta. Per far sì che possano essere mostrate tutte nella stessa rotta, bisogna modificare quattro file.

### `header/src/app/app-routing.module.ts`
``` diff
@NgModule({
-  providers: [{ provide: APP_BASE_HREF, useValue: window.__POWERED_BY_QIANKUN__ ? '/header' : '/' }]
+  providers: [{ provide: APP_BASE_HREF, useValue: '/' }]
})
```

### `footer/src/main.js`
``` diff
function render(props = {}) {
  history = createWebHistory("/");
-   history = createWebHistory(window.__POWERED_BY_QIANKUN__ ? "/footer" : "/");
+   history = createWebHistory("/");
}
```

### `container/public/index.html`
``` diff
- <div id="root"></div>
+ <div id="header"></div>
+ <div id="body"></div>
+ <div id="footer"></div>
```

### `container/public/index.js`
``` diff
- import { registerMicroApps, start, setDefaultMountApp } from "qiankun";
+ import { registerMicroApps, start } from "qiankun";

  registerMicroApps([
    {
      name: "header",
      entry: "//localhost:4200",
-     container: "#root",
+     container: "#header",
-     activeRule: "/header",
+     activeRule: () => true,
-     props: { Routerbase: "/header" },
    },
    {
      name: "body",
      entry: "//localhost:3001",
-     container: "#root",
+     container: "#body",
-     activeRule: "/body",
+     activeRule: () => true,
-     props: { Routerbase: "/body" },
    },
    {
      name: "footer",
      entry: "//localhost:8080",
-     container: "#root",
+     container: "#footer",
-     activeRule: "/footer",
+     activeRule: () => true,
-     props: { Routerbase: "/footer" },
    },
  ]);
- setDefaultMountApp("/body"); // optional

- start();
+ start({ singular: false });
```