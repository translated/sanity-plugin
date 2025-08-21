# TranslationOS plugin for Sanity Studio v3

A plugin for **Sanity Studio v3** that integrates with the TranslationOS API to translate Sanity documents.

This plugin requires either the [Sanity document internationalization plugin](https://www.sanity.io/plugins/document-internationalization) or the [Sanity field-level internationalization plugin](https://github.com/sanity-io/sanity-plugin-internationalized-array).

## Features

The TranslationOS plugin for Sanity offers two independent translation modes:

- **Single document translation**: Sends a single document to the TranslationOS API for translation. Supports both document-level and field-level localization.
- **Bulk document translation**: Sends multiple documents to the TranslationOS API for translation. Supports **document-level** localization only.

Single document translation is available within the [Structure tool](https://www.sanity.io/docs/studio/structure-tool). Bulk document translation is available as a top-level tool.

## Installation

Add the plugin to your Sanity Studio project as a dependency by running:

```sh
npm install sanity-plugin-tos
```

## Configuration

Add the plugin to your `sanity.config.ts` (or `.js`).

Below is an example configuration using both modes, sharing languages and document types with the internationalization plugins, and defining supported field types.

We recommended using [TranslationOS-supported language codes](https://api.translated.com/v2/symbol/languages), but we can unofficially support other language codes by configuring a custom mapping.

```typescript
import {defineConfig} from 'sanity'
import {StructureBuilder} from "sanity/structure";
import {tosPlugin} from 'sanity-plugin-tos'
import {internationalizedArray} from 'sanity-plugin-internationalized-array'

// Languages shared across plugins
const languages = [
    {id: 'en-US', title: 'English (US)'},
    {id: 'fr-FR', title: 'French (France)'},
    {id: 'de-DE', title: 'German (Germany)'},
    {id: 'es-ES', title: 'Spanish (Spain)'},
];

// Document types handled by each mode
const documentLevelTypes = ['post', 'author']
const fieldLevelTypes = ['blog']

// Common TranslationOS configuration
const tosCommonConfig = {
    apiKey: 'YOUR_TOS_API_KEY',
    env: 'staging',
    supportedLanguages: languages,
    customBlockTypes: ['myblock'],
}

export default defineConfig({
    // [...]
    plugins: [
        // [...]
        documentInternationalization({
            supportedLanguages: languages,
            schemaTypes: documentLevelTypes,
            weakReferences: true,
        }),
        structureTool({
            defaultDocumentNode: (S: StructureBuilder, {schemaType}: DefaultDocumentNodeContext) => {
                if ([...documentLevelTypes, ...fieldLevelTypes].includes(schemaType)) {
                    return S.document().views([
                        S.view.form(),
                        // Structure tool plugin view for single document translation
                        tosPlugin(S, {
                            ...tosCommonConfig,
                            // Document types supported in document-level translation mode
                            documentLocalizationSchemaTypes: documentLevelTypes,
                            // Document types supported in field-level translation mode
                            fieldLocalizationSchemaTypes: fieldLevelTypes,
                        })
                    ])
                }
            }
        }),
        internationalizedArray({
            languages,
            fieldTypes: ['string', 'text', 'block'],
        }),
    ],

    // Top-level tool for bulk document translation in document-level translation mode
    tools: [
        translationOS({
            ...tosCommonConfig,
            schemaTypes: documentLevelTypes,
        })
    ],
})
```

## Plugin options
### `tosPlugin` (single document translation view)

Returns a `ComponentViewBuilder` to be added to the _editor views_ for specific document types.

**Recommended**: only enable for schema types configured in your internationalization plugin.

Options:
- `env` – TranslationOS API environment (`staging`, `sandbox` or `production`)
- `apiKey` – TranslationOS API key for the above environment
- `supportedLanguages` – array of `{id, title}` language objects
- `documentLocalizationSchemaTypes` – document types for document-level translation
- `fieldLocalizationSchemaTypes` – document types for field-level translation
- `customBlockTypes` – additional block type names to treat like Sanity's `block`

> **Note:** A schema type can't be included in both the `documentLocalizationSchemaTypes` and the `fieldLocalizationSchemaTypes` options,
otherwise the plugin will display a configuration error and won't work properly.

### `translationOS` (bulk document translation tool)

**Recommended**: only enable for schema types configured in the document internationalization plugin.

Options:
- `env` – TranslationOS API environment (`staging`, `sandbox` or `production`)
- `apiKey` – TranslationOS API key for the above environment
- `supportedLanguages` – array of `{id, title}` language objects
- `customBlockTypes` – additional block type names to treat like Sanity's `block`
- `schemaTypes` – document types to manage with bulk translation

## Known limitations

For field translation mode:
- Supports common types: `string`, `text`, `block`
- Supports nested `object` fields
- Does **not** traverse `array` fields to locate nested translatable fields

## Field-level options in schema

You can add `tosProperties` to any field to control translation behavior.

```typescript
defineField({
  name: 'fullname',
  type: 'string',
  title: 'Full Name',
  options: {
    tosProperties: {
      exclude: true, // exclude from translation
    }
  }
})
```

Adding `tosProperties.exclude: true` to an `object` field excludes all nested fields.

## Default behavior in document-level mode

By default, the plugin translates:
- All fields of type `string`, `text`, `slug`, `block`
- Nested fields within `array` or `object` values (recursively)
- Excludes Sanity metadata (`_id`, `_rev`, `_type`, `createdAt`, etc.)
- Excludes fields with `tosProperties.exclude: true`

## Rich text (block content) support

Sanity's `block` type represents rich text as an array of [objects](https://www.sanity.io/docs/block-type). The plugin converts these to HTML before sending to TranslationOS, preserving structure and context. Supported features include:
- Bold, italic, underline, strikethrough, code spans
- Headings, blockquotes
- Bullet and numbered lists
- Image references
- Links with metadata (metadata preserved, not translated)

## Custom block types

If you define custom block names in your schema, list them in `customBlockTypes` so the plugin can process them correctly.

Let's say you have the following rich text type definition:

```typescript
export default defineType({
  name: 'myrichtext',
  title: 'My Rich Text Content',
  type: 'array',
  of: [
    defineArrayMember({
      name: 'myblock', // <--- custom block type name
      title: 'My Block',
      type: 'block',
      styles: [
        {title: 'Normal', value: 'normal'},
        {title: 'H1', value: 'h1'},
      ],
      marks: {
        decorators: [
          {title: 'Code', value: 'code'}
        ],
      },
    }),
  ],
})
```

Fields of type `myrichtext` would have the following JSON structure:

```json5
{
  "_createdAt": "2025-01-16T13:48:35Z",
  "_id": "drafts.cc28b1ec-8340-41ae-a820-3cd3e4f6038c",
  "_rev": "b464c3da-548a-4402-a2a0-be29fb61120e",
  "_type": "post",
  "_updatedAt": "2025-01-17T13:44:05Z",
  "body": [
    {
      "_key": "aba42a503204",
      "_type": "myblock", // <--- custom block type name
      "children": [
        {
          "_key": "e8799f8c0f9e",
          "_type": "span",
          "marks": [],
          "text": "The Post Body"
        }
      ],
      "markDefs": [],
      "style": "h2"
    }
  ]
}
```

You must configure the plugin to correctly handle fields of type `myrichtext` with `myblock` elements as follows:

```typescript
tosPlugin(S, {
  // [...]
  customBlockTypes: ['myblock'],
})
```

## Link handling

The plugin supports links with arbitrary metadata inside rich text. Metadata is preserved but not translated.

Example:

```typescript
export default defineType({
  name: 'blockContent',
  title: 'Block Content',
  type: 'array',
  of: [
    defineArrayMember({
      title: 'Block',
      type: 'block',
      // [...]
      marks: {
        decorators: [
          // [...]
        ],
        annotations: [
          {
            name: 'link',
            title: 'Complex Link',
            type: 'object',
            fields: [
              {
                name: 'title',
                title: 'Title',
                type: 'string',
              },
              {
                name: 'external',
                title: 'External Link',
                type: 'url',
              },
              {
                name: 'author',
                title: 'Author',
                type: 'reference',
                to: [{type: 'author'}]
              },
              {
                name: 'newTab',
                title: 'Open in new tab',
                type: 'boolean',
                initialValue: false,
                description: 'Set to true to open the link in a new tab.',
              },
            ],
          },
        ],
      },
    }),
  ],
})
```
