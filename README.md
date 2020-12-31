# React Server Components in Next.js

Experimental app of React Server Components with Next.js, based on [React Server Components Demo](https://github.com/reactjs/server-components-demo). **It's not ready for adoption. Use this in your projects at your own risk.**

## Development

### Prepare

You need these environment variables to run this app (you can create a `.env` file):

```
REDIS_URL='rediss://:<password>@<url>:<port>' // or `redis://` if no TLS
ENDPOINT='http://localhost:3000'              // need to be absolute url to run in prod/local
NEXT_PUBLIC_ENDPOINT='http://localhost:3000'  // same as above
SESSION_KEY='<random key for cookie-based session>'
OAUTH_CLIENT_KEY='github oauth app id'
OAUTH_CLIENT_SECRET='github oauth app secret'
```

### Start

1. `yarn install` (this will trigger the postinstall command)
2. `yarn dev`

Go to `localhost:3000` to view the application.

### Deploy

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/git/external?repository-url=https%3A%2F%2Fgithub.com%2Fvercel%2Fnext-server-components&env=REDIS_URL,ENDPOINT,NEXT_PUBLIC_ENDPOINT,SESSION_KEY,OAUTH_CLIENT_KEY,OAUTH_CLIENT_SECRET&project-name=next-server-components&repo-name=next-server-components&demo-title=React%20Server%20Components%20(Experimental%20Demo)&demo-description=Experimental%20demo%20of%20React%20Server%20Components%20with%20Next.js.%20&demo-url=https%3A%2F%2Fnext-server-components.vercel.app&demo-image=https%3A%2F%2Fnext-server-components.vercel.app%2Fog.png)

## Caveats

- Only `.js` extension is supported.
- Client / server components should be under the `components` directory.
- Some React Hooks are not supported in server components, such as `useContext`.
- You have to manually import `React` in your server components.

## How does it work?

Application APIs:

- `GET, POST /api/notes` (Get all notes, Create a new note)
- `GET, PUT, DELETE /api/notes/<note_id>` (Action for a specific note)

React Server Components API (`pages/api/index.js`):

- `GET /api` (render application and return the serialized components)

Note: Some of the application APIs (`POST`, `PUT`, `DELETE`) will render and return the serialized components as well. The render logic is handled by `libs/send-res.js`.

`libs/send-res.js` accepts the props (from `req.query.location` and `req.session.login`) that needs to be rendered by `components/App.server.js` (the component tree entry). Then, it renders the tree and streams it to `res` using:

```js
pipeToNodeWritable(React.createElement(App, props), res, moduleMap)
```

`moduleMap` is generated by client-side Webpack (through Next.js). It traverses both `.server.js` and `.client.js` and generates the full module map from the `react-server-dom-webpack/plugin` Webpack plugin (see `next.config.js`).
Then, we use a custom plugin to copy it to `libs/react-client-manifest.json` and include it from the lambdas (see `libs/send-res-with-module-map.js`).

`App` is a special build of `components/App.server.js`, which removes all the client components (marked as special modules) because they're not accessible from the server. We bundled it together with `libs/send-res.js` with another Webpack loader into `libs/send-res.build.js`. The Webpack script and loader are under `scripts/`. It should run whenever a server component is updated.

Finally, everything related to OAuth is inside `pages/api/auth.js`. It's a cookie-based session using GitHub for login.