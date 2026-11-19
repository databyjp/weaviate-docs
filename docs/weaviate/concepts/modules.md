---
title: Modules
sidebar_position: 15
description: "Modular architecture overview for extending Weaviate functionality with specialized add-on components."
image: og/docs/concepts.jpg
# tags: ['modules']
---
<!-- :::caution Migrated From:
- Combines theoretical explanations from `Configuration/Modules` + `Modules/Index`. e.g.:
  - `Introduction` is from `Configuration/Modules`
  - `Vectorization modules (Dense Retriever modules)` is from `Modules/Index`
::: -->

Weaviate has a modularized structure. Functionality such as vectorization or backups is handled by *optional* modules.

The core of Weaviate, without any modules attached, is a pure vector-native database.
[![Weaviate modules introduction](./img/weaviate-module-diagram.svg "Weaviate Module Diagram")](./img/weaviate-module-diagram.svg)

Weaviate stores objects with optional vector representations. Configured vectors can be searched with a [vector index](../concepts/indexing/vector-index.md). Without any vectorizer modules attached, Weaviate does not know how to *vectorize* an object, i.e. *how* to calculate the vectors given an object.

Depending on the type of data you want to store and search (text, images, etc.), and depending on the use case (like search, question answering, etc., depending on language, classification, ML model, training set, etc.), you can choose and attach a vectorizer module that best fits your use case. Or, you can "bring your own" vectors to Weaviate.

This page explains what modules are, and what purpose they serve in Weaviate.

Module enablement at the instance level is separate from selecting a vectorizer for a collection or an integration for a supported query. In core/self-managed Weaviate, API-based integrations are enabled by default from v1.33, but still require applicable provider configuration and credentials. Local inference modules can require explicit enablement and additional services. Module availability does not automatically vectorize a collection: select its vectorizer explicitly. See [module configuration](../configuration/modules.md) for deployment-specific settings.


## Available module types

Modules include vectorizers, rerankers, readers, generators, and operational integrations such as backups. For maintained catalogs, see [model-provider integrations](../model-providers/index.md) and the [module reference](../modules/index.md).

### Vectorizer & Ranker modules

Vectorizer modules, like the `text2vec-*`, `multi2vec-*` or `img2vec-*` modules, transform data into vectors. Ranker modules, like the `reranker-*` (for example, `reranker-cohere`) modules, rank the results.

### Reader & Generator modules

Reader or Generator modules can be used on top of a Vectorizer module. These modules take the set of relevant documents that are retrieved, and performs another operation, such as question answering, or a generative task. An example Reader module is [`qna-transformers`](../modules/qna-transformers.md) module, which extracts an answer directly from a document. A Generator module would, on the other hand, use *language generation* to generate an answer from the given document.

### Other modules

These include those such as `backup-gcs` or `text-spellcheck`.

## Dependencies

Modules can depend on capabilities provided by other modules. For example, [`qna-transformers`](../modules/qna-transformers.md) requires a compatible text vectorization module providing `nearText`; configure the appropriate text vectorizer for the collection. See [module configuration](../configuration/modules.md) for multi-vectorizer constraints.

## Weaviate without modules

Weaviate can be used without optional modules. For vector similarity search without a vectorizer module, configure a vector representation and supply compatible vectors yourself. Objects can also be stored and queried without vectors, for example with keyword search and filters. See [vector configuration](../manage-collections/vector-config.mdx).

## Custom modules

It is possible for anyone to create a custom module for use with Weaviate. Click [here](../modules/custom-modules.md) to see how you can create and use your own modules.

## Further resources

:::info Related pages
- [Configuration: Modules](../configuration/modules.md)
- [References: Modules](../modules/index.md)
:::

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>
