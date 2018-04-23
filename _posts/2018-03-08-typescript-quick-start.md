---
layout: post
title:  "Starting a new Typescript project"
date:   2018-03-08 16:16:01 -0600
categories: Typescript
---

This guide is a quick start guide which help with the setup of a new typescript project.
It will give you everything you need to setup your environment and start building your javascript website.

The editor of choice used is VS Code on Linux. However this guide will still be of use if you are using a different OS and/or editor.

The project setup uses [gulp](https://gulpjs.com/). Gulp is a toolkit for automating painful or time-consuming tasks in your development workflow, so you can stop messing around and build something.

## Pre-Requisites
If you are using a Unix system, you may need to prefix the npm install commands in this guide with sudo.

npm
```
npm -v
npm install npm@latest -g
```
typescript
```
tsc -v
npm install typescript@latest -g
or
npm install -g typescript
```
gulp
```
npm install -g gulp-cli
```

## Getting Started
### Create the Folder Structure
Let’s start out with a new directory. We’ll name it hellohash for now, but you can change it to whatever you want.
```
mkdir hellohash
cd hellohash
```
To start, we’re going to structure our project in the following way:
```
hellohashproj/
   ├─ src/
   └─ dist/
```
TypeScript files will start out in your src folder, run through the TypeScript compiler and end up in dist.

Let’s scaffold this out:
```
mkdir src
mkdir dist
```
### Initialize the project
Now we’ll turn this folder into an npm package.

This will create a package.json file.
```
npm init
```
This will create a tsconfig.json file.
```
tsc --init
```

### Install Dependencies
Install typescript, gulp and gulp-typescript in your project’s dev dependencies. Gulp-typescript is a gulp plugin for Typescript.
```
npm install --save-dev typescript gulp gulp-typescript
```
This will add the 'devDependencies' section to your package.json.

You can also install other dependencies, many of which are available via npm. in this example crypto-js is used
```
npm install crypto-js
```

### Now lets write some code....
A Hello World example with a hash..

Create a new file in the src/ folder called main.ts 

```typescript
import SHA256 = require("crypto-js/sha256");

let newHash = calculateHash(0, "9dfd5ade27d09cabf3b5162757bfab3c11ce8f0b840cf9f2f526bed84263027c", "05/03/2018 12:00:00",  
            { sender: "address1", receiver: "address2", amount: 4 }); 

console.log("Hello. New Hash = " + newHash);

function calculateHash(index:number, previousHash:string, timestamp:string, data:string) { 
    return SHA256(index + previousHash + timestamp + JSON.stringify(data)).toString(); 
}
```
Edit the file tsconfig.json to include your new .ts file
```
{
    "files": [
        "src/main.ts"
    ],
    "compilerOptions": {
        ......
    }
}
```

### Compile and Test
Before we create our glup build pipeline lets manually compile and test our code:
```
tsc
node src/main.js
```
### Create a gulpfile.js
In the project root, create the file gulpfile.js:

```javascript
var gulp = require("gulp");
var ts = require("gulp-typescript");
var tsProject = ts.createProject("tsconfig.json");

gulp.task("default", function () {
    return tsProject.src()
        .pipe(tsProject())
        .js.pipe(gulp.dest("dist"));
});
```
Test the resulting app

```
gulp
node dist/main.js
```
The program should print “Hello. New Hash = ff995303450da99a051f7519ce48361326f51723e9ec68cb6b39ec283636ce39”.


### Add some structure to the code
Create a file called src/hash.ts:

```typescript
import SHA256 = require("crypto-js/sha256");

export function calculateHash(index:number, previousHash:string, timestamp:string, data:any) { 
    return SHA256(index + previousHash + timestamp + JSON.stringify(data)).toString(); 
}
```
Now change the code in src/main.ts to import calculateHash from hash.ts:

```typescript
import { calculateHash } from "./hash"

let newHash = calculateHash(0, "9dfd5ade27d09cabf3b5162757bfab3c11ce8f0b840cf9f2f526bed84263027c", "05/03/2018 12:00:00",  
            { sender: "address1", receiver: "address2", amount: 4 }); 

console.log("Hello. New Hash = " + newHash);
```
Finally, add src/hash.ts to your tsconfig.json:
```
  "files": [
    "src/main.ts",
    "src/hash.ts"
  ],
```
Now you can run your gulp build pipline again and test with node:
```
gulp
node dist/main.js
```

### Bringing it all togther with [Browserify](http://browserify.org/)
Browserify will bundle all of our modules togther to generate a single js file.

```
npm install --save-dev browserify tsify vinyl-source-stream
```

tsify is a Browserify plugin that, like gulp-typescript, gives access to the TypeScript compiler.

vinyl-source-stream lets us adapt the file output of Browserify back into a format that gulp understands called vinyl.

Create a file in src named index.html:
```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8" />
        <title>Hello World!</title>
    </head>
    <body>
        <p id="hash">Loading ...</p>
        <script src="bundle.js"></script>
    </body>
</html>
```
Now change main.ts to update the page:
```typescript
function helloHash() {
    let newHash = calculateHash(0, "9dfd5ade27d09cabf3b5162757bfab3c11ce8f0b840cf9f2f526bed84263027c", "05/03/2018 12:00:00",  
                { sender: "address1", receiver: "address2", amount: 4 }); 
    
    let myDiv = document.getElementById("hash");
    if (newHash && myDiv) {
        myDiv.innerHTML = "Hello. New Hash = " + newHash;
    }
}

helloHash();
```

Now change your gulpfile to the following:
```javascript
var gulp = require("gulp");
var browserify = require("browserify");
var source = require('vinyl-source-stream');
var tsify = require("tsify");
var paths = {
    pages: ['src/*.html']
};

gulp.task("copy-html", function () {
    return gulp.src(paths.pages)
        .pipe(gulp.dest("dist"));
});

gulp.task("default", ["copy-html"], function () {
    return browserify({
        basedir: '.',
        debug: true,
        entries: ['src/main.ts'],
        cache: {},
        packageCache: {}
    })
    .plugin(tsify)
    .bundle()
    .pipe(source('bundle.js'))
    .pipe(gulp.dest("dist"));
});
```
You can now test the page by running gulp and then opening dist/index.html in your browser.

Uglify

The point of Uglify is to minify our code. We also need to install vinyl-buffer.
```
npm install --save-dev gulp-uglify vinyl-buffer gulp-sourcemaps
```
Now update your gulpfile to include the following:
```javascript
var uglify = require('gulp-uglify');
var buffer = require('vinyl-buffer');

gulp.task("default", ["copy-html"], function () {
    ...
    .pipe(buffer())
    .pipe(uglify())
    .pipe(gulp.dest("dist"));
});

```
That's it we're done!
