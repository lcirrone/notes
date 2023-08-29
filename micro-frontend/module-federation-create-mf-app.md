[← back](module-federation.md) | [home](..\README.md)

# ModuleFederationPlugin, procedimento tramite `create-mf-app`

## Applicazione host e applicazione remota con lo stesso framework

Fonte: https://www.youtube.com/watch?v=s_Fs4AXsTnA&t=2s

- eseguire il comando `npx create-mf-app` per creare l’applicazione container (nome: *host*, porta: 8080) e quella remota (nome: *remote*, porta: 3000) utilizzando la seguente configurazione
    - tipo di progetto: application
    - framework: solid-js
    - linguaggio: javascript
    - css: tailwind
- in un terminale eseguire `cd remote & yarn & yarn start`
- in un altro terminale eseguire `cd host & yarn & yarn start`
- creare un componente Counter.jsx all’interno dell’applicazione *remote*
    
    ```jsx
    import { createSignal } from "solid-js";
    
    export default () => {
      const [count, setCount] = createSignal(0);
    
      return (
        <div>
          <div>Count = {count()}</div>
          <button onClick={() => setCount(count() + 1)}>Add One</button>
        </div>
      );
    };
    ```
    
    e importarlo all’interno di *App.js*
    
    ```jsx
    import Counter from "./Counter";
    
    const App = () => (
      ...
      <Counter />
      ...
    );
    ```
    
- esporre il componente aggiungendolo alla voce "exposes" del ModuleFederationPlugin del *webpack.config.js* dell’applicazione *remote* e far ripartire il server → `yarn start`
    
    ```jsx
    exposes: {
      "./Counter": "./src/Counter.jsx"
    },
    ```
    
    sarà ora disponibile il manifest dei moduli esposti da questo *remote* all’indirizzo http://localhost:3000/remoteEntry.js (se non si è modificato il "filename" all’interno del ModuleFederationPlugin, altrimenti il nome scelto sostituirà "remoteEntry.js")
    
- copiare l'url e incollarlo alla voce "remotes" del ModuleFederationPlugin del *webpack.config.js* dell’applicazione *host*, assicurandosi che il nome precedente alla **@** corrisponda al "name" del ModuleFederationPlugin  dell’applicazione *remote* corrispondente
    
    ```jsx
    remotes: {
      "remote": "remote@http://localhost:3000/remoteEntry.js"
    },
    ```
    
    a questo punto il server dell’applicazione *host* darà errore *'Module not found: Can’t resolve "remote/Counter"'* perché va riavviato → `yarn start`
    

## Applicazione host e applicazione remota con framework diversi

- creare un’altra applicazione host, stavolta in react (nome: *react-host*, porta: 3001, framework: react) → `npx create-mf-app`
- copiare e incollare alla voce "remotes" del ModuleFederationPlugin del *webpack.config.js* dell’applicazione *react-host* lo stesso remote usato nella *host*, per poter accedere al componente Counter
    
    ```jsx
    remotes: {
      "remote": "remote@http://localhost:3000/remoteEntry.js"
    },
    ```
    
In questo caso, trattandosi di un framework diverso, bisogna avere una funzione wrapper che permetta di usare un componente Solid.js all’interno di React, se infatti provassimo a importare il componente come abbiamo fatto nell’applicazione *host* riceveremmo l’errore *"Objects are not valid as a React child (found: [object HTMLDivElement])"*

- creare il file *counterWrapper.jsx* all’interno dell’src dell’applicazione *remote*:
    
    ```jsx
    import { render } from "solid-js/web";
    
    import Counter from "./Counter";
    
    export default (el) => {
      render(Counter, el);
    };
    ```
    
- esporre il componente aggiungendolo alla voce "exposes" del ModuleFederationPlugin del *webpack.config.js* dell’applicazione *remote* e far ripartire il server → `yarn start`
    
    ```jsx
    exposes: {
      "./Counter": "./src/Counter.jsx",
      "./counterWrapper": "./src/counterWrapper.jsx"
    },
    ```
    
- abbiamo quindi ora a disposizione la funzione **mount()** che monta, appunto, il componente in Solid.js. Trattandosi di una funzione e non già del componente pronto, bisogna triggerarla al montaggio dell'applicazione *react-host*. Modificare, quindi, *App.jsx* dell’applicazione react-host in modo che renderizzi il nuovo componente counterWrapper al primo render all’interno di un div tramite l’utilizzo di una ref e far ripartire il server → `yarn start`
    
    ```jsx
    import React, { useRef, useEffect } from "react";
    import ReactDOM from "react-dom";
    
    import counterWrapper from "remote/counterWrapper";
    
    import "./index.scss";
    
    const App = () => {
      const divRef = useRef(null);
    
      useEffect(() => {
        counterWrapper(divRef.current)
      }, [])
    
      return (
        <div className="mt-10 text-3xl mx-auto max-w-6xl">
          <div>Name: react-host</div>
          <div ref={divRef}></div>
        </div>
      );
    };
    ReactDOM.render(<App />, document.getElementById("app"));
    ```
    
## React host + React body + Vue footer + Angular header

Procediamo ora con la creazione di un microfrontend composto da:
- *host* in React
- *body* in React
- *footer* in Vue
- *header* in Angular

Utilizziamo lo stesso procedimento visto in precedenza per quanto riguarda applicazione host, body e footer.

## Vue

All’interno dell'applicazione Vue, nella cartella *src* crearne una ulteriore, *components*, in cui inserire un componente *Footer.vue*:

```jsx
<template>
    <div>FooterSite</div>
</template>
```

Modificare il file *bootloader.js* nel seguente modo, per permettere di esportare la funzione **mount(el)** che renderizzi il componente Footer all’interno dell’elemento desiderato (come nel caso del counterWrapper dell’esempio precedente in Solid.js)

```jsx
import { createApp } from "vue";
import Footer from "./components/Footer.vue";
import "./index.css";

const mount = (el) => {
  const app = createApp(Footer);
  app.mount(el);
};

if (process.env.NODE_ENV === "development") {
  const devRoot = document.querySelector("#app");
  if (devRoot) {
    mount(devRoot);
  }
}

export { mount };
```

Esporre il componente aggiungendolo alla voce "exposes" del ModuleFederationPlugin del *webpack.config.js* dell’applicazione *footer* e far ripartire il server → `yarn start`

```jsx
exposes: {
  "./Footer": "./src/bootloader.js"
},
```

Sarà ora disponibile il manifest dei moduli esposti da questo *remote* all’indirizzo http://localhost:3001/remoteEntry.js (se non si è modificato il "filename" all’interno del ModuleFederationPlugin, altrimenti il nome scelto sostituirà "remoteEntry.js").

Collegare questo nuovo remote all’applicazione *host* copiando e incollando questo url alla voce "remotes" del ModuleFederationPlugin del *webpack.config.js* dell’applicazione *host* per poter accedere al componente Footer. Come sempre, assicurarsi che il nome precedente alla **@** corrisponda al "name" del ModuleFederationPlugin  dell’applicazione *remote* corrispondente:

```jsx
remotes: {
  "body": "body@http://localhost:3000/remoteEntry.js",
  "footer": "footer@http://localhost:3001/remoteEntry.js"
}
```

Modificare *App.jsx* dell’applicazione *host* in modo che renderizzi il nuovo componente utilizzando la funzione esportata **mount()** (1) oppure importando un componente wrapper (2) e far ripartire il server → `yarn start`

1. App.jsx utilizzando la funzione esportata **mount()**

    ```jsx
    import React, { useEffect, useRef } from "react";
    import ReactDOM from "react-dom";

    import Body from "body/Body";
    import { mount } from "footer/Footer";

    import "./index.css";

    const App = () => {
    const footerRef = useRef(null);

    useEffect(() => {
        mount(footerRef.current);
    }, []);

    return (
        <div className="container">
        <div>Name: host</div>
        <Body />
        <div ref={footerRef}></div>
        </div>
    );
    };
    ReactDOM.render(<App />, document.getElementById("app"));
    ```

2. App.jsx importando un componente wrapper → soluzione più pulita

    ```jsx
    import React from "react";
    import ReactDOM from "react-dom";

    import Body from "body/Body";

    import "./index.css";
    import FooterWrapper from "./wrappers/footerWrapper";

    const App = () => {
    return (
        <div className="container">
        <div>Name: host</div>
        <Body />
        <FooterWrapper />
        </div>
    );
    };
    ReactDOM.render(<App />, document.getElementById("app"));
    ```

    FooterWrapper.jsx

    ```jsx
    import { mount } from "footer/Footer";
    import React, { useRef, useEffect } from "react";

    const FooterWrapper = () => {
    const ref = useRef(null);

    useEffect(() => {
        mount(ref.current);
    }, []);

    return <div ref={ref} />;
    };

    export default FooterWrapper;
    ```

## Angular

Per quanto riguarda il progetto dell'header in Angular, non è previsto il comando `create-mf-app`.

Creare una cartella *header* clonandoci dentro https://github.com/BeijePeopleFirst/DEMO-BeijeMicrosite-HeaderSite/tree/header-webpack

**N.B.** prestare particolare attenzione al *webpack.config.js* e al componente esportato, stesso concetto di Vue: non esponi il componente finale, ma il file che esporta la funzione **mount()**!

Collegare questo nuovo remote all’applicazione *host* copiando e incollando alla voce "remotes" del ModuleFederationPlugin del *webpack.config.js* dell’applicazione *host* l’url generato dopo aver esposto il componente (se non si è modificato il "filename" all’interno del ModuleFederationPlugin, di default sarà "remoteEntry.js"), per poter accedere al componente Header:

```jsx
remotes: {
  "body": "body@http://localhost:3000/remoteEntry.js",
  "footer": "footer@http://localhost:3001/remoteEntry.js",
  "header": "header@http://localhost:3002/remoteEntry.js"
}
```

Modificare *App.jsx* dell’applicazione *host* in modo che renderizzi il nuovo componente importando un componente wrapper e far ripartire il server → `yarn start`

- App.jsx

    ```jsx
    import React from "react";
    import ReactDOM from "react-dom";

    import Body from "body/Body";

    import "./index.css";
    import FooterWrapper from "./wrappers/FooterWrapper";
    import HeaderWrapper from "./wrappers/HeaderWrapper";

    const App = () => {
    return (
        <div className="container">
        <div>Name: host</div>
        <HeaderWrapper />
        <Body />
        <FooterWrapper />
        </div>
    );
    };
    ReactDOM.render(<App />, document.getElementById("app"));
    ```

- HeaderWrapper.jsx

    ```jsx
    import { mount } from "header/AppModule";
    import React, { useEffect } from "react";

    const HeaderWrapper = () => {
    useEffect(() => {
        mount();
    }, []);
    return (
        <div className="remote-module">
        <app-root></app-root>
        </div>
    );
    };

    export default HeaderWrapper;
    ```
