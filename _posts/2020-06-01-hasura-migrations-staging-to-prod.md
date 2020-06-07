---
layout: post
title: Migrating changes from Staging to Prod using Hasura CLI
comments: true
---

I've been tinkering with Hasura for some time now. One of the issues I have is migrating changes from one env to another. It involves 4 steps:

- Create migrations
- Export metadata
- Apply migrations
- Apply metadata

I use this handy script to take all my changes from my staging env to prod env:

<script src="https://gist.github.com/ayushgp/fa9c0c7f6f2d45a26b25bba4df32b07b.js"></script>

Instead of passing the params every time I prefer to add them to my `~/.zshrc` file:

```bash
STAGING_ENDPOINT="https://staging.example.com/"
STAGING_SECRET="secret"
PROD_ENDPOINT="https://www.example.com/"
PROD_SECRET="prod-secret"
```

And use the following command to migrate the changes:

```bash
./migrate.sh $STAGING_ENDPOINT $STAGING_SECRET $PROD_ENDPOINT $PROD_SECRET "<migration-name>"
```

This can further be improved using [hasura squash](https://hasura.io/docs/1.0/graphql/manual/hasura-cli/hasura_migrate_squash.html) but works good enough for me now.
