# TranslationOS Sanity Plugin

A **Sanity Studio v3** plugin for translating Sanity documents using the TranslationOS API.

Please note that this plugin requires
the [Sanity document-internationalization plugin](https://www.sanity.io/plugins/document-internationalization).

## Installation

Add the plugin as a dependency in your Sanity Studio project by running:

```sh
npm install sanity-plugin-tos
```

## Configuration

Add it as a plugin in your `sanity.config.ts` (or `.js`).

The following configuration example shows how to add the plugin and share the list of languages and the active document types with
the document internationalization plugin. It is recommended that the language identifiers are taken from
the [list of TOS supported languages](https://api.translated.com/v2/symbol/languages).
Use of non-supported language identifiers is possible but must be agreed with Translated in advance.

```typescript
import {defineConfig} from 'sanity'
import {StructureBuilder} from "sanity/structure";
import {tosPlugin} from 'sanity-plugin-tos'

const languages = [
  {id: 'en-US', title: 'English'},
  {id: 'fr-FR', title: 'French'},
  {id: 'de-DE', title: 'German'},
  {id: 'es-ES', title: 'Spanish'},
];
const documentTypes = ['post', 'author']

export default defineConfig({
  //... other configuration
  plugins: [
    // ... other plugins
    documentInternationalization({
      supportedLanguages: languages,
      schemaTypes: documentTypes,
      weakReferences: true,
    }),
    structureTool({
      defaultDocumentNode: (S: StructureBuilder, {schemaType}: DefaultDocumentNodeContext) => {
        if (documentTypes.includes(schemaType)) {
          return S.document().views([
            S.view.form(),
            tosPlugin(S, {
              env: 'staging',
              apiKey: 'TOS_API_KEY',
              supportedLanguages: languages,
              customBlockTypes: ['myblock'],
            })
          ])
        }
      }
    }),
    //... other plugins
  ],
})
```

The `tosPlugin` function returns a `ComponentViewBuilder` that should be added to the _editor views_ for specific document types.
It is of course **highly recommended** to add the view only to the document types that have been configured in the document
internationalization plugin.
The function takes the following options:

- `env` - the environment of the TOS API, either `staging`, `sandbox` or `production`
- `apiKey` - your API key for the above TOS API environment
- `supportedLanguages` - the list of languages to be supported by the TOS plugin, the array has the same structure of the one used
  in the document internationalization plugin.
- `customBlockTypes` - an optional array of type names that should be treated by the plugin like the standard Sanity `block` type (see
  below for details).

## TOS plugin options in Sanity Studio's schema

TOS plugin specific options can be set in the schema definition of a field and are grouped in a `tosProperties` block.
The following options are currently available:

- `exclude` - a boolean value, set it to `true` to exclude the field from being sent to the TOS API for translation.

```typescript
// ... other field definitions
defineField({
  name: 'fullname',
  type: 'string',
  title: 'Full Name',
  options: {
    tosProperties: {
      exclude: true,
    }
  }
})
// ... other field definitions
```

Note that the `tosProperties` block can be added to any field type, including `object`. By adding it to an `object` field, all the
fields inside the object will be excluded from translation.

## What is translated by default

The plugin will send to translation all document fields of type `string`, `text`, `slug` or `block`.

Fields of type `array` and `object` will be recursively traversed.

The plugin will also exclude the fields excluded via a `tosProperties` block with `exclude` set to true as indicated in the
previous section.

While inspecting the document JSON structure, the plugin will also exclude the Sanity metadata fields (such as `_id`, `_rev`,
`_type`, `createdAt`, etc.).

### Block content

The `block` field type is used in Sanity to represent Rich Text content. A rich text field is defined as an array of `block`
objects. Each object uses a [JSON representation](https://www.sanity.io/docs/block-type) to express the formatting and styling of
the text.

To offer translators a better context, the plugin will send the rich text fields content transforming the array of block objects
into HTML where each object generally corresponds to a structural unit (paragraph, heading, list item, etc.).

The block type is highly customizable and can be redefined in the project schema in multiple ways. As we cannot support all the
possible combinations here are the supported block features:

- bold, italic, underline, strikethrough and code spans
- headings and blockquote styles
- bullet and numbered lists
- references to images
- links with generic metadata (see below for details)

### Custom block types

If you redefine the rich text type in your schema, you'll have to specify the custom block type names in the plugin configuration.
More specifically if you define a `name` for the block array elements, you should add those names to the `customBlockTypes` option.

For example, if you have the following rich text type definition:

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

Fields of type `myrichtext` will have the following structure in JSON:

```javascript
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

You should then configure the plugin to correctly handle fields of type `myrichtext` with their `myblock` elements as follows:

```typescript
tosPlugin(S, {
  // ...
  customBlockTypes: ['myblock'],
})
```

### Links

The plugin supports links in the rich text content with arbitrary metadata. The metadata is not sent to translation but preserved.

Here follows a sample of supported link definition:

```typescript
export default defineType({
  name: 'blockContent',
  title: 'Block Content',
  type: 'array',
  of: [
    defineArrayMember({
      title: 'Block',
      type: 'block',
      // ...
      marks: {
        decorators: [
          // ...
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