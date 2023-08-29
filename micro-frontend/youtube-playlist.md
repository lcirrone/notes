[← back](..\README.md)

# [YouTube Playlist from the Core Team](https://www.youtube.com/playlist?list=PLLUD8RtHvsAOhtHnyGx57EYXoaNsxGrTU)

## [Microfrontends & single-spa](https://www.youtube.com/watch?v=3EUfbnHi6Wg&list=PLLUD8RtHvsAOhtHnyGx57EYXoaNsxGrTU&index=3)

With the single-spa recommended setup,  each microfrontend is one in-browser JavaScript module. If you have a look at the Dev Tools Network tab there’s a **one to one relationship between a microfrontend and a in-browser js module → they’re all separate network requests**.

Within the context of single-spa there are three kinds of microfrontends:

- **single spa applications** → a chunk of ui that is in charge of a url route or routes, e.g. *esm-login*
- **single spa parcels** → a chunk of ui that is not in charge of any url routes
- **helper modules** → not a chunk of ui/ui component, it’s a in-browser js module that is just js code, doesn’t create dom element, there’s no react/vue/angular inside, just a js module that helps you do something, e.g. *esm-error-handling* (exports functions that helps capturing errors and reporting them to the user and the server successfully) or *esm-api* (helps with authentication, authorization, permissioning and just communication making API requests to the server)

## [Javascript tutorial: in-browser versus build-time modules](https://www.youtube.com/watch?v=Jxqiu6pdMSU&list=PLLUD8RtHvsAOhtHnyGx57EYXoaNsxGrTU&index=2)

A **module** in js is when you use an *import* and *export* keywords between multiple files

**In-browser vs. build-time modules** always comes down to your bundler → *[webpack](https://webpack.js.org/)* or *[rollup](https://rollupjs.org/)* are the two most popular bundlers:

- **in-browser module** → if you have source code and output bundle, in the output bundle anytime you still see an import keyword (e.g. `import React from 'react'`) that means there’s a dependency on an in-browser module so we’re asking the browser to give us React → **all separate network requests**
- **build-time module** → in the source code we have `import foo from './foo'`, but there’s no import for *foo* in output bundle, *foo* itself got inlined. Bundlers (both rollup, webpack, parcel, and others) are always deciding “should I make it an in-browser module or a build-time module?” → **one network request that downloads all of it** and that one network request would contain all of the code for all of the requests we’ve seen before

**In-browser means we preserve the import vs. build-time means we just actually plop it right into the same file without it being a separate file.**

Webpack and rollup are usually configured so that almost everything is a build-time module, in fact majority of code in production in 2020 is build-time modules, people aren’t using in-browser modules very much.

## [Javascript tutorial: import maps](https://www.youtube.com/watch?v=Lfm2Ge_RUxs&list=PLLUD8RtHvsAOhtHnyGx57EYXoaNsxGrTU&index=4)

`import 'https://cdn.jsdelivr.net/...'` → is the only way to use js modules in a browser,

`import 'single-spa'` wouldn’t work

 This is part of the ES6 specification, as of the beginning of 2020 in browsers you cannot import a module simply by specifying its name, you can only use a url, either absolute or relative, but it has to be a url. This is a pretty big, and known, js limitation, so they created a browser specification to solve it: **import maps** → script elements inside HTML files that have to have a `type="importmap"` and contain a JSON object.

Three major variations of import maps:

- **inline import maps** → this way we are able to `import 'single-spa'`
    
    ```jsx
    <script type="importmap">
        {
          "imports": {
            "single-spa": "https://cdn.jsdelivr.net/npm/single-spa@5.9.0/lib/system/single-spa.min.js",
          }
        }
      </script>
    ```
    
- **external import maps** → the import map has to be hosted on a server and you’re basically telling the browser to download the import map and it will tell you all the names of the modules and the URLs for those modules
    
    ```jsx
    <script type="importmap" src="https://cdn.jsdelivr.net/npm/single-spa@5.9.0/lib/system/single-spa.min.js"></script>
    ```
    
- **scopes** → supporting multiple versions of the same module within the same page. You might have some modules that require older versions of the same module so you’re forced to have multiple versions in the page, not great for performance, however import maps allow you to do this. The scope matches the URL of the requester, meaning that if the code that is attempting to load *lodash* was itself downloaded on a URL that matches the scope (if you have a module called “a” in a “module-a” folder) then it will get lodash 4, however any other `import lodash` will get it from lodash 3
    
    ```jsx
    <script type="importmap">
      {
        "imports": {
          "lodash": "https://unpkg.com/lodash@3",
        },
        "scopes": {
          "/module-a/": {
            "lodash": "https://unpkg.com/lodash@4"
          }
        }
      }
    </script>
    ```
    

**Browser compatibility = bad news!**

As of the beginning of 2020 you cannot use import maps in browsers, Chrome has an experimental implementation that is hidden behind a feature flag not turned on for users. Firefox, Safari and Edge have not implement import maps at all.

If you’re looking to use in-browser js modules then import maps are the key to a much much better developer experience, however if you want to use import maps you actually can’t, so you’re gonna have to go for a polyfill. Among them, *[System JS](https://github.com/systemjs/systemjs)* → allows you to use all the features of import maps without having (since it’s a polyfill) to get your browser to do it.

There is a System JS import map that is pointing to an external import map:

```jsx
<script type="systemjs-importmap" src="https://storage.googleapis.com/react.microfrontends.app/importmap.json"></script>
```

## [Javascript tutorial: local development with microfrontends, single-spa, and import maps](https://www.youtube.com/watch?v=vjjcuIxqIzY&list=PLLUD8RtHvsAOhtHnyGx57EYXoaNsxGrTU&index=4)

This is part of the single-spa recommended setup

Local development → should we boot  up all the micro frontends or just the one that we’re working on. Let’s say you have a big team, 20 developers, 10 micro frontends, so 10 git repos, 10 webpack configs, 10 CI processes, 10 npm starts, so for local development booting up all 10 would mean *npm install* and *npm start* 10 times and 10 tabs → that’s a pain

Additionally you’re gonna have to be pulling a lot in order to make sure your code is always up to date and it’s also sort of a pain just because some developers don’t work on the same stuff as the other developers, like, not even the same projects/teams, so why are they having to constantly deal with “your thing is broken”, “why do I have to pull, reinstall, keep things up to date?” → it becomes a lot to manage when you have 10 repos that all of your developers have to keep going

So, this requires a bit of upfront work but it’s worth it, 2 steps:

- you must have a deployed test environment → not your localhost, different URL, it has your backend and your frontend running
- you must use use a library called *import-map-overrides*

```jsx
<script src="https://cdn.jsdelivr.net/npm/import-map-overrides@2.2.0/dist/import-map-overrides.js"></script>
```

We have a script in our HTML file that points to *import-map-overrides* → you can load it from CDN or from your node modules, however you want. You have to put in in a particular place: it **must be after your main import map, but before any System.import()**, very important!

It provides you a UI that shows you all the modules that you have, your in-browser modules, and lets you click on one you want to override and change the URL for that particular in-browser module, e.g. you click on *@openmrs/esm-login* and instead of the digital ocean CDN you want to use localhost → `https://localhost:8080/open-mrs-esm-login.js` or without the *https://* so `//localhost:8080/open-mrs-esm-login.js` or just the port `8080` and it will assume the default URL. You can even configure what the default URL is for a port.

You can override more than one module at a time or just one, as you want. Micro frontends really works best when you’re overriding one at a time or maaaybe two. If you’re overriding 5 or 6 or 10 at a time then that’s problematic and it’s causing a lot of work for your developers.

When you click around in this UI or when you use a js API `importMapOverrides.getOverrideMap()` it’s actually manipulating local storage → if you open the Dev Tools Storage tab you’ll see some import map override keys with some values that are the URLS. *import-map-overrides* checks local storage when the page is loading and it injects a new import map in addition to the one you already have being loaded in the HTML

## [Deploying Microfrontends Part 1 - Import Map Deployer](https://www.youtube.com/watch?v=QHunH3MFPZs&list=PLLUD8RtHvsAOhtHnyGx57EYXoaNsxGrTU&index=5)

How to do CI, deployments and releases for single-spa and micro frontends

You have some js files, some css files, some html files, some static assets that you need to have hosted somewhere, so you can use s3 or cloud storage, nginx and a server, a node express server with the static directory that is serving up… ultimately you just have some js files, they need to be referenced from the html files so you have one html file for single page app and that single page app needs to load some of the js files. Deployment is swapping out your js files, you can either swap them or upload new ones, the point is as long as the customer’s getting new code.

Import maps are the center of deployments with the single spa core team’s recommended setup. The html file says: download this import map, and then load up the root and the the root’s going to download all the rest of them. As long as the URL for the root is correct that’s it.

An independent deployment for a module X is simply updating the URL to point to a new URL in this import map. There’s one import map for all production and you update the URL in that import map to point to a different file, that file has some new code and it’s on some server CDN that has your code. You upload some new js, you change your import map to point to that js.

**[Import map deployer](https://github.com/single-spa/import-map-deployer)** → in a large organization you’re going to have lots of deployments to test environments, and you need to make sure that if two deployments happen at the sime time you handle concurrency and the import map deployer does that. It ensures that if you try to update the import map at the same time no weird behaviour occurs, both deployments will be successful. You can deploy forward and roll back all with concurrent safety. It’s a backend service, there’s a docker image for it, it turns out to be a nodeJS server, but it’s a backend HTTP server that you can use via Docker and it **exposes REST APIs to view and modify the import map** for production environment.

**Environments** → you can have a production environment and a test environment

- you can get the import map
- you can update the import map
- you can update an individual module or service within the import map
- you can delete things from the import map
- etc.

When you call the import map deployer it will update the import map JSON file, it exists for concurrent safety!

We need to deploy the import map deployer in order for it to be running so that we can then call it and verify that it is able to read and update an import map.

- it can be done on AWS, Digital Ocean… now Google Cloud + Kubernetes to run the import map deployer as a dock container (or ask a dev ops to make your docker image run and be publicly accessible through a public URL). In Google Cloud create a Kubernetes cluster (with **1 node** because the import map deployer uses a process lock to do its concurrency stuff and this means you need to have exactly one of them. If you have duplicates, it no longer guarantees concurrent safety) and add **some specific permissions** for our node pool in order for it to work →  under the “Management” section you need to choose “Set access for each API” as “Access scopes”, we need permissions to cloud storage so we’re going to put an import map JSON file into cloud storage. Cloud storage is like s3 if you’ve used AWS and we also need a Kubernetes docker container cluster thing node pool to have access to read and write to cloud storage. So “Computer Engine → Full” (probably only Read Write). If you forget to do this here you can’t change it later within Google Cloud!
- check that “config.json” is correct → “locations” refers to the environments and the URL has to be the name of a cloud storage bucket, so create one (in Google Cloud create a Storage bucket (location type: region, access-control: uniform) and use that URL there. Here you can also password protect it with username and password

```jsx
{
  "manifestFormat": "importmap",
  "locations": {
    "test": "google://youtube-imd-test/importmap.json"
  }
}
```

- clone the import map deployer repo and build it locally (or it’s also available on docker hub)
`docker build .` → creates our Docker image
- tag the docker image
`docker tag f21dc83c2b0d us.gcr.io/neural-passkey-248222/youtube-import-map-deployer`
- push the docker image to Google’s Docker Container Registry, it will allow us to create a deploument later within the cloud console
`docker push us.gcr.io/neural-passkey-248222/youtube-import-map-deployer`
- when it’s done and the Kubernetes cluster has built, press the Deploy button and choose an Existing container image by selecting the docker image that has just been pushed to the Google Container Registry, Done, Continue and then Deploy → it will run the Docker container inside of the Kubernetes cluster that we’ve just created. Kubernetes is not mandatory here, we can use AWS, ECS, you don’t even have to use Docker if you don’t want to. The Google Kubernetes engine is pretty easy for people who’re not expert at load balancing stuff and there’s a button we can press that will just create a public URL for our docker container.
- have an empty *importmap.json* file → `{"imports": {}}`
- make sure there’s only one pod, and then press the Expose button → choose the same port as stated in the *README.md* file, should be 5000, Done and then Expose → we now have a Docker container running inside the Kubernetes cluster, a public IP address (look for the “External endpoint”) for it. It won’t be able to read or modify the import map yet because it doesn’t exist
- navigate to `35.188.106.70:5000` and do a health check, it should print “everything okay”, then navigate to `35.188.106.70:5000/import-map.json?env=test`
- in Google Cloud > Storage > Browser > the bucket you created > Upload files and choose your *importmap.json* → it’s not publicly accessible, but Cloud Storage/Kubernetes cluster should have access to it. Then we’re going to make it publicly accessible for reading: Permissions > Add members > New members: allUsers, Select a role: Storage > Storage Object Viewer > Save and then copy the public link
- now let’s write to it: `http PATCH http://35.188.106.70:5000/services\?env=test service=react url=https:/cdn.jsdelivr.net/npm/react/umd/react.development.js` → go back to the browser and you should see the updated import map

We’ve now set up the backed stuff needed in order to actually deploy things
