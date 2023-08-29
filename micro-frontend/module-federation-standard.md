[← back](module-federation.md) | [home](..\README.md)

# ModuleFederationPlugin, procedimento standard

Fonte: https://medium.com/@samho1996/microfrontend-with-module-federation-in-react-98b72b347238

## Creazione applicazioni

- creare due progetti in React, uno per il container e uno per il componente da esportare → `npx create-react-app NOME` 
**N.B.** se voglio evitare che mi chieda ogni volta se usare la stessa porta dell’altra app o una diversa, modificare lo script `start` all’interno del *package.json → `"start": "npx cross-env PORT=3001 react-scripts start"`*
- per evitare il warning sulla libreria “@babel/plugin-proposal-private-property-in-object” aggiungerla all’interno delle devDependencies nei package.json:
    
    ```json
    "devDependencies": {
        "@babel/plugin-proposal-private-property-in-object": "7.21.0-placeholder-for-preset-env.2"
      },
    ```
    

## Installazione di Webpack

- installare Webpack in entrambi i progetti
`npm install –-save-dev webpack webpack-cli html-webpack-plugin webpack-dev-server babel-loader css-loader`

**N.B.** assicurarsi che sia presente `import React from 'react'` nei componenti App.js di entrambi i progetti, altrimenti Webpack non funziona!

## Configurazione di Webpack

- creare un file *webpack.config.js* nella root di entrambi i progetti modificando riga 8 relativa alla porta da usare per il *devServer*:
    
    ```jsx
    //home-app/webpack.config.js
    const HtmlWebpackPlugin = require("html-webpack-plugin");
    
    module.exports = {
      entry: "./src/index",
      mode: "development",
      devServer: {
        port: 3000, // port 3001 for react-app
      },
      module: {
        rules: [
          {
            test: /\.(js|jsx)?$/,
            exclude: /node_modules/,
            use: [
              {
                loader: "babel-loader",
                options: {
                  presets: ["@babel/preset-env", "@babel/preset-react"],
                },
              },
            ],
          },
          {
            test: /\.css$/i,
            use: ["style-loader", "css-loader"],
          },
        ],
      },
      plugins: [
        new HtmlWebpackPlugin({
          template: "./public/index.html",
        }),
      ],
      resolve: {
        extensions: [".js", ".jsx"],
      },
      target: "web",
    };
    ```
    
- rimuovere **reportWebVitals** dagli *index.js* di entrambi i progetti
- rimuovere `<link>` e `<meta>` dagli *index.html* di entrambi i progetti
- cambiare gli script di “start” e “build” all’interno del *package.json* di entrambi i progetti:
    
    ```json
    "scripts": {
        "start": "webpack serve",
        "build": "webpack --mode production",
     },
    ```
    
- far partire entrambi i progetti → `cd NOME && yarn start`

## Configurazione di Module Federation

- creare un file *entry.js* nella cartella *src* di entrambi i progetti → `import('./index.js')`
- modificare la proprietà “entry” del *webpack.config.js* di entrambi i progetti
    
    ```jsx
    module.exports = {
        entry: "./src/entry.js",
        //...
    }
    ```
    

## Messa a disposizione dei componenti per Module Federation

Bisogna ora rendere disponibile il componente in React (*reactapp* nel nostro esempio)

- nel *webpack.config.js* assicurarsi di avere/aggiungere il seguente codice e far ripartire il server. Se si raggiunge http://localhost:3001/remoteEntry.js è possibile visualizzare un manifest di tutti i moduli esposti dall’applicazione *reactapp*. **N.B.** controllare che nomi di applicazioni e componenti da esporre combacino!
    
    ```jsx
    // reactapp/webpack.config.js
    const HtmlWebpackPlugin = require("html-webpack-plugin");
    
    // import ModuleFederationPlugin from webpack
    const { ModuleFederationPlugin } = require("webpack").container;
    // import dependencies from package.json, which includes react and react-dom
    const { dependencies } = require("./package.json");
    
    //...
    
    module.exports = {
        //...
        plugins: [
            //...
            new ModuleFederationPlugin({
    	      name: "reactapp", // This application named 'reactapp'
    	      filename: "remoteEntry.js", // output a js file
    	      exposes: {
    	        // which exposes
    	        "./Header": "./src/App", // a module 'Header' from './src/App'
    	      },
    	      shared: {
    	        // and shared
    	        ...dependencies, // some other dependencies
    	        react: {
    	          // react
    	          singleton: true,
    	          requiredVersion: dependencies["react"],
    	        },
    	        "react-dom": {
    	          // react-dom
    	          singleton: true,
    	          requiredVersion: dependencies["react-dom"],
    	        },
    	      },
    	    }),
        ],
    };
    ```
    

## Aggiungere Module Federation all’app container

- aggiungere ModuleFederationPlugin nel *webpack.config.js* dell’app container, **N.B.** controllare che nomi di applicazioni e componenti combacino!
    
    ```jsx
    // container/webpack.config.js
    const HtmlWebpackPlugin = require("html-webpack-plugin");
    
    // import ModuleFederationPlugin from webpack
    const { ModuleFederationPlugin } = require("webpack").container;
    // import dependencies from package.json, which includes react and react-dom
    const { dependencies } = require("./package.json");
    
    //...
    
    module.exports = {
        //...
        plugins: [
            //...
            new ModuleFederationPlugin({
    	      name: "container", // This application named 'container'
    	      // This is where we define the federated modules that we want to consume in this app.
    	      // Note that we specify "Header" as the internal name
    	      // so that we can load the components using import("Header/").
    	      // We also define the location where the remote's module definition is hosted:
    	      // reactapp@[http://localhost:3001/remoteEntry.js].
    	      // This URL provides three important pieces of information: the module's name is "reactapp", it is hosted on "localhost:3001",
    	      // and its module definition is "remoteEntry.js".
    	      remotes: {
    	        reactapp: "reactapp@http://localhost:3001/remoteEntry.js",
    	      },
    	      shared: {
    	        // and shared
    	        ...dependencies, // other dependencies
    	        react: {
    	          // react
    	          singleton: true,
    	          requiredVersion: dependencies["react"],
    	        },
    	        "react-dom": {
    	          // react-dom
    	          singleton: true,
    	          requiredVersion: dependencies["react-dom"],
    	        },
    	      },
    	    }),
        ],
    };
    ```
    
- modificare il componente *App.js* dell’app container perché importi il componente Header appena esposto dalla react-app e far ripartire entrambi i server
    
    ```jsx
    import React, { lazy, Suspense } from 'react'; // Must be imported for webpack to work
    import './App.css';
    
    const Header = lazy(() => import('HeaderApp/Header'));
    
    function App() {
      return (
        <div className="App">
          <Suspense fallback={<div>Loading Header...</div>}>
            <Header />
          </Suspense>
          <div className="container">Demo home page</div>
        </div>
      );
    }
    
    export default App;
    ```
