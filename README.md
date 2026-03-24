# example-flights
A Graphene project analyzing FAA flights data from 2000-2005.

## Setup

1. `git clone https://github.com/graphene-data/example-flights.git`
2. `cd example-flights && npm install`

## Usage

1. Run `npx graphene serve` to start the dev server
2. Go to **localhost:4000** in your browser to view the example dashboard provided here.
3. Ask your coding agent: "Come up with an interesting question about this data, and then build a dashboard answering it." The dashboard will be viewable at **localhost:4000/dashboard-filename**.

_When is the best time of day for a flight to depart?_
<img width="2618" height="1701" alt="screen" src="https://github.com/user-attachments/assets/f2448f82-6a19-463f-939d-cac91753cdb0" />

## About

Graphene is primarily comprised of three things:
1. Dashboard `.md` files for visualizing data.
2. A semantic modeling and querying language, GSQL. There are dedicated `.gsql` files for semantic models, whereas GSQL `select` statements are  embedded inside dashboard files.
3. A CLI for checking syntax, running queries, starting up the dev server, and more.

The graphene-cli npm package ships with an agent skill that documents everything an agent needs to know about Graphene. You can read it in the .claude or .agents folders (after cloning this repo and installing the CLI).

## Connecting Graphene to other data

Ask kevin@graphenedata.com for setup instructions to connect Graphene to local data files or a remote SQL database.
