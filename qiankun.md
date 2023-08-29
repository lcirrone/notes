# Qiankun

### Fonti utilizzate
- https://medium.com/@fibonalabsdigital/how-to-implement-micro-frontends-using-qiankun-9f308eddc5f4
- https://qiankun.umijs.org/guide/tutorial

## Creazione progetto "container" (React)
- `npx create-react-app container`
- aggiungere al package.json la dipendenza dev "@babel/plugin-proposal-private-property-in-object" per evitare il warning
  ``` json
  "devDependencies": {
    "@babel/plugin-proposal-private-property-in-object": "^7.21.0"
  }
  ```
- installare la dipendenza Qiankun, necessaria solo nel progetto container -> `npm i qiankun -S`
- rimuovere App.js, App.css, App.test.js e index.css

## Creazione progetto "body" (React)
- `npx create-react-app body`
- modificare lo script "start" all'interno del package.json perché l'app usi una porta diversa da quella di default (la 3000, già usata dal progetto container)
  ``` json
  "scripts": {
    "start": "npx cross-env PORT=3001 react-scripts start",
    "...": "..."
  }
  ```
- ogni micro app deve esportare i tre hook/metodi dei cicli di vita (bootstrap, mount e unmount) dal proprio file entry principale, nel caso di React index.js:
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
  e importarlo all'interno di index.js -> `import "./public-path";`
  
  **N.B.** la riga `// eslint-disable-next-line` è necessaria per evitare l'errore `__webpack_public_path__ is not defined`

### Configurazione Webpack
Dato che l'applicazione container e le micro app girano su porte diverse, bisogna sovrascrivere la configurazione del dev server perché i browser non permettono l'accesso da una porta all'altra. Per farlo si può utilizzare "react-app-rewired" o "@rescripts/cli".
- `npm i react-app-rewired -S`
- nella root creare un file config-overrides.js con la seguente configurazione:
  ``` js
  const { name } = require("./package");

  module.exports = {
    webpack: (config) => {
      config.output.library = `${name}-[name]`;
      config.output.libraryTarget = "umd";
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
  **N.B.** la riga `config.output.chunkLoadingGlobal = `jsonp_${name}`;` era in precedenza `config.output.jsonpFunction = `webpackJsonp_${name}`;`, ma con WebPack 5 va aggiornata
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

## Configurazione applicazione principale
- modificare l'index.js dell'applicazione container nel seguente modo:
  ``` js
  import { registerMicroApps, start, setDefaultMountApp } from "qiankun";

  registerMicroApps([
    // {
    //   name: 'header',
    //   entry: '//localhost:4200',
    //   container: '#root',
    //   activeRule: '/header',
    //   props: { Routerbase: "/body" },
    // },
    {
      name: "body", // app name registered should match name in package.json of microapp1
      entry: "//localhost:3001", // where our microapp1 exists
      container: "#root", // div which exists in main app with id root, so inside this div our micro app will render
      activeRule: "/body", // our microapp will be visible in main app under this path
      props: { Routerbase: "/body" }, // used by qiankun for routing purpose
    },
    // {
    //   name: "footer",
    //   entry: "//localhost:8081",
    //   container: "#root",
    //   activeRule: "/footer",
    //   props: { Routerbase: "/body" },
    // },
  ]);
  setDefaultMountApp("/body"); // optional
  // by default, if you want to display a micro app you can use this
  /// in the above case by default, if we enter it will route localhost:3000 to localhost:3000/body
  
  // start qiankun
  start();
  ```
- aaa
