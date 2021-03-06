# Using Puppeteer with Node2Nix

I haven't written anything for a while, but recently, I put together a demo of how to use Puppeteer with Node2Nix. In this post, I'll elaborate some more on how the steps to the demonstration work.

## Initial setup

First, we start off with a normal executable NodeJS program that goes to a page and takes a screenshot.

```js
#!/usr/bin/env node
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto('https://example.com');
  console.log ('navigated to example.com');

  await page.screenshot({path: 'example.png'});
  console.log ('saved screenshot to example.png');

  await browser.close();
})();
```

We also make sure all the regular pieces are there by running `npm init` and `npm install --save puppeteer`.

Then we can get started.

## Steps

### 1. Add "bin": index.js

In package.json:

```js
{
  // ...
  "bin": "index.js",
```

This tells npm that this project has an executable file at `index.js`. There are some various details about this in the npm docs: <https://docs.npmjs.com/files/package.json#bin>

### 2. Run node2nix

You should run `node2nix --nodejs-10`, since node2nix will use a default of near-EOL NodeJS 8.x.

The Node2Nix project is one of the more well-known solutions for generating nix derivations from NodeJS projects. It has been a center of some people's complaints, but it works well enough overall <https://github.com/svanderburg/node2nix>

### 3. Move `default.nix` generated by node2nix to something else

I typically use `node2nix.nix`.

```
$ mv default.nix node2nix.nix
```

I do this becuase I do not want to use the default results of node2nix, which gives you three derivations as set attributes.

### 4. Edit node2nix sources to filter by gitignore (optional)

In node-packages.nix, add `gitignoreSource` as an argument to the top level so we can filter gitignored sources.

```nix
{nodeEnv, fetchurl, fetchgit, globalBuildInputs ? [], gitignoreSource}:
```

Then go look for the `src` attribute defined in the `args` let-binding, and replace the definition.

```diff
  args = {
    name = "puppeteer-node2nix";
    packageName = "puppeteer-node2nix";
    version = "1.0.0";
-   src = ./.;
+   src = pkgs.nix-gitignore.gitignoreSource [ ".git" ] ./.;
    dependencies = [
```

Once these are done, you can add this argument to `node2nix.nix`

```diff
import ./node-packages.nix {
  inherit (pkgs) fetchurl fetchgit;
+ inherit (pkgs.nix-gitignore) gitignoreSource;
  inherit nodeEnv;
}
```

By default, node2nix just blindly uses the current directory files as src, but this means that anytime any file has changed, a new store entry is made. Instead of doing this, I prefer to actually filter out gitignored files. Ideally, you could also specify the very specific files you need in your project.

### 5. Make a new derivation, e.g. `default.nix` using the  package attribute.

```nix
{ pkgs ? import <nixpkgs> {} }:

let
  node2nix = import ./node2nix.nix { inherit pkgs; };

  # note that this derivation does not work because the puppeteer
  # package is not very nice. more details to follow.
  package = node2nix.package;

in pkgs.stdenv.mkDerivation {
  name = "puppeteer-node2nix-demo";

  src = package;

  installPhase = ''
    mkdir -p $out/bin
    ln -s $src/bin/puppeteer-node2nix $out/bin/puppeteer-node2nix
  '';
}
```

Now you can try to build this package and observe puppeteer doing bad things: `nix-build` (optionally with `-j 10` to give it more jobs)

```
ERROR: Failed to download Chromium r674921! Set
"PUPPETEER_SKIP_CHROMIUM_DOWNLOAD" env variable to skip download.
```

This is the beginnings of a normal mkDerivation derivation, which is how most derivations are made in nix. You should know that the purpose of a derivation is to take some source inputs and produce an output, which is why I create a symlink to the puppeteer-node2nix binary that is created in the node2nix.package derivation.

### 6. Set Puppeteer to not download Chrome, as suggested.

```diff
- package = node2nix.package;
+ package = node2nix.package.override {
+   preInstallPhases = "skipChromiumDownload";
+   skipChromiumDownload = ''
+     export PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=1
+   '';
+ };
```

### Note

`preInstallPhases` is one of the phases of the generic builder in nixpkgs mkDerivation <https://nixos.org/nixpkgs/manual/#sec-stdenv-phases>. Because so many derivations in nixpkgs are based on the generic builder and its setup script, you will be well served reviewing these contents and reviewing the sources <https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh>.

-----

Now you can build this and get a binary. But wait, if you run this, you will just get an error that Chormium is missing.

```
$ ./result/bin/puppeteer-node2nix

(node:1988) UnhandledPromiseRejectionWarning: Error: Chromium revision is not
downloaded. Run "npm install" or "yarn install"
```

This is documented in the Puppeteer docs, where `PUPPETEER_SKIP_CHROMIUM_DOWNLOAD` variable is documented <https://github.com/GoogleChrome/puppeteer/blob/v1.19.0/docs/api.md#environment-variables>. This is important also in the next step, where the Chormium executable path must be supplied to the running NodeJS program.

### 7. Wrap the program with a Chromium available to it

Of course, you will soon find in the Puppeteer docs that you need to provide the `PUPPETEER_EXECUTABLE_PATH` environment variable, with a path to the `chromium` binary. That can be done easily using `wrapProgram`.

```diff
in pkgs.stdenv.mkDerivation {
  name = "puppeteer-node2nix-demo";

  src = package;

+ buildInputs = [ pkgs.makeWrapper ];

  installPhase = ''
    mkdir -p $out/bin
    ln -s $src/bin/puppeteer-node2nix $out/bin/puppeteer-node2nix
+
+   wrapProgram $out/bin/puppeteer-node2nix \
+     --set PUPPETEER_EXECUTABLE_PATH ${pkgs.chromium.outPath}/bin/chromium
  '';
}
```

Now when we build the derivation, we can see the result that the original puppeteer-node2nix has now been wrapped with a script that will set the `PUPPETEER_EXECUTABLE_PATH` environment variable:

```
$ ls -a result/bin/
.  ..  puppeteer-node2nix  .puppeteer-node2nix-wrapped
```

Contents of puppeteer-node2nix

```sh
#! /nix/store/{someSha}-bash-4.4-p23/bin/bash -e
export PUPPETEER_EXECUTABLE_PATH='/nix/store/{someSha}-chromium-76.0.3809.87/bin/chromium'
exec -a "$0" "/nix/store/{someSha}-puppeteer-node2nix-demo/bin/.puppeteer-node2nix-wrapped"  "${extraFlagsArray[@]}" "$@"
```

`wrapProgram` is a very useful program for providing various things required in the environment of a program <https://nixos.org/nixpkgs/manual/#ssec-stdenv-functions>. You might find yourself using the `--prefix` option for prefixing PATH arguments more often than providing other environment variables.

## Result

We can now run this as expected:

```
$ ./result/bin/puppeteer-node2nix
navigated to example.com
saved screenshot to example.png
```

## Conclusion

Hopefully this short post with some elaborations on how my demo works will give you some more ideas on how to make your own derivations and work with various crap2nix tools.

## Links

* This repo <https://github.com/justinwoo/puppeteer-node2nix>
* Node2Nix <https://github.com/svanderburg/node2nix>
* generic builder setup.sh <https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/setup.sh>
