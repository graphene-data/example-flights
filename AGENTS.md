This is a Graphene project to analyze FAA flight data from 2000-2005.

# Resources

- Graphene documentation: @/node_modules/@graphenedata/cli/dist/docs/graphene.md. It is CRITICAL that you read this **in its entirety** (it's not too long). It is imperative that you understand the Graphene CLI, GSQL, and Graphene Markdown.
- Available data: /tables directory. The .gsql files inside document the schemas, joins, metrics, and usage instructions for each table.

# Best practices

## When formulating GSQL queries (in CLI, .gsql, or .md)
- Use stored expressions whenever possible
- Write optimized GSQL queries that filter and limit to only the data needed
- Include comments explaining complex logic
- If you encounter a syntax error, consult the Graphene documentation again before trying a different approach.
- Only the GSQL functions supported on DuckDB are available (see documentation).

## When writing to a .gsql file
- ALWAYS check your code with `npm run graphene check`.

## When writing to a Graphene .md file
- If you add or edit a code-fenced GSQL query, run it separately in the CLI first to verify the results.
- ALWAYS check your code with `npm run graphene check [mdPath]`. Run the command with full permissions because the screenshot may not work in a sandbox.
- If `check` is successful, it will save a screenshot. Look at the screenshot and critique what you see: 
  - Do all the numbers make sense?
  - Are all the data values and axes labels formatted in a way that is easy to read?
  - Does the shape of the visualized data require an adjustment to scale, axis min/max, axis split, etc.?
  - Are metrics colored consistently across visualizations?
  - Are any visualizations missing data altogether? 
  - Is that visualization type really the best way to illustrate the data?
  - Are any visualizations redundant? Can you say the same thing with less?
- Do not multiply percentages by 100. Graphene's visualizations do this automatically.
