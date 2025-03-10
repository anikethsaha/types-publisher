
[![Build Status](https://travis-ci.org/microsoft/types-publisher.svg?branch=master)](https://travis-ci.org/microsoft/types-publisher)

# About

This is the source code for the types-publisher service, which publishes the contents of [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) to npm.

# Disclaimer

If there's functionality from the project you'd like to use, please file an issue detailing that. The script isn't intended for public consumption (i.e. we will break the API whenever convenient for us).

# Filing issues

If you've noticed a problem with the way a package is published, file an issue here.
If you don't like the contents of a given definition, file an issue (or pull request) on [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) instead.

# Manually running

Normally, types-publisher is run through its webhook, but to test it out you can do it yourself.
You will need to see the "Environment variables" section first.

```
cat settings.json
```
Make sure your settings are correct.

```
npm run build
npm run full
```

*or*
```
npm run build
npm run clean
npm run parse
npm run check
npm run calculate-versions
npm run generate
npm run index
npm run publish
npm run upload-blobs
```

You can run tests with

```
npm run test
```

# Overview

To update the types packages, the following steps must be performed:

	* Parse the definitions
	* Check for conflicts
	* Calculate versions
	* Generate packages on disk
	* Create a search index
	* Publish packages on disk
	* Upload blobs to Azure

Importantly, each of these steps is *idempotent*.
Running the entire sequence twice should not have any different results unless one of the inputs has changed.

# Parse the definitions

First, obtain a local copy of the DefinitelyTyped repo. For running
locally, the script assumes that it is at `../DefinitelyTyped` and
checks to make sure that it has no outstanding changes. It does *not*
check that it has master checked. For running in the cloud, the script
downloads a gzipped copy and unzips it into memory. This saves a lot
of time if the filesystem is very slow.

You can manually run this step locally with `npm run get-definitely-typed`.
Pass `--dry` to download the DefinitelyTyped copy and unzip it into memory.

> `node bin/parse-definitions.js`

This generates the data file `data/definitions.json`.
All future steps depend on this file.
One can also pass `--single=package_name` to test this on a single package.

## Contents of `data/definitions.json`

This file is a key/value mapping used by other steps in the process.

### Example entry
```js
"jquery": {
	"authors": "Boris Yankov <https://github.com/borisyankov/>",
	"definitionFilename": "jquery.d.ts",
	"libraryDependencies": [],
	"moduleDependencies": [],
	"libraryMajorVersion": "1",
	"libraryMinorVersion": "10",
	"libraryName": "jQuery 1.10.x / 2.0.x",
	"typingsPackageName": "jquery",
	"projectName": "http://jquery.com/",
	"sourceRepoURL": "https://www.github.com/DefinitelyTyped/DefinitelyTyped",
	"kind": "Mixed",
	"globals": [
		"jQuery",
		"$"
	],
	"declaredModules": [
		"jquery"
	],
	"root": "C:\\github\\DefinitelyTyped\\jquery",
	"files": [
		"jquery.d.ts"
	],
	"contentHash": "5cfce9ba1a777bf2eecb20d0830f4f4bcd5eee2e1fd9936ca6c2f2201a44b618"
	}
```

### Fields in `data/definitions.json`

* `"jquery"` (i.e. the property name): The name of the *folder* from the source repo
* `authors`: Author data parsed from a header comment in the entry point .d.ts file
* `definitionFilename`: The filename of the entry point .d.ts file. This file must be either `index.d.ts`, `folderName.d.ts` (where `folderName` is the folder name), or the only .d.ts file in the folder
* `libraryDependencies`: Which other definitions this file depends on. These will refer to *package names*, not *folder names*
* `libraryMajorVersion` / `libraryMinorVersion`: Version data parsed from a header comment in the entry point .d.ts. These values will be `0` if the entry point .d.ts file did not specify a version
* `libraryName`: Library name parsed from a header comment in the entry point .d.ts file
* `typingsPackageName`: The name on NPM that the type package will be published under
* `projectName`: Project name or URL information parsed from a header comment in the entry point .d.ts file
* `sourceRepoURL`: The URL to the originating type definition repo. Currently hardcoded to DefinitelyType's URL
* `kind`: One of the following strings based on the declarations in the folder:
	* `Unknown`: The type of declaration could not be detected
	* `MultipleModules`: Multiple ambient module declarations (`declare module "modName" {`) were found
	* `Mixed`: At least one global declaration and exactly one ambient module declaration
	* `DeclareModule`: Exactly one ambient module declaration and zero global declarations
	* `Global`: Only global declarations. **Preferred**
	* `ProperModule`: Only top-level `import` and `export` declarations. **Preferred**
	* `ModuleAugmentation`: An ambient module declaration and at top-level `import` or `export` declaration. **Preferred**
	* `UMD`: Only top-level `import` and `export` declarations, as well as a UMD declaration. **Preferred**
	* `OldUMD`: Exactly one namespace declaration and exactly one ambient module declaration
* `globals`: A list of *values* declared in the global namespace. Note that this does not include types declared in the global namespace
* `declaredModules`: A list of modules declared. If `kind` is `ProperModule`, this list will explicitly list the containing folder name
* `root`: A full path to the declaration folder
* `files`: A list of the .d.ts files in the declaration folder
* `contentHash`: A hash of the names and contents of the `files` list, used for versioning

## Contents of `logs/parser-log-summary.md`

This log file contains a summary of the outcome of each declaration, as well as a set of warnings.

### Failure States

Currently, the only error condition is if there are multiple .d.ts files in the declaration folder and none of them are the obvious entry point.
These will be listed in the *warnings* section of `parser-log-summary.md`; search for "Found either zero or more" in this file.

### Warnings

The following warnings may be present.
Some warnings block package creation and should be addressed sooner.

#### Too Many Files

> Found either zero or more than one .d.ts file and none of google-apps-script.d.ts or index.d.ts

This warning means the script could not determine what the entry point .d.ts file was.
Fix this by renaming some .d.ts file to the containing folder name, or index.d.ts.
This warning blocks package creation.

#### Incorrect Declared Module

> Declared module `howler` is in folder with incorrect name `howlerjs`

This warning means that a module declaration's name does not match the containing folder's name.
Determine which is correct and rename the folder or the module declaration appropriately.

#### Casing

> Package name joData should be strictly lowercase

Nearly all package names should be lowercased to conform with NPM naming standards.
This warning might not be appropriate; consider logging an issue.

# Check for conflicts

> `node bin/check-parse-results.js`

This is an optional script that checks for multiple declaration packages with the same library name or same project name.

### Contents of `logs/conflicts.md`

> * Duplicate Library Name descriptions "Marked"
>   * marked
>   * ngwysiwyg

Examine these declarations and change them to have distinct library names, if possible.

> * Duplicate Project Name descriptions "https://github.com/jaredhanson/passport-facebook"
>   * passport-facebook
>   * passport-google-oauth
>   * passport-twitter

Examine these declarations and change them to have distinct package names, if possible.

# Calculate versions

This generates `versions.json` based on the last uploaded `versions.json` and by the content hashes computed during parsing.

## Arguments to `calculate-versions`

The `--forceUpdate` argument will cause a build version bump even if the `contentHash` of the originating types folder has not changed.
This argument may be needed during development, but should not be used during routine usage.

# Create a search index

> `node bin/create-search-index.js`

This script creates `data/search-index-min.json`, which (in the upload step) will be uploaded to Azure and used by [TypeSearch](https://github.com/microsoft/typesearch).
This step is not necessary for other steps in the process.

### Arguments to `create-search-index`

You can generate a prettier output in `data/search-index-full.json`.
This version is for human review only and is not compatible with TypeSearch.

By default, `create-search-index` fetches download counts from NPM for use in search result ranking.
The argument `--skipDownloads` disables this behavior.

### Search Entries

Each `search-*.json` file consists of an array.
An example unminified entry is:
```js
{
	"projectName": "http://backgridjs.com/",
	"libraryName": "Backgrid",
	"globals": [
		"Backgrid"
	],
	"typePackageName": "backgrid",
	"declaredExternalModules": [
		"backgrid"
	],
	"downloads": 532234
},
```
These fields should hopefully be self-explanatory.
`downloads` refers to the number in the past month.
If `--skipDownloads` was specified, `downloads` will be -1.
In the case where the type package name is different from the NPM package name, or no NPM package name exists, `downloads` will be 0.

In the minified files, the properties are simply renamed. See `src/lib/search-index-generator.ts` for documentation.

Empty arrays may be elided in future versions of the minified files.

# Generate packages on disk

> `node bin/generate-packages.js`

This step writes all type packages to disk.
The output folder is specified in `settings.json` (see section "Settings").

You can also output a single package with e.g. `node bin/generate-packages.js --single abs`.
(This will still require parsing every package first, as we may need information about referenced packages.)

## Arguments to `generate-packages`

Use the `--single foo` option to generate just the package named "foo".
Use the `--all` option to generate even packages that have not changed.
Use the `--tgz` option to create `.tgz` archives as well. These should represent what is actually uploaded to NPM.

## Outputs of `generate-packages`

### Package Folders

The package generation step creates a folder for each package under the output folder.

The following files are produced automatically:
 * `package.json`
 * `README.md`
 * `metadata.json`: This is the entry from `definitions.json`, excluding the `root` property
 * All declaration files are transformed and copied over

### Definition File Transforms

The following changes occur when a file is transformed:
* `/// <reference path=` directives are changed to corresponding `/// <reference types=` directives
* The file is saved in UTF-8 format

### `logs/package-generator.md`

This file is currently uninteresting.

# Publish packages on disk

> `node bin/publish-packages.js`

This step publishes the files to the NPM registry.

Several keys in `settings.json` affect this step; be sure to read this section.

Before publishing, the script checks the NPM registry to see if a package with the same version number has already been published.
If so, the publishing is skipped.

## Outputs of `publish-packages.js`

### `logs/publishing.md`

This log file indicates which packages were published and which were skipped.
It also indicates any errors that may have occurred during publishing.

Note that unlike other steps, this log file output is *not* idempotent.
Scripts should save this log under a unique filename so any errors may be reviewed.

# Publish registry

> `node bin/publish-registry.js [--dry]`

This step publishes the `types-registry` package on NPM, which keeps a list of all `@types` packages.
This step only happens if there are some new packages to register.

# Upload blobs

This uploads the `data` and `logs` directories to Azure.
`data` always overwrites any old data, while `logs` has a timestamp prepended so old logs can still be viewed.
Blobs can be viewed [here](https://typespublisher.blob.core.windows.net/typespublisher/index.html)
or on [Azure](https://ms.portal.azure.com/?flight=1#resource/subscriptions/99160d5b-9289-4b66-8074-ed268e739e8e/resourceGroups/types-publisher/providers/Microsoft.Storage/storageAccounts/typespublisher).

# Testing the webhook

(Since this is a test, make sure you are not logged in to npm (`npm logout`), and use the `--dry` flag.)

### Testing the webhook without a repository

The script `npm run make-server-run` will trigger the local webhook just like Github would.
(For the production server, use `npm run make-production-server-run`.)

### Testing the webhook with a repository

* Create a dummy repository (e.g. `https://github.com/your/dummy-repo`)

* Set up forwarding:
	* Install [ngrok](https://ngrok.com)
	* `ngrok http 80` (or whatever `PORT` environment variable you're using)
	* Copy the forwarding URL (Looks like: http://deadbeef.ngrok.io)

* Add a hook:
	* Go to https://github.com/your/dummy-repo/settings/hooks
	* Payload URL = url copied from ngrok
	* Secret = swordfish

* Start the server:
	* Change `settings.json`:
		"sourceRepository": "https://github.com/your/dummy-repo"
	* Set the `GITHUB_SECRET` environment variable to `swordfish`
	* `npm install; npm run build`
	* `node bin/webhook.js --dry`

* Make a test change:
	* git clone https://github.com/your/dummy-repo.git
	* Copy the name of the `sourceBranch` from `types-publisher/settings.json`
	* `git checkout -b branch_name`
	* `git push -u origin branch_name`
	* To test again in future, just:
		* `echo "different text" > README.md`
		* `git add --all`
		* `git commit --amend -m "first commit"`
		* `git push -f`

# Using the webhook

```sh
npm run webhook-dry
```

This requires environment variables to be set; see the "Environment variables" section.

# Settings

This file contains settings used by the publisher.

The following properties are supported:

### scopeName

Required. Example value: `types`

This changes the scope name packages are published under.
Do not prefix this value with `@`.

### outputPath

Required. Example value: `./output`

This is the path where packages are written to before publishing.

### definitelyTypedPath

Required. Example value: `../DefinitelyTyped`

This is the path to the DefinitelyTyped (or other similarly-structured) repo.

### sourceRepository

This is the URL of the DefinitelyTyped repo.

### tag

Optional. Example value `latest`

If present, packages are published with the provided version tag.

### azureStorageAccount

Name of the Azure storage account.

### azureContainer

Name of the Azure container.

### errorsIssue

GitHub issue to use to report errors from the webhook.

## Environment variables

#### `TYPES_PUBLISHER_CLIENT_ID` and `TYPES_PUBLISHER_CLIENT_SECRET`

These are needed to access all other secrets. See `src/lib/secrets.ts`.

#### `WEBHOOK_FORCE_DRY`

This lets you run the webhook in dry mode in Azure, without needing command line flags.

#### `PORT`

This is the port the webhook uses for GET requests.

### `LONGJOHN`

Setting this variable turns on [longjohn](https://github.com/mattinsler/longjohn) stacktraces.

### Set environment variables in Azure

* Go to https://ms.portal.azure.com
* Go to `types-publisher` (*not* the `typespublisher` storage account)
* Go to Settings -> General -> Application settings -> App Settings


# Validating published packages

To validate published packages run:

```sh
npm run build
npm run validate [<package>]
```

for instance:

```sh
npm run validate node exress jquery
```

will try to install the three packages, and run the tsc compiler on them.

Specifing no options to the command will validate **all** known packages.


# Publishing to azure

Azure is set up to listen to the `production` branch, which is like `master` but includes `bin/`.

## Update production branch

```sh
npm run push-production
```

This script merges changes from master into production and updates the `bin/` directory.
Azure is listening for changes to `production` and should restart itself.
The server also serves a simple web page [here](http://types-publisher.azurewebsites.net).

## Debugging Azure

While the server is running, you can view logs live:

```sh
npm install -g azure-cli
azure config mode asm
azure login
azure site log tail types-publisher
```

If the server is working normally, you can view log files [here](https://typespublisher.blob.core.windows.net/typespublisher/index.html).

You can view the full server logs at [ftp](ftp://waws-prod-bay-011.ftp.azurewebsites.windows.net).
For FTP credentials, ask Andy or reset them by going to https://ms.portal.azure.com → types-publisher → Quick Start → Reset deployment credentials.
You can also download a ZIP using the azure-cli command `azure site log download`.
The most useful logs are in LogFiles/Application.

## Testing Azure

Instead of waiting for someone to push to DefinitelyTyped,
you should test out your new deployment by running `npm run make-production-server-run`,
which will trigger a full build .
