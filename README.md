<img src="https://inexorabletash.github.io/font-table-access/logo-font-table-access.svg" height=100 align=right>

# Font Table Access Explained

> August 14th, 2018<br>
> Last Update: July 8, 2019
>
> Alex Russell <code>&lt;slightlyoff@google.com&gt;</code><br>
> Emil A Eklund <code>&lt;eae@google.com&gt;</code><br>
> Josh Bell <code>&lt;jsbell@google.com&gt;</code><br>

## What’s all this then?

Professional-quality design and graphics tools have historically been difficult to deliver on the web.

One stumbling block has been an inability to access and use the full variety of professionally constructed and hinted fonts which designers have locally installed. The web's answer to this situation has been the introduction of [Web Fonts](https://developer.mozilla.org/en-US/docs/Learn/CSS/Styling_text/Web_fonts) which are loaded dynamically by browsers and are subsequently available to use via CSS. This level of flexibility enables some publishing use-cases but fails to fully enable high-fidelity, platform independent vector-based design tools for several reasons:

 * System font engines (and browser stacks) may handle the parsing and display of certain glyphs differently. These differences are necessary, in general, to create fidelity with the underlying OS (so web content doesn't "look wrong"). These differences reduce fidelity.
 * Developers may have legacy font stacks for their applications which they are bringing to the web. To use these engines, they usually require direct access to font data; something Web Fonts do not provide.

We propose two cooperating APIs to help address this gap:

 * A [font-enumeration API](https://github.com/inexorabletash/font-enumeration) which may, optionally, allow users to grant access to the full set of available system fonts in addition to network fonts
 * A font-table-access API which provides low-level (byte-oriented) access to the various [TrueType/OpenType](https://docs.microsoft.com/en-us/typography/opentype/spec/otff#font-tables) tables of local

Taken together, these APIs provide high-end tools access to the same underlying data tables that browser layout and rasterization engines use for drawing text. Such as the [glyf](https://docs.microsoft.com/en-us/typography/opentype/spec/glyf) table for glyph vector data, the GPOS table for glyph placement, and the GSUB table for ligatures and other glyph substitution. This information is necessary for these tools in order to guarantee both platform-independence of the resulting output (by embedding vector descriptions rather than codepoints) and to enable font-based art (treating fonts as the basis for manipulated shapes).

This document focuses on the latter API - **a font-table-access API**.

> NOTE: Long term, we expect that this proposal would merge into an existing CSS-related spec rather than stand on its own.

### Goals

A successful API should:

 * Enable access to all [browser-allowed font tables](https://chromium.googlesource.com/external/ots/+/master/docs/DesignDoc.md) (may vary per browser)
 * Re-use Web Font types and interfaces to the greatest extent possible
 * Restrict access to local font data to Secure Contexts
 * Be availabile from Workers
 * Enable a memory efficient implementation, avoiding leaks and copies by design

#### Possible/Future Goals

 * Access to font tables for web (network-loaded) fonts

### Non-goals

These APIs will not try to:

 * Fully describe how font loading works within the web platform. Fonts are a complex topic and Web Font loading implicates aspects of layout and style recalculation which are not at this time pluggable. As this design isn't addressing those aspects, we will not describe font application or CSS recalculation semantics.
 * Describe or provide full access to an existing WOFF/TTF/PS parser.
 * Provide access to the underlying WOFF/TTF/PS font files or describe their locations on disk.
 * Provide a guarantee that the set of available tables or their content matches the font on disk byte to byte.
 * Normalize differences in processed font tables across browser implementations. The font tables that will be exposed will have been processed by browser-provided parsers, but we will not describe or constrain them. For instance, if a library like [OTS](https://chromium.googlesource.com/external/ots/+/master/docs/DesignDoc.md) reduces the available information for a font, this spec will not require implementations to do more than they already would or provide alternative ways of getting such information back from the source font files.

## Key scenarios

> Note: Earlier versions of this document attempted to sketch out two versions of each API; one based on `FontFaceSource` and the other the fully-asynchronous verison that survives in this doc. While attractive from a re-use perspective, [`FontFaceSource`](https://drafts.csswg.org/css-font-loading/#font-face-source) (and the implied global `window.fonts`) implies synchronous iteration over a potentially unbounded (and perhaps slow) set of files, and each item may require synchronous IPCs and I/O. This, combined with the lack of implementations of `FontFaceSource` caused us to abandon this approach.

### Accessing Font Tables

Here we use enumeration and new APIs on `FontFace` to access specific OpenType tables of local fonts; we can use this to parse out specific data or feed it into, e.g., WASM version of [HarfBuzz](https://www.freedesktop.org/wiki/Software/HarfBuzz/) or [Freetype](https://www.freetype.org/):

```js
(async () => { // Async block
  // May prompt the user
  let status = await navigator.permissions.request({ name: "local-fonts" });
  if (status.state != "granted") {
    throw new Error("Cannot continue to style with local fonts");
  }
  for await (const f of navigator.fonts.query() {
    // Looking for a specific font:
    if (f.family !== "Consolas")
      continue;

    // 'getTables()' returns ArrayBuffers of table data. The default is
    // to return all available tables. See:
    //    https://docs.microsoft.com/en-us/typography/opentype/spec/
    // Here we ask for a subset of the tables:
    const tables = await f.getTables([ "glyf", "cmap", "head" ]);

    // 'tables' is a Map of table names to ArrayBuffers
    const head = new DataView(tables.get("head"));

    // Parse out the version number of our font:
    //    https://docs.microsoft.com/en-us/typography/opentype/spec/head
    const major = head.getInt16(0);
    const minor = head.getInt16(2);
    console.log("Consolas version:", (major + (minor/10)));
  }
})();
```

## Detailed design discussion

Several aspects of this design need validation:

* `FontFace` objects provide a lot of metadata synchronously, by default. Is this a problem?
* The arguments to `getTables()` could easily be done a dozen different ways. Feedback appreciated.
* This design tries to address concerns with `FontFaceSet` and friends at the cost of introducing a new API surface.
* It isn't strictly clear that providing table-by-table access is the best choice.
* The specifics of table filtering (i.e. what tables/subtables are included) is left up to user agents.


Other issues that feedback is needed on:

* Enumeration order of the returned table map needs to be defined.


### Privacy and Security Considerations

The `local-fonts` permission appears to provide a highly fingerprintable surface. However, UAs are free to return anything they like.

For example, the Tor Browser or Brave may choose to only provide a set of default fonts built into the browser. Similarly, UAs are not required to provide table data exactly as it appears on disk. Chrome, e.g., will only provide access to table data after sanitization via [OTS](https://github.com/khaledhosny/ots) and will fail to reflect certain tables entirely.

## Considered alternatives

### Raw Font File Access

A previous design for [Local Font Access](https://github.com/DHNishi/LocalFontAccess/blob/master/explainer.md) was designed to explicitly provide access to the underlying bytes of the font file without parsing or metadata extraction. This is arguably a better layering, in that it's possible to build all of the proposed methods for extracting metadata on top of such a system, however it received [significant push-back from experts in the domain](https://groups.google.com/a/chromium.org/forum/#!msg/blink-dev/G-hC66MRTso/uVrmHV0NAwAJ).

### Glyph Vector Data Access

Processing of font table data eventually ends up with a vectorized description of the glyph to be painted. This vectorized representation is the goal of professional design software in getting access to low-level font descriptions, however the engines that turn font tables into glyphs do not do so consistently across browsers and OSes.

Professional design packages therefore contain their own table-to-glyph conversion code (frequently by embedding a copy of [FreeType](https://www.freetype.org/)) to guarantee consistency. While it may be a valuable next step to also deliver this higher-level information for developers who want it, we have not had requests yet for such an API, whereas we have engaged and excited users asking for font table access.

### FreeType/HarfBuzz APIs (Roughly)

Chromium embeds FreeType and Harfbuzz to handle font parsing, font metrics computation, line-breaking, HTML/CSS and OpenType layout, text shaping and many other font-related tasks. Why not simply expose APIs that look very similar to theirs? A few reasons. First, these low-level systems may not be what other vendors implement, making their standardisation premature or inappropriate (depending). Secondly, these interfaces may change. Better in that case to design the abstractions we need, allowing our implementations to change without undue stress. Lastly, the calling conventions of these APIs is not compatible with JavaScript's cooperative multi-tasking, meaning we'd need to re-design wrappers for them anyhow.

## References & acknowledgements

The following references have been invaluable:

* [MSDN DirectWrite overview](https://docs.microsoft.com/en-us/windows/desktop/directwrite/introducing-directwrite#accessing-the-font-system)
* [OpenType Specification](https://docs.microsoft.com/en-us/typography/opentype/spec/)
* [OpenType Font Table overview](https://docs.microsoft.com/en-us/typography/opentype/spec/chapter2)

We'd like to acknowledge the contributions of:

* Daniel Nishi, Owen Campbell-Moore, and Mike Tsao who helped pioneer the previous local font access proposal
* Evan Wallace, Biru, Leah Cassidy, Katie Gregorio, Morgan Kennedy, and Noah Levin of Figma who have patiently enumerated the needs of their ambitious web product.
* Tab Atkins and the CSS Working Group who have provided usable base-classes which only need slight extension to enable these cases
* Dominik Röttsches and Igor Kopylov for their thoughtful feedback
