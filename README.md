# gulp-markdown-equations
[![Build Status](https://travis-ci.org/rreusser/gulp-markdown-equations.svg)](https://travis-ci.org/rreusser/gulp-markdown-equations) [![npm version](https://badge.fury.io/js/gulp-markdown-equations.svg)](http://badge.fury.io/js/gulp-markdown-equations) [![Dependency Status](https://david-dm.org/rreusser/gulp-markdown-equations.svg)](https://david-dm.org/rreusser/gulp-markdown-equations)

A gulp plugin that makes it easy to replace markdown latex equations with rendered images


## Introduction

This module exposes the tools necessary to to transform <img alt="undefined" valign="middle" width="57.6" height="19.8" src="https://rawgit.com/rreusser/gulp-markdown-equations/master/docs/images/latex-8ae45909b6.svg"> equations in a markdown document into rendered raster or vector images. It uses the [transform-markdown-mathmode](https://www.npmjs.com/package/transform-markdown-mathmode) node module to locate and transform equations and reconnects with the gulp pipeline after the results have been rendered to complete the transformation using information from the result.

This means you can just mix <img alt="undefined" valign="middle" width="57.6" height="19.8" src="https://rawgit.com/rreusser/gulp-markdown-equations/master/docs/images/latex-8ae45909b6.svg"> into your markdown document. For example,

```markdown
It handles inline equations like $\nabla \cdot \vec{u} = 0$ and display equations like $$\frac{D\rho}{Dt} = 0.$$
```

gets transformed into:

It handles inline equations like <img alt="undefined" valign="middle" width="82.8" height="16.2" src="https://rawgit.com/rreusser/gulp-markdown-equations/master/docs/images/nabla-cdot-vecu-0-5ecc63754f.svg"> and display equations like <p align="center"><img alt="undefined" valign="middle" width="81" height="61.2" src="https://rawgit.com/rreusser/gulp-markdown-equations/master/docs/images/fracdrhodt-0-ebec560456.svg"></p>

Of course it's gulp plugin though, so that means you can really do whatever you want with it!


## Installation

To install, run:

```bash
$ npm install gulp-markdown-equations
```

## Example

The following is a gulp task that locates equations in markdown, renders them, and lets you do whatever you want with the result! First things first, here's the data flow:

<p align="center"><img src="docs/images/flowchart.png" width="408" height="460"></p>

```javascript
var gulp = require('gulp')
  , mdEqs = require('gulp-markdown-equations')
  , tap = require('gulp-tap')
  , filter = require('gulp-filter')
  , latex = require('gulp-latex')
  , pdftocairo = require('gulp-pdftocairo')


gulp.task('mdtex',function() {

  var texFilter = filter('*.tex')
  var mdFilter = filter('*.md')

  // Instantiate the transform and set some defaults:
  var transform = mdEqs({
    defaults: {
      display: { margin: '1pt' },
      inline: {margin: '1pt'}
    }
  })

  return gulp.src('*.mdtex')

    // Locate equations in the markdown stream and pass them as separate
    // *.tex file objects:
    .pipe(transform)

    // Filter to operate on *.tex documents:
    .pipe(texFilter)

    // Render the equations to pdf:
    .pipe(latex())

    // Convert the pdf equations to png:
    .pipe(pdftocairo({format: 'png'}))

    // Send them to the images folder:
    .pipe(gulp.dest('images'))

    // Match the output images up with the closures that are still waiting
    // on their callbacks from the `.pipe(transform)` step above. That means
    // we can use metadata from the image output all the way back  up in
    // the original transform. Sweet!
    .pipe(tap(function(file) {
      transform.completeSync(file,function() {
        var img = '<img alt="'+this.alt+'" valign="middle" src="'+this.path+
                  '" width="'+this.width/2+'" height="'+this.height/2+'">'
        return this.display ? '<p align="center">'+img+'</p>' : img
      })
    }))

    // Restore and then change filters to operate on the *.md document:
    .pipe(texFilter.restore()).pipe(mdFilter)

    // Output in the current directory:
    .pipe(gulp.dest('./'))
})
```

The task is run with:

```bash
$ gulp mdtex
```


## API

#### `require('gulp-markdown-mathmode')( [options] )`
Create a gulp-compatible buffered file stream transform. Options are:

- `defaults`: Parameters that get added to each equation object and passed to the <img alt="undefined" valign="middle" width="57.6" height="19.8" src="https://rawgit.com/rreusser/gulp-markdown-equations/master/docs/images/latex-8ae45909b6.svg"> templator. A good example is setting default margins that can be overridden later on with config variables added to a specific equation.
  - `inline`: an associative array of key/value pairs for inline equations, into which preprocessed parameters are merged
  - `display`: an associative array of key/value pairs for displaystyle equations, into which preprocessed parameters are merged
- `preprocessor`: a function that extracts equation-specific parameters. In other words, if your equation is `$[margin=10pt] y=x$`, the preprocessor extracts `{margin: '10pt'}`. Default preprocessor is [square-parameters](https://github.com/rreusser/square-parameters). If `null`, preprocessor step is skipped. See below for more details.
- `templator`: a function of format `function( tex, params ) {}` that receives the preprocessed `\LaTeX` and parameters and returns a templated <img alt="undefined" valign="middle" width="57.6" height="19.8" src="https://rawgit.com/rreusser/gulp-markdown-equations/master/docs/images/latex-8ae45909b6.svg"> document. If null, the original tex string will be passed through unmodified. Default templator is [equation-to-latex](https://github.com/rreusser/equation-to-latex). See below for more detail.


### Methods:

#### `.completeSync( file, callback )`
Once a file has been processed, you *must* tap into the stream and complete the markdown transformation. See above for an example. `callback` is executed with `this` set to an object containing equation metadata. The value you return is inserted into the markdown document.

The metadata contains all information about the processed equation, including the fully processed file at this point in the pipeline. If image dimensions are available, they will be added. For example:

```javascript
{ display: false,
  inline: true,
  margin: '0 1pt 0pt 0pt',
  path: 'docs/images/latex-d3a0aa2938.png',
  height: 38,
  width: 104,
  equation:
   { tex: '\\LaTeX',
     alt: '&bsol;LaTeX',
     templated: '\\documentclass[10pt] ... \end{document}\n',
     file: <File "latex-d3a0aa2938.png" <Buffer 89 50 ... >> } }
```

#### `.complete( file, callback )`
An asynchronous version of `.complete()`. The format of the callback is `function( resultCallback ) { … }`. Once you've computed the value, you may pass the result to `resultCallback` (e.g. `resultCallback('<img src="...">')`) and it will be inserted into the markdown document.



## Internals

### Preprocessor

The preprocessor is just a function that operates on the content of an equation before it's inserted in the LaTeX template in order to extract equation-specific config. The extracted parameters are merged into this equation's parameters. The default preprocessor [square-parameters](https://github.com/rreusser/square-parameters). Sample usage:

```javascript
var pre = require('square-parameters')`

pre("[name=parabola][margin=10pt 0pt]y=x^2")
// => { params: { name: "parabola", margin: "10pt 0pt" }, content: "y=x^2" }
```

### Templator

The templator's job is to insert <img alt="undefined" valign="middle" width="57.6" height="19.8" src="https://rawgit.com/rreusser/gulp-markdown-equations/master/docs/images/latex-8ae45909b6.svg"> into a full <img alt="undefined" valign="middle" width="57.6" height="19.8" src="https://rawgit.com/rreusser/gulp-markdown-equations/master/docs/images/latex-8ae45909b6.svg"> document. It receives configuration first from preprocessed parameters, then from the display/inline defaults passed as transform options. By default, the delimiters (`$$...$$` or `$...$`) add parameters `{display:true, inline: false}` or `{display:false, inline:true}`, respectively. The default templator is [equation-to-latex](https://github.com/rreusser/equation-to-latex), but in general it only must be a function of form:

```javscript
function templator( content, parameters ) {
  // your logic
  return "\\documentclass... <latex>"
}
```


## Credits

(c) 2015 Ricky Reusser. MIT License