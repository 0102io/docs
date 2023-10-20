# Dev Page Documentation

## Installation Requirements

- install nodejs
- instal bun ( https://bun.sh )

then in terminal, run:

```
bun install
```

## Running

Open two terminals, in the tapmeta directory and run:

Terminal 1:
```
cd packages/tapcanvas
bun run dev
```

Terminal 2:
```
cd packages/tapserver
bun run dev
```

## Building

```
cd packages/tapcanvas
bun run build
```

## Running Build

```
cd packages/tapcanvas
bun run build && bunx serve -n .
http://localhost:3000/app-dev/
```

## Publishing

```
cd packages/tapcanvas
bun publish.js
```

## Running chrome:

```
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --app="https://localhost:8581/app-dev/"
```