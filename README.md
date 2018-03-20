## Primero instala las dependencias

> `npm i --save @nguniversal/common @nguniversal/express-engine @nguniversal/module-map-ngfactory-loader @angular/platform-server`

> `npm i --save-dev ts-loader@2.3.7` (es importante instalar esta versión, con la última todo se rompe)


## Ahora modifica la aplicación actual

1. En `src/app/app.module.ts`

```js
import { TransferHttpCacheModule } from '@nguniversal/common';
//...

  imports: [
    BrowserModule.withServerTransition({appId: 'mi-aplicacionnnnnnnnnn'}),
    TransferHttpCacheModule
  ],

//...
```
2. Crear el archivo `src/app/app.server.module.ts`

```js
import { NgModule } from '@angular/core';
import { ServerModule, ServerTransferStateModule } from '@angular/platform-server';
import { ModuleMapLoaderModule } from '@nguniversal/module-map-ngfactory-loader';

import { AppModule } from './app.module';
import { AppComponent } from './app.component';

@NgModule({
  imports: [
    AppModule,
    ServerModule,
    ModuleMapLoaderModule,
    ServerTransferStateModule,
  ],
  bootstrap: [AppComponent],
})
export class AppServerModule {}

```
3. Crear el archivo `src/main.server.ts`

```js
export { AppServerModule } from './app/app.server.module';

```

4. Crear el archivo `src/tsconfig.server.json`
```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "outDir": "../out-tsc/app",
    "baseUrl": "./",
    "module": "commonjs",
    "types": [
      "node"
    ]
  },
  "exclude": [
    "test.ts",
    "**/*.spec.ts"
  ],
  "angularCompilerOptions": {
    "entryModule": "app/app.server.module#AppServerModule"
  }
}

```

5. en `angular-cli.json` :
    1. Reemplazar `"outDir": "dist"` con `"outDir": "dist/browser"`
    1. En el array `"apps"` agrega el objeto: 
        ```json
        // ...
        {
            "platform": "server",
            "root": "src",
            "outDir": "dist/server",
            "assets": [
                "assets",
                "favicon.ico"
            ],
            "index": "index.html",
            "main": "main.server.ts",
            "test": "test.ts",
            "tsconfig": "tsconfig.server.json",
            "testTsconfig": "tsconfig.spec.json",
            "prefix": "app",
            "styles": [
                "styles.css"
            ],
            "scripts": [],
            "environmentSource": "environments/environment.ts",
            "environments": {
                "dev": "environments/environment.ts",
                "prod": "environments/environment.prod.ts"
            }
        }
        // ...
        ``` 

6. En `package.json` reemplaza las propiedades existentes en el objeto `scripts` por: 
```json
// ...
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "lint": "ng lint --fix",
    "build:client-and-server-bundles": "ng build --prod && ng build --prod --app 1 --output-hashing=false",
    "build:ssr": "npm run build:client-and-server-bundles && npm run webpack:server",
    "webpack:server": "webpack --config webpack.server.config.js --progress --colors",
    "serve:ssr": "node dist/server"
  },
// ...
``` 

7. Crea el archivo `server.ts` en la carpeta raíz del proyecto:
```js
import 'zone.js/dist/zone-node';
import 'reflect-metadata';
import { enableProdMode } from '@angular/core';
// Express Engine
import { ngExpressEngine } from '@nguniversal/express-engine';
// Import module map for lazy loading
import { provideModuleMap } from '@nguniversal/module-map-ngfactory-loader';

import * as express from 'express';
import { join } from 'path';

// Faster server renders w/ Prod mode (dev mode never needed)
enableProdMode();

// Express server
const app = express();

const PORT = process.env.PORT || 4000;
const DIST_FOLDER = join(process.cwd(), 'dist');

// * NOTE :: leave this as require() since this file is built Dynamically from webpack
const {AppServerModuleNgFactory, LAZY_MODULE_MAP} = require('./dist/server/main.bundle');

// Our Universal express-engine (found @ https://github.com/angular/universal/tree/master/modules/express-engine)
app.engine('html', ngExpressEngine({
  bootstrap: AppServerModuleNgFactory,
  providers: [
    provideModuleMap(LAZY_MODULE_MAP)
  ]
}));

app.set('view engine', 'html');
app.set('views', join(DIST_FOLDER, 'browser'));

/* - Example Express Rest API endpoints -
  app.get('/api/**', (req, res) => { });
*/

// Server static files from /browser
app.get('*.*', express.static(join(DIST_FOLDER, 'browser'), {
  maxAge: '1y'
}));

// ALl regular routes use the Universal engine
app.get('*', (req, res) => {
  res.render('index', { req });
});

// Start up the Node server
app.listen(PORT, () => {
  console.log(`Node Express server listening on http://localhost:${PORT}`);
});
```

8. Crea el archivo `webpack.server.config.js` en la carpeta raíz del proyecto:
```js
// Work around for https://github.com/angular/angular-cli/issues/7200

const path = require('path');
const webpack = require('webpack');

module.exports = {
  entry: {
    // This is our Express server for Dynamic universal
    server: './server.ts',
  },
  target: 'node',
  resolve: { extensions: ['.ts', '.js'] },
  // Make sure we include all node_modules etc
  externals: [/(node_modules|main\..*\.js)/,],
  output: {
    // Puts the output at the root of the dist folder
    path: path.join(__dirname, 'dist'),
    filename: '[name].js'
  },
  module: {
    rules: [
      { test: /\.ts$/, loader: 'ts-loader' }
    ]
  },
  plugins: [
    new webpack.ContextReplacementPlugin(
      // fixes WARNING Critical dependency: the request of a dependency is an expression
      /(.+)?angular(\\|\/)core(.+)?/,
      path.join(__dirname, 'src'), // location of your src
      {} // a map of your routes
    ),
    new webpack.ContextReplacementPlugin(
      // fixes WARNING Critical dependency: the request of a dependency is an expression
      /(.+)?express(\\|\/)(.+)?/,
      path.join(__dirname, 'src'),
      {}
    )
  ]
}
```

9. Compila el proyecto:
> `npm run build:ssr`

10. Ejecuta el servidor: 
> `node dist/server`