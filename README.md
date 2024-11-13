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

The `block` field type is highly customizable and can be redefined in the project schema. The plugin supports the following block
features:

- bold, italic, underline, strikethrough, code and link spans
- headings and blockquote styles
- bullet and numbered lists
- references to images