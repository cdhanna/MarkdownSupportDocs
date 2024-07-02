# Markdown Support

_Markdown Support_ adds capabilities for the `.md` and `.markdown` file extensions inside
Unity. Markdown is a text formatting standard that is easy to read in plain text,
and can be rendered into HTML for web clients. _Markdown Support_ converts plain text
markdown files into Unity UIToolkit panels that are viewable in both Editor and Runtime. 

---
## Markdown Files

In Unity, open the Create menu to make a _"Markdown File"_. This will create a 
`.markdown` file in your project. The Unity Inspector window will render the 
markdown. You can also use the `.md` file extension, but Unity will recognize it 
as a `TextAsset`, so the icon of the file will be the standard `TextAsset` icon.

---
## Markdown SDK

_Markdown Support_ has a small SDK that you can use to integrate Markdown into
your workflow. For a demonstration, see the two sample projects, `Markdown/Samples/RuntimeDemo` and
`Markdown/Samples/EditorDemo`. 

The SDK is inside the `BrewedInk.MarkdownSupport` assembly, which is not set to be
automatically referenced. If you need access to the Editor specifics, use the `BrewedInk.MarkdownSupport.Editor` assembly. This means that you will need to create your own assembly
definition and add a reference to `BrewedInk.MarkdownSupport` to use the SDK. 
Alternatively, you can change the `autoReferenced` property to `true` in the 
_Markdown Support_ assembly definition. 

### `UMarkdown.Parse`

The entrypoint to the SDK is the `UMarkdown.Parse` function. 

```csharp

// somehow, get some markdown text
string markdownText = @"
# Example
*hello* __world__, its `nice` to meet you.
";

// then convert it into a `MarkdownVisualElement`
MarkdownVisualElement markdownElement = UMarkdown.Parse(
    markdown: markdownText, 
    context: UMarkdownContext.GetDefault(false));

```

The `Parse()` method takes 2 parameters, a `string` of markdown text, and a `UMarkdownContext`, 
which is discussed in the [configuration section](#configuration).

The output of the method is a `MarkdownVisualElement`, which can be inserted into whatever 
document you like. The `Parse()` method **will not** create a `ScrollView` automatically, so if
your markdown is long, you need to make sure a `ScrollView` exists in the document where the
`MarkdownVisualElement` is inserted. In most _Editor_ use cases, Unity itself will inject a 
`ScrollView` into the parent panel. However, in _Runtime_, please don't forget to handle the
`ScrollView`.

### `MarkdownVisualElement`

The `MarkdownVisualElement` is a sub class of `VisualElement` (the base type of most UIToolkit
components). A small SDK is available beyond the standard UIToolkit methods. 

#### `MarkdownVisualElement.ScrollTo`

The `ScrollTo` method will find the `ScrollView` in the parent lineage of the element, and focus
the scroll position around the desired markdown element.

```csharp
string headingAnchor = "#example";
markdownElement.ScrollTo(headingAnchor);
```

The method takes a single input, either a `string` or a `VisualElement`. 

If a `string` is given,
it must start with a `"#"` symbol, and be a valid heading anchor link. Heading anchors are lowercased,
hyphenated versions of the headings in the markdown. For example, here is a table of example headings
and their corresponding anchors.

| markdown        | anchor         |
|-----------------|----------------|
| `# Example`     | `#example`     |
| `# Hello World` | `#hello-world` |
| `# Tuna Truck`  | `#tuna-truck`  |

By default, only headings have anchor tags. However, _Markdown Support_ includes the ability to specify
custom element ids, classes, and attributes. Check the [docs](https://github.com/xoofx/markdig/blob/master/src/Markdig.Tests/Specs/GenericAttributesSpecs.md)
for more information. Below is an example of how to attach an id, `#example` to paragraph in markdown.

```markdown
# hello
This paragraph has a custom id, and can be selected. {#example}
```

If a `VisualElement` is given the `ScrollTo()` function, then the markdown document will scroll so that 
the given `VisualElement` is within view. The element must be a child of the markdown document. 


#### `MarkdownVisualElement.QAttribute`

It is possible to add custom attributes to markdown elements, but UIToolkit does not have a way to let
you interact with those attributes by default. The following markdown sample will add an attribute, "tuna"
with a value of "stinky". 

```markdown
this paragraph has an attribute, "tuna", and another, "fish" set to "stinky". {tuna=stinky}
```

Then, the element can be identified via the `QAttribute` function.
```csharp
var elem = content.QAttribute("fish", "stinky");
```

This could be useful for running a procedural operation on the generated markdown.

**Warning**, this method uses Reflection to access the attributes on the element, because Unity does
not publically expose them. Reflection may be acceptable in your use case, but you should never call
Reflection based code in a hot loop, or ideally, even in an Update loop. Also, with any Reflection based
code, support for attribute filtering may break with future releases of Unity.

---
## Configuration Context

Everytime _Markdown Support_ renders a markdown file, it needs to contextualize the rendering with
some extra data. There are 4 pieces of data, the configuration context, the `isRuntime` field, the the `rootFilePath` field,
and the `textureLoader` field.
These pieces of information are stored in the `UMarkdownContext`, which can be obtained in a standard
set of ways.

1. the context can be created manually, using constructors.
2. the context can be fetched using `UMarkdownContext.ForFile()`, which will create a context 
using the standard set of configuration, and `isRuntime` and `rootFilePath` must be specified.
2. the context can be fetched using `UMarkdownContext.Runtime()`, which will create a context using
the standard set of configuration, assume that `isRuntime` is `true`, and sets a blank string to the 
`rootFilePath`.


The properties of the context are defined in detail in the following sections.

### `rootFilePath`

Loading images in markdown can be done with URLs, or relative file paths. When images are loaded with 
relative file paths, then the markdown rendering must know where the file path is relative _to_. 

In the Editor, the default Inspectors use the `ForFile()` method and pass the file location of the markdown
file being rendered, which means the file paths are relative to the markdown file itself. The images are
then loaded with the `AssetDatabase`. 

However, in Runtime, `AssetDatabase` is not available, and in a built game, images do not retain their
source path information _anyway_. Images are loaded via `Resources`, which means that the filepath is
also relative to the `Resources` folder regardless of where the original markdown file was located. 

It is possible to override the image load behaviour, see the `textureLoader` property below.

### `isRuntime`

Images in Runtime are loaded with the `Resources` SDK, and images in Editor are loaded with the `AssetDatabase`. 
However, there is no obvious way for code in Unity to know if its intended to be executed in Editor, or in Runtime.
_Markdown Support_'s SDK can be used in either case, so the decision of which image loading technique to use
is left as a contextual requirement. 

### `textureLoader`

By default, images are loaded from the `AssetDatabase` when in the Editor, and from `Resources` folders 
when in Runtime. As mentioned, it would be too overbearing to generalize an image loading 
approach for _all_ possible use cases in the Runtime. Instead, the _Markdown Support_ library takes the approach
of a "less is more", and offers only `Resources` loading out of the box.

However, within the _Markdown Support_ library, all images are loaded through the `textureLoader` delegate.
The delegate must be of the `UMarkdownTextureLoader` type, which takes 3 parameters, 
1. the `UMarkdownContext` instance 
2. the `MarkdownLinkUri` instance that is being invoked
3. a callback `Action<Texture2D>` function 

The only requirement is that the given `Action<Texture2D>` callback is executed with a `Texture2D` instance. 
The `Markdown/Samples/RuntimeDemo/Files/Logic.cs` file has an example showing a custom loader function that
takes the link content and matches it against a set of `Texture2D` asset names attached to the game object.


### Configuration

_Markdown Support_ uses a configuration everytime plain text is rendered into markdown. Normally, 
a single global configuration is used, but it is possible to create and use customized configurations
for custom workflows. 

The Inspector for `.markdown` and `.md` files will always use the global configuration settings. 
To change the settings, you must create a `UMarkdownConfig` scriptable object in a `Resources` folder,
and the file must be named, `MarkdownSettings`, otherwise it will not be loaded. 

The configuration controls the visual theming of the rendered markdown, the root file path for any
relative file paths given in the markdown document, and a few behavioural items. 

#### `useCodeCopyButtons`
The `useCodeCopyButtons` option is set to `true` by default. When `true`, any code fence will appear 
with a "copy" button in the upper-right corner of the code block. While this is a nice tool for Editor
use cases, it _may_ not be helpful in a Runtime setting. 

#### `validTextAssetExtensions`
The `validTextAssetExtensions` option is an array of `string`, that specify which file extensions 
among the Unity Text Asset extensions should be rendered as markdown. From Unity's [documentation](https://docs.unity3d.com/Manual/class-TextAsset.html),
these are the Text Asset extensions in 2022. 
```
.txt
.html
.htm
.xml
.bytes
.json
.csv
.yaml
.fnt
.md
```

_Markdown Support_ provides a custom Inspector for Text Assets, but only renders files with extensions
included in the `validTextAssetExtensions` array as markdown. Otherwise, the file content is rendered
_similarly_ to how Unity normally renders the content. By default, the `validTextAssetExtensions` 
field only contains the `.md` extension, which is the conventional extension for markdown documents. 


#### `styleSheets`
_Markdown Support_ uses [Unity Style Sheets](https://docs.unity3d.com/2020.1/Documentation/Manual/UIE-USS.html)
to control _most_ of the rendering of the markdown document. By default, there is a single `.uss` file 
that handles all of the styling, `MarkdownStyle.uss`. However, this can be removed, or additional style 
sheets can be added to override specific portions of the document. 

This could be used to completely change the default appearance of markdown in your Unity workspace. 

#### `richTextStyleAsset`
Unfortunately, there are some things that _Markdown Support_ cannot style through `.uss`. These remaining
items can be controlled through the `richTextStyleAsset` field. There can be only one of these assets 
referenced. You can create a `RichTextStyleAsset` through the `Create/Markdown Support` menu. 

The sub properties of the rich text style asset are below.

##### `codeSelectionColor`
Code blocks often have a different background color than the rest of the markdown document, and as 
such, have a different text highlight color when the user selects code in the block. The `codeSelectionColor`
option allows you to control the color of the highlighted text in a code block. 

##### `codeMarkupColor`
_Markdown Support_ uses Rich Text to handle a lot of the emphasis in markdown. Specifically, then
you use the single  \`  character to emphasis text, it shows up `this way`. The `codeMarkupColor` 
will control the background color of `this text`. 

##### `codeFontAsset`
When code is rendered, Rich Text is used to inline a new font into the paragraph element.
Be aware that font assets must be placed in a "Resources/Font & Materials"folder (or whatever 
the given folder is in the TMP Settings).

## Limitations

Unfortunately, there are a few known limitations of the _Markdown Support_ tool. 

###### Limited Extended Markdown Support

Extended Markdown is full of wonderful extensions that this asset does not support. Below 
is a comptability table.

| Feature         | Supported |
|-----------------|-----------|
| Tables          | yes       |
| Alignment       | no        |
| Fenced Code     | yes       |
| Footnotes       | no        |
| Definition List | no        |
| Strike Through  | yes       |
| Task list       | no        |
| Emoji           | no        |
| Highlight       | no        |


###### Links do not support emphasis

The following markdown _should_ produce a link in italics, but _Markdown Support_ will only render the link,
without any italic text. 

```markdown
this _[link](https://brewed.ink)_ should have been italic, but it won't be in MarkdownSupport.
```

###### Unity Lightmode support

Support for Unity Lightmode is not ideal.

###### Text Selection

Unity UIToolkit's `Label` class is not selectable in Unity 2021. In 2022, it is selectable, but selection
cannot go between multiple elements, making the selection of all text in a markdown document impossible.

## Acknowledgements 

_Markdown Support_ relies on two excellent existing code libraries, and one font.

The font is _Roboto_, and is licensed under Apache 2.0.
- [Roboto License](ROBOTO_LICENSE.txt)

There are two code libraries...

- [Markdig](https://github.com/xoofx/markdig) is under BSD-2 license, https://github.com/xoofx/markdig/blob/master/license.txt.
- [Highlight](https://github.com/thomasjo/highlight/tree/master) is under MIT license, https://github.com/thomasjo/highlight/blob/master/LICENSE.md . I made a private fork of the project and built my own `.dll`.

Without these excellent libraries, _Markdown Support_ would not be possible in Unity.
