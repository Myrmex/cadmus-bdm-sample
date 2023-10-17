# BDM Conversion

Companion material for paper B. Bessi & D. Fusi, _Modelling the Archipelago: Corfu as a Case Study for a Digital Edition of Cristoforo Buondelmonti’s Liber Insularum_.

>The Liber Insularum by Cristoforo Buondelmonti can be considered the first guide to the Greek islands, each of them described by a textual paragraph and illustrated by color maps, in a format which gave rise to the new literary genre of Isolaria.” Mapping the Aegean: Cristoforo Buondelmonti’s Liber Insularum” is a Marie Skłodowska-Curie project aimed at the study of this book. This paper illustrates the application of Cadmus, a structured content management tool, to the creation of a digital edition of the Liber and to do, we focus on the text and map of Corfu as a case study. After a historical introduction on the author and his work and the presentation of the project, we explain why we chose to use this tool and its main characteristics, and we offer a concrete example of its application to the material pertaining to the description of Corfu by showing its frontend output.

🔗 Related repositories:

- [Cadmus introduction](https://myrmex.github.io/overview/cadmus/): here you can find further information and links to Cadmus repositories, projects, online demos, and technical documentation. All the Cadmus repositories are open source and hosted in this VeDPH repository.
- [MapAeg Proteus-based import tool](https://github.com/vedph/cadmus-bdm-tool)
- [MapAeg Cadmus API](https://github.com/vedph/cadmus_bdm_api)
- [MapAeg Cadmus app](https://github.com/vedph/cadmus-bdm-app)

📁 This repository contains accompanying material for a paper:

- [bdm-cadmus.json](./bdm-cadmus.json): [Proteus]((https://myrmex.github.io/overview/proteus/)) import pipeline.
- [bdm-dump.json](./bdm-dump.json): Proteus diagnostic dump pipeline.
- [cadmus-bdm.agz](./cadmus-bdm.agz): the dump of the imported MongoDB database.
- [corfu.docx](./corfu.docx): the legacy input Word document.
- [corfu.xml](./corfu.xml) and [corfu_fmt.xml](./corfu_fmt.xml): the results of the extraction from DOCX into intermediate XML.
- [dump.xlsx](./dump.xlsx): the decoded entries dump produced by Proteus using the diagnostic dump pipeline.

## Document Description

The input DOCX documents have alternating language paragraphs. English paragraphs have footnotes, which in turn use some conventional escapes:

(e1) textual notes are marked with `&` to separate them from commentary notes, i.e.: the first character of a textual note is `&`.

(e2) tokens starting with "http" end with the first whitespace or comma or `)` or at paragraph end. This sequence represents a set of 1 or more URLs, each starting with "http": `http[^\s,)]+`. Their anchor text is the token before them: i.e. before http we expect a set of non-whitespaces + whitespace(s). That set is the reference word.

Example:

```txt
Kerkyra https://www.wikidata.org/wiki/Q121378http://www.geonames.org/2463678/corfu.htmlhttps://pleiades.stoa.org/places/530835https://topostext.org/placde/396199PKerhttps://manto.unh.edu/viewer.p/60/2616/object/6580-9587576, the ...
```

(e3) ancient references are prefixed with `@` and have forms:

- "@author, work, location" terminated by `;` or `)` (e.g. "@Ap. Rhod., Argon. 4.565–6"): `\@(?<a>[^,\d]+),\s*(?<w>[^;),]+),\s*(?<l>[^;)]+)`.
- "@author, location" terminated by `;` or `)` (e.g. "@Paus., 5.22.6"): `\@(?<a>[^,\d]+),\s*(?<l>[^;)]+)`.

(e4) modern references are prefixed with `@` and have form "@AUTHOR YEAR,LOC" where both YEAR and LOC are digits, but LOC can also be digits-digits for a range of pages: `\@(?<a>[^,\d]+)\s+(?<y>[12]\d{3})(,\s*(?<l>[^;)]+))?` (e.g. "@LEONTSINI 2014, esp. 32-35").

(e5) tags for notes are in `[]` at the very beginning of their text, except when they are preceded by `&` (see `e1` above). Multiple tags are separated by space.

## Import Procedure

(1) get the DOCX document in some folder. Say it is `C:\users\dfusi\desktop\bdm\corfu.docx`.

(2) pick text and essential formatting:

```ps1
.\PickDocx pick C:\users\dfusi\Desktop\bdm\corfu.docx C:\users\dfusi\Desktop\bdm\ -f -x -m
```

This produces `corfu.xml` and `corfu_fmt.xml`. The former file contains the extracted text distributed into its paragraphs and runs. A run is the maximum span of text having the same formatting attributes. Additionally, here we included also footnotes (option `-f`), as they bear comments. The `corfu_fmt.xml` file is the list of all the sets of formatting attributes considered as relevant; each has a numeric ID, which is used to reference them from `corfu.xml`.

(3) run a [Proteus](https://myrmex.github.io/overview/proteus/) pipeline via the [MapAeg CLI tool](https://github.com/vedph/cadmus-bdm-tool) to dump the parsing process:

```ps1
.\bdmtool import C:\users\dfusi\Desktop\bdm\bdm.json c:\users\dfusi\Desktop\bdm\
```

```json
{
  "EntryReader": {
    "Id": "entry-reader.pdcx",
    "Options": {
      "Source": "c:\\users\\dfusi\\desktop\\bdm\\corfu.xml",
      "Mappings": [
        {
          "ElementTag": "par",
          "Entries": ["C block(open=1)"]
        },
        {
          "ElementTag": "par",
          "IsClosing": true,
          "Entries": ["C block(open=0)"]
        },
        {
          "ElementTag": "fn",
          "Entries": ["C fn(open=1)", "T {{content}}"]
        },
        {
          "ElementTag": "fn",
          "IsClosing": true,
          "Entries": ["C fn(open=0)"]
        },
        {
          "ElementTag": "run",
          "Entries": ["T {{content}}"]
        }
      ]
    }
  },
  "EntryFilters": [
    {
      "Id": "entry-filter.txt-merger"
    },
    {
      "Id": "entry-filter.escape",
      "Options": {
        "EscapeDecoders": [
          {
            "Id": "escape-decoder.pattern",
            "Options": {
              "Patterns": [
                {
                  "Pattern": "(?<urls>http[^\\s,)]+)",
                  "Entries": ["C urls(l={urls})"]
                }
              ]
            }
          },
          {
            "Id": "escape-decoder.pattern",
            "Options": {
              "Patterns": [
                {
                  "Pattern": "\\@(?<a>[^,\\d]+),\\s*(?<w>[^;),]+),\\s*(?<l>[^;)]+)(?:;\\s*)?",
                  "Entries": ["C aref(a={a},w={w},l={l})"]
                }
              ]
            }
          },
          {
            "Id": "escape-decoder.pattern",
            "Options": {
              "Patterns": [
                {
                  "Pattern": "\\@(?<a>[^,\\d]+),\\s*(?<l>[^;)]+)(?:;\\s*)?",
                  "Entries": ["C aref(a={a},l={l})"]
                }
              ]
            }
          },
          {
            "Id": "escape-decoder.pattern",
            "Options": {
              "Patterns": [
                {
                  "Pattern": "\\@(?<a>[^,\\d]+)\\s+(?<y>[12]\\d{3})(,\\s*(?<l>[^;)]+)(?:;\\s*)?)?",
                  "Entries": ["C aref(a={a},y={y},l={l})"]
                }
              ]
            }
          },
          {
            "Id": "escape-decoder.pattern",
            "Options": {
              "Patterns": [
                {
                  "Pattern": "^\\s*&?\\s*\\[(?<t>[^\\]]+)\\]",
                  "Entries": ["C tags(t={t})"]
                }
              ]
            }
          }
        ]
      }
    }
  ],
  "EntrySetBoundaryDetector": {
    "Id": "entry-set-detector.cmd",
    "Options": {
      "Name": "block",
      "Type": 4
    }
  },
  "EntryRegionDetectors": [
    {
      "Id": "region-detector.nth-set",
      "Options": {
        "Multiplier": 2,
        "Offset": -1,
        "Tag": "eng"
      }
    },
    {
      "Id": "region-detector.nth-set",
      "Options": {
        "Multiplier": 2,
        "Offset": 0,
        "Tag": "lat"
      }
    },
    {
      "Id": "region-detector.explicit",
      "Options": {
        "Tag": "fn",
        "PairedCommandNames": ["fn"]
      }
    },
    {
      "Id": "region-detector.unmapped",
      "Options": {
        "UnmappedRegionTag": "x"
      }
    }
  ],
  "EntryRegionParsers": [
    {
      "Id": "entry-region-parser.excel-dump",
      "Options": {
        "MaxEntriesPerDumpFile": 10000,
        "OutputDirectory": "c:\\users\\dfusi\\desktop\\bdm\\dump\\"
      }
    }
  ]
}
```

(4) create the MapAeg databases by starting its [API](https://github.com/vedph/cadmus_bdm_api). Then, delete parts and items if they were seeded.

(5) run the same pipeline by replacing the dump entry region parser with the true importer, and specifying a more specialized context, like this:

```json
  "Context": {
    "Id": "entry-set-context.cadmus"
  },
  "EntryRegionParsers": [
    {
      "Id": "entry-region-parser.bdm-text",
      "Options": {
        "GroupId": "corfu"
      }
    }
  ]
```

(6) you can now open the [MapAeg app](https://github.com/vedph/cadmus-bdm-app) and browse the items.
