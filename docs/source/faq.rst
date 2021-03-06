.. index::
   single: FAQ

..  _faq:

Frequently asked questions
==========================

Inserting new lines
-------------------

.. code-block:: javascript

    doc.setOptions({linebreaks: true})

then in your data, if a string contains a newline, it will be translated to a linebreak in the document.

Insert HTML formatted text
--------------------------

It is possible to insert HTML formatted text using the `HTML pro module`_

.. _`HTML pro module`: https://docxtemplater.com/modules/html/


Generate smaller docx using compression
---------------------------------------

The size of the docx output can be big, in the case where you generate the zip the following way:

.. code-block:: javascript

    docx.getZip().generate({ type: "nodebuffer"})

This is because the zip will not be compressed in that case. To force the compression (which could be slow because it is running in JS for files bigger than 10 MB)

.. code-block:: javascript

    var zip = docx.getZip().generate({
            type: "nodebuffer",
            compression: "DEFLATE"
    });

Writing if else
---------------

To write if/else, see the documentation on `sections`_ for if and `inverted sections`_ for else.

You can also have conditions with operators `>` and `<` using `angular parser conditions`_.

.. _`sections`: tag_types.html#sections
.. _`inverted sections`: tag_types.html#inverted-sections
.. _`angular parser conditions`: angular_parse.html#conditions


Conditional Formatting
----------------------

With the `PRO styling module`_ it is possible to have a table cell be styled depending on a given condition (for example).

.. _`PRO styling module`: https://docxtemplater.com/modules/styling/. 

Using data filters
------------------

You might want to be able to show data a bit differently for each template. For this, you can use the angular parser and the filters functionality.

For example, if a user wants to put something in uppercase, you could write in your template :


.. code-block:: text

    { user.name | uppercase }

See `angular parser`_ for comprehensive documentation

.. _`angular parser`: angular_parse.html

Performance
-----------

Docxtemplater is quite fast, for a pretty complex 50 page document, it can generate 250 output of those documents in 44 seconds, which is about 180ms per document.

There is also an interesting blog article https://javascript-ninja.fr/ at https://javascript-ninja.fr/optimizing-speed-in-node-js/ that explains how I optimized loops in docxtemplater.

Support for IE9 and lower 
-------------------------

docxtemplater should work on almost all browsers as of version 1 : IE7 + . Safari, Chrome, Opera, Firefox.

The only 'problem' is to load the binary file into the browser. This is not in docxtemplater's scope, but here is the recommended code to load the zip from the browser:

https://github.com/open-xml-templating/pizzip/blob/master/documentation/howto/read_zip.md

The following code should load the binary content on all browsers:

.. code-block:: javascript

    PizZipUtils.getBinaryContent('path/to/content.zip', function(err, data) {
      if(err) {
        throw err; // or handle err
      }

      var zip = new PizZip(data);
    });

Get list of placeholders
-------------------------

To be able to construct a form dynamically or to validate the document
beforehand, it can be useful to get access to all placeholders defined in a
given template.  Before rendering a document, docxtemplater parses the Word
document into a compiled form.  In this compiled form, the document is stored
in an `AST`_ which contains all the necessary information to get the list of
the variables and list them in a JSON object.

With the simple inspection module, it is possible to get this compiled form and
show the list of tags.
suite : 

.. _`AST`: https://en.wikipedia.org/wiki/Abstract_syntax_tree

.. code-block:: javascript

    var InspectModule = require("docxtemplater/js/inspect-module");
    var iModule = InspectModule();
    doc.attachModule(iModule);
    doc.render(); // doc.compile can also be used to avoid having runtime errors
    var tags = iModule.getAllTags();
    console.log(tags);

With the following template : 

.. code-block:: text

    {company}

    {#users}
    {name}
    {age}
    {/users}

It will log this object :

.. code-block:: json

    {
        "company": {},
        "users": {
            "name": {},
            "age": {},
        },
    }

You can also get a more detailled tree by using : 

.. code-block:: javascript

    console.log(iModule.fullInspected["word/document.xml"]);

The code of the inspect-module is very simple, and can be found here : https://github.com/open-xml-templating/docxtemplater/blob/master/es6/inspect-module.js

Convert to PDF
--------------

It is not possible to convert docx to PDF with docxtemplater, because docxtemplater is a templating engine and doesn't know how to render a given document. There are many
tools to do this conversion.

The first one is to use `libreoffice headless`, which permits you to generate a
PDF from a docx document :

You just have to run :

.. code-block:: bash

   libreoffice --headless --convert-to pdf --outdir . input.docx

This will convert the input.docx file into input.pdf file.

The rendering is not 100% perfect, since it uses libreoffice and not microsoft
word.  If you just want to render some preview of a docx, I think this is a
possible choice.  You can do it from within your application by executing a
process, it is not the most beautiful solution but it works.

If you want something that does the rendering better, I think you should use
some specialized software. `PDFtron`_ is one of them, I haven't used it myself,
but I know that some of the users of docxtemplater use it. (I'm not affiliated to PDFtron in any way).

.. _`PDFtron`: https://www.pdftron.com/pdfnet/addons.html

Pptx support
------------

Docxtemplater handles pptx files without any special configuration (since version 3.0.4).

It does so by detecting whether there is a file called "/word/document.xml", if there is one, the file is "docx", if not, it is pptx.

My document is corrupted, what should I do ?
--------------------------------------------

If you are inserting multiple images inside a loop, it is possible that word cannot handle the docPr attributes correctly. You can try to add the following code just after doing `const doc = new Docxtemplater()` : 

.. code-block:: javascript

    const doc = new Docxtemplater();
    doc.attachModule({
        set(options) {
            if (options.Lexer) {
                this.Lexer = options.Lexer;
            }
            if (options.zip) {
                this.zip = options.zip;
            }
        },
        on(event) {
            if (event !== "syncing-zip") {
                return;
            }
            const zip = this.zip;
            const Lexer = this.Lexer;
            let prId = 1;
            function setSingleAttribute(partValue, attr, attrValue) {
                const regex = new RegExp(`(<.* ${attr}=")([^"]+)(".*)$`);
                if (regex.test(partValue)) {
                    return partValue.replace(regex, `$1${attrValue}$3`);
                }
                let end = partValue.lastIndexOf("/>");
                if (end === -1) {
                    end = partValue.lastIndexOf(">");
                }
                return (
                    partValue.substr(0, end) +
                        ` ${attr}="${attrValue}"` +
                        partValue.substr(end)
                );
            }
            zip.file(/\.xml$/).forEach(function(f) {
                let text = f.asText();
                const xmllexed = Lexer.xmlparse(text, {
                    text: [],
                    other: ["wp:docPr"],
                });
                if (xmllexed.length > 1) {
                    text = xmllexed.reduce(function(fullText, part) {
                        if (part.tag === "wp:docPr") {
                            return fullText + setSingleAttribute(part.value, "id", prId++);
                        }
                        return fullText + part.value;
                    }, "");
                }
                zip.file(f.name, text);
            });
        }
    });

Attaching modules for extra functionality
-----------------------------------------

If you have created or have access to docxtemplater PRO modules, you can attach them with the following code : 


.. code-block:: javascript

    var doc = new Docxtemplater();
    doc.loadZip(zip);

    // You can call attachModule for each modules you wish to include
    doc.attachModule(imageModule)
    doc.attachModule(htmlModule)

    //set the templateVariables
    doc.setData(data);

Ternaries are not working well with angular-parser
--------------------------------------------------

There is a common issue which is to use ternary on scopes that are not the current scope, which makes the ternary appear as if it always showed the second option.

For example, with following data : 

.. code-block:: javascript

   doc.setData({
      user: {
         gender: 'F',
         name: "Mary",
         hobbies: [{
            name: 'play football',
         },{
            name: 'read books',
         }]
      }
   })

And by using the following template :

.. code-block:: text

   {#user}
   {name} is a kind person.

   {#hobbies}
   - {gender == 'F' : 'She' : 'He'} likes to {name}
   {/hobbies}
   {/}

This will print : 


.. code-block:: text

   Mary is a kind person.

   - He likes to play football
   - He likes to read books

Note that the pronoun "He" is used instead of "She".

The reason for this behavior is that the {gender == 'F' : "She" : "He"} expression is evaluating in the scope of hobby, where gender does not even exist. Since the condtion `gender == 'F'` is false (since gender is undefined), the return value is "He". However, in the scope of the hobby, we do not know the gender so the return value should be null.

We can instead write a custom filter that will return "She" if the input is "F", "He" if the input is "M", and null if the input is anything else.

The code would look like this : 

.. code-block:: javascript

    expressions.filters.pronoun = function(input) {
      if(input === "F") {
         return "She";
      }
      if(input === "M") {
         return "He";
      }
      return null;
    }

And use the following in your template :

.. code-block:: text

   {#user}
   {name} is a kind person.

   {#hobbies}
   - {gender | pronoun} likes to {name}
   {/hobbies}
   {/}


Multi scope expressions do not work with the angularParser
----------------------------------------------------------

If you would like to have multi-scope expressions with the angularparser, for example : 

You would like to do : `{#users}{ date - age }{/users}`, where date is in the "global scope", and age in the subscope `users`, as in the following data : 

.. code-block:: json

   {
     "date": 2019,
     "users": [
       {
         "name": "John",
         "age": 44
       },
       {
         "name": "Mary",
         "age": 22
       }
     ]
   }

You can make use of a feature of the angularParser and the fact that docxtemplater gives you access to the whole scopeList.

.. code-block:: javascript

   // Please make sure to use angular-expressions 1.0.1 or later
   // More detail at https://github.com/open-xml-templating/docxtemplater/issues/488
   var expressions = require("angular-expressions");
   var merge = require("lodash/merge");
   function angularParser(tag) {
      if (tag === ".") {
         return {
            get(s) {
               return s;
            },
         };
      }
      const expr = expressions.compile(
          tag.replace(/(’|‘)/g, "'").replace(/(“|”)/g, '"')
      );
      return {
         get(scope, context) {
            let obj = {};
            const scopeList = context.scopeList;
            const num = context.num;
            for (let i = 0, len = num + 1; i < len; i++) {
                obj = merge(obj, scopeList[i]);
            }
            return expr(scope, obj);
         },
      };
   }

   doc.setOptions({parser: angularParser})

.. _cors:

Access to XMLHttpRequest at file.docx from origin 'null' has been blocked by CORS policy
----------------------------------------------------------------------------------------

This happens if you use the HTML sample script but are not using a webserver.

If your browser window shows a URL starting with `file://`, then you are not using a webserver, but the filesystem itself. 

For security reasons, the browsers don't let you load files from the local file system.

To do this, you have to setup a small web server.

The simplest way of starting a webserver is to run following command : 

.. code-block:: bash

   npx http-server
   # if you don't have npx, you can also do :
   # npm install -g http-server && http-server .

On your production server, you should probably use a more robust webserver such as nginx, or any webserver that you are currently using for static files.

Docxtemplater in an angular project
-----------------------------------

There is an `online demo`_ available on stackblitz. 

.. _`online demo`: https://stackblitz.com/edit/angular-docxtemplater-example?file=src%2Fapp%2Fproduct-list%2Fproduct-list.component.ts

If you are using an angular version that supports the `import` keyword, you can use the following code :

.. code-block:: javascript

    import { Component } from "@angular/core";
    import Docxtemplater from "docxtemplater";
    import PizZip from "pizzip";
    import PizZipUtils from "pizzip/utils/index.js";
    import { saveAs } from "file-saver";

    function loadFile(url, callback) {
      PizZipUtils.getBinaryContent(url, callback);
    }

    @Component({
      selector: "app-product-list",
      templateUrl: "./product-list.component.html",
      styleUrls: ["./product-list.component.css"]
    })
    export class ProductListComponent {
      generate() {
        loadFile("https://docxtemplater.com/tag-example.docx", function(
          error,
          content
        ) {
          if (error) {
            throw error;
          }
          var zip = new PizZip(content);
          var doc = new Docxtemplater().loadZip(zip);
          doc.setData({
            first_name: "John",
            last_name: "Doe",
            phone: "0652455478",
            description: "New Website"
          });
          try {
            // render the document (replace all occurences of {first_name} by John, {last_name} by Doe, ...)
            doc.render();
          } catch (error) {
            // The error thrown here contains additional information when logged with JSON.stringify (it contains a properties object containing all suberrors).
            function replaceErrors(key, value) {
              if (value instanceof Error) {
                return Object.getOwnPropertyNames(value).reduce(function(
                  error,
                  key
                ) {
                  error[key] = value[key];
                  return error;
                },
                {});
              }
              return value;
            }
            console.log(JSON.stringify({ error: error }, replaceErrors));

            if (error.properties && error.properties.errors instanceof Array) {
              const errorMessages = error.properties.errors
                .map(function(error) {
                  return error.properties.explanation;
                })
                .join("\n");
              console.log("errorMessages", errorMessages);
              // errorMessages is a humanly readable message looking like this :
              // 'The tag beginning with "foobar" is unopened'
            }
            throw error;
          }
          var out = doc.getZip().generate({
            type: "blob",
            mimeType:
              "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
          }); //Output the document using Data-URI
          saveAs(out, "output.docx");
        });
      }
    }


Getting access to page number / total number of pages or regenerate Table of Contents
-------------------------------------------------------------------------------------

Sometimes, you would like to know what are the total number of pages in the
document, or what is the page number at the current tag position.

This is something that will never be achievable with docxtemplater, because
docxtemplater is only a templating engine : it does know how to parse the docx
format. However, it has no idea on how the docx is rendered at the end : the
width, height of each paragraph determines the number of pages in a document.

Since docxtemplater does not know how to render a docx document, (which
determines the page numbers), this is why it is impossible to regenerate the
page numbers within docxtemplater.

Also, even across different "official" rendering engines, the page numbers may vary. Depending on whether you open a document with Office Online, Word 2013 or Word 2016 or the Mac versions of Word, you can have some slight differences that will, at the end, influence the number of pages or the position of some elements within a page.

The amount of work to write a good rendering engine would be very huge (a few years at least for a team of 5-10 people).

Special character keys with angular parser throws error
-------------------------------------------------------

The error that you could see is this, when using the tag `{être}`.

.. code-block:: text

    Error: [$parse:lexerr] Lexer Error: Unexpected next character  at columns 0-0 [ê] in expression [être].

This is because angular-expressions does not allow non-ascii characters.
You will need angular-expressions version 1.1.0 which adds the
`isIdentifierStart` and `isIdentifierContinue` properties.

You can fix this issue by adding the characters that you would like to support, for example : 

.. code-block:: javascript

    function validChars(ch) {
        return (
            (ch >= "a" && ch <= "z") ||
            (ch >= "A" && ch <= "Z") ||
            ch === "_" ||
            ch === "$" ||
            "ÀÈÌÒÙàèìòùÁÉÍÓÚáéíóúÂÊÎÔÛâêîôûÃÑÕãñõÄËÏÖÜŸäëïöüÿß".indexOf(ch) !== -1
        );
    }
    function angularParser(tag) {
        if (tag === '.') {
            return {
                get: function(s){ return s;}
            };
        }
        const expr = expressions.compile(
            tag.replace(/(’|‘)/g, "'").replace(/(“|”)/g, '"'),
            {
                isIdentifierStart: validChars,
                isIdentifierContinue: validChars
            }
        );
        return {
            get: function(scope, context) {
                let obj = {};
                const scopeList = context.scopeList;
                const num = context.num;
                for (let i = 0, len = num + 1; i < len; i++) {
                    obj = merge(obj, scopeList[i]);
                }
                return expr(scope, obj);
            }
        };
    }
    new Docxtemplater().loadZip(zip).setOptions({parser:angularParser})
