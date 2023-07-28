[← back](README.md)

# create-single-spa

https://single-spa.js.org/docs/create-single-spa/ → utility da linea di comando per generare un boilerplate da cui partire, per chi preferisce configurazioni autogenerate e gestite per webpack, babel, jest, ecc.

`npm install --global create-single-spa`

## Quick start

[Quick start → root config + single-spa application](https://single-spa.js.org/docs/getting-started-overview#quick-start)→ creazione di root-config e delle singole applicazioni single-spa

### *root-config*

- `create-single-spa --moduleType root-config`
- `yarn install`
- `yarn start`
- http://localhost:9000/ → app contenitore, la guida lì presente non è aggiornata quindi è meglio non seguirla

### applicazione single-spa

procedimento da ripetere per ogni micro applicazione, quindi scegliendo ogni volta il framework corrispondente in fase di creazione

- `create-single-spa --moduleType app-parcel`
- `yarn install`
- `yarn start` → se è un’app in Angular, si vedrà pagina bianca, andare nei Dev Tools > Network > main.js > Headers e copiare il Request URL, altrimenti copiare il link indicato in pagina e inserirlo nell’app contenitore:
    - *src/index.ejs* → alla voce `<% if (isLocal) { %>` all’interno degli *imports* dello `<script type="systemjs-importmap">` aggiungere le applicazioni che si vogliono includere
        
        ```jsx
        "@beije/angular-app-test": "http://localhost:4200/main.js",
        "@beije/react-app-test": "http://localhost:8080/beije-react-app-test.js",
        "@beije/vue-app-test": "http://localhost:8081/js/app.js"
        // ecc.
        ```
        
        chiave → deve coincidere con il *name* presente nel *package.json* dell’applicazione single-spa che si vuole includere
        
        valore → l’url copiato prima, che punta al file in cui sono presenti i metodi per i cicli di vita
        
    - sempre all’interno di _src/index.ejs_ → se si sta creando un’applicazione in Angular va decommentata la riga `<script src="https://cdn.jsdelivr.net/npm/zone.js@0.11.3/dist/zone.min.js"></script>` come spiegato nei commenti ([qui](https://single-spa.js.org/docs/ecosystem-angular/#zonejs) per ulteriori informazioni)
    - *src/microfrontend-layout.html* → all’interno della `<route default>` sostituire l’`<application>` presente con quelle che si vogliono includere:
        
        ```html
        <application name="@beije/react-app-test"></application>
        <application name="@beije/vue-app-test"></application>
        <application name="@beije/angular-app-test"></application>
        <!-- ecc. -->
        ```
        
- `yarn start:standalone`
- http://localhost:8080/ → applicazione funzionante

## Problemi conosciuti

- errore nell’applicazione single-spa in **Vue**: **meta undefined tag** → all’interno di _vue.config.js_ aggiungere nel defineConfig
    
      `configureWebpack: {
        output: {
          libraryTarget: 'system',
        }
      }`
    
    se non basta, all’interno di _webpack.config.js_ dell’applicazione contenitore aggiungere al merge() delle configurazioni:
    
        `output: {
          libraryTarget: "system"
        }`
    
- errore Cross-origin resource sharing (**CORS**) nell’applicazione **contenitore** sull’**accedere al localhost** → in *index.ejs*, modificare il _<meta content=”…”>_ aggiungendo il proprio indirizzo ip al “connect-src”: `connect-src https: localhost:* ws://localhost:* ws://192.168.1.91:8081/ws;`
- errore Cross-origin resource sharing (**CORS**) nell’applicazione **contenitore** sull’**accedere ai path delle immagini in localhost** → in *index.ejs*, modificare il *<meta content=”…”>* aggiungendo `img-src * 'self' data: https:;`
- applicazione nell’applicazione single-spa in Angular: `**import { environment } from './environments/environment';` missing** → in *src* creare *environments/environment.ts* contenente:
    
    ```tsx
    export const environment = {
      production: false,
      apiKey: "devKey",
    };
    ```
## Demo

Clonare i branch *single-spa* delle seguenti repository:
- [DEMO-BeijeMicrosite-Container](https://github.com/BeijePeopleFirst/DEMO-BeijeMicrosite-Container)
- [DEMO-BeijeMicrosite-FooterSite](https://github.com/BeijePeopleFirst/DEMO-BeijeMicrosite-FooterSite)
- [DEMO-BeijeMicrosite-HeaderSite](https://github.com/BeijePeopleFirst/DEMO-BeijeMicrosite-HeaderSite)
- [DEMO-BeijeMicrosite-BodySite](https://github.com/BeijePeopleFirst/DEMO-BeijeMicrosite-BodySite)

Far partire i singoli progetti, *container* per ultimo:
- footer (Vue) -> `yarn serve:standalone` -> http://localhost:8081/
- header (Angular) -> `npm run serve:single-spa:header-site` -> http://localhost:4200/ (pagina bianca, ma funziona, vedi sopra)
- body (React) -> `yarn start:standalone` -> http://localhost:8080/
- container (React) -> `cd shell & yarn start` -> http://localhost:9000/
