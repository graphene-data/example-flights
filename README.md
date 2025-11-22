# example-flights
A Graphene project analyzing FAA flights data from 2000-2005.

## Setup

1. `git clone https://github.com/graphene-data/example-flights.git`
2. `cd example-flights && npm install`
3. `npm run setup`

## Usage

1. Run `npm run graphene serve` to start the dev server
2. Go to **localhost:4000** in your browser to view the example dashboard provided here.
3. Ask your coding agent: "Come up with an interesting question about this data, and then build a dashboard answering it." The dashboard will be viewable at **localhost:4000/\<markdown-file-path-without-extension\>**.

Graphene is primarily comprised of three things:
1. Dashboard `.md` files, for presenting and visualizing data.
2. A semantic modeling and querying language, GSQL. There are dedicated `.gsql` files for semantic models, whereas GSQL `select` statements are typically embedded inside dashboard files.
3. A CLI for checking syntax, running queries, starting up the dev server, and more.

Comprehensive documentation is available at `node_modules/@graphenedata/cli/dist/docs/graphene.md`.

## Connecting Graphene to other data

Ask kevin@graphenedata.com for setup instructions to connect Graphene to local data files or a remote SQL database.
