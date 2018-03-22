# Migrating from Pawno to sampctl

Tutorial for migrating from an old-school Pawno environment to a nice new
sampctl environment!

The goal of this tutorial is to help you drop "The Pawno Way" once and for all.
What I mean when I say "The Pawno Way" is:

* Storing all your includes in a single, monolithic and confusing `includes`
  folder.
* Manually downloading includes from file hosting sites with no regard for
  versioning or updates.
* Using the old compiler with no real knowledge of how it's being invoked or how
  to truly take advantage of its features
* Getting confused when you try to use YSI but it complains about amx_assembly,
  so you download it and it still doesn't work, so you google it and you search
  on the forums and... agh! why won't it work!? Isn't there an easier way?

This tutorial makes a few assumptions:

* You know what sampctl is and vaguely how to use it.
* You're using Windows - as far as I know, most users are on Windows but the
  tools (vscode, sampctl) work on all platforms.
* Your gamemode script is in a folder named `gamemodes`
* You use `#include <>` for including external scripts
* You (may) use `#include ""` for including internal scripts ("modules")

You should learn the basics of sampctl and get a devleopment environment set up
first,
[this will only take you 10 minutes!](http://forum.sa-mp.com/showthread.php?t=651240).

## Turning your Gamemode into a Package

A gamemode can be a
[Pawn Package](https://github.com/Southclaws/sampctl/wiki/Packages) just like
any other. Just like what you created in the "Hello World" tutorial. Your
**server directory** probably looks something like this:

```text
.
├── announce.exe
├── filterscripts/
├── gamemodes/
│   ├── gamemode.amx
│   └── gamemode.pwn
├── pawno/
│   └── include/
│       └── a_samp.inc (etc...)
├── plugins/
├── samp-npc.exe
├── samp-server.exe
├── scriptfiles/
└── server.cfg
```

The goal of this is to get rid of that `pawno` directory forever. It holds 3
things: a bad text editor, an out of date compiler and an unorganised pile of
include files.

So first, just run `sampctl package init` to generate a `pawn.json`/`pawn.yaml`
package definition file and create a `dependencies` directory:

```text
.
├── announce.exe
├── dependencies/
│   ├── pawn-stdlib/
│   └── samp-stdlib/
├── filterscripts/
├── gamemodes/
│   ├── gamemode.amx
│   └── gamemode.pwn
├── pawn.json
├── pawno/
│   └── include/
│       └── a_samp.inc (etc...)
├── plugins/
├── samp-npc.exe
├── samp-server.exe
├── scriptfiles/
└── server.cfg
```

Two useful things happen here

* "Choose an entry point" should detect your gamemode so you can select it and
  add it to the package `entry` and `output` fields - this allows you to build
  your gamemode.
* "Scan for dependencies" will run through your source code looking for popular
  `#include`s and automatically add their proper names as dependencies.

### Dealing with "Legacy" Includes

Now you may not want to perform the full migration all at once. And that's fine,
you might be dependent on outdated includes or includes that have been deleted.
So what you can do is create a temporary folder to store includes in while you
migrate. This will allow you to gradually switch to using package dependencies
one include at a time, instead of doing it all at once.

* First, create a folder called `legacy`
* Then drop all your `.inc` files from `pawno/include` in there
* Finally, delete all the `a_` files, such as `a_samp.inc` - you don't need
  these because they are in `dependencies/samp-stdlib` already because the
  "SA:MP Standard Library" is a dependency of your package. You also need to
  delete the following files because they are part of the Pawn Standard Library,
  which is also now stored in `dependencies/pawn-stdlib`:

  * `core.inc`
  * `datagram.inc`
  * `file.inc`
  * `float.inc`
  * `string.inc`
  * `time.inc`

You now have a folder named `legacy` which contains all the old include files
but sampctl has no knowledge of it.

To solve that, all you need to do is add the following block to your Package
Definition File:

```json
  "builds": [
    {
      "name": "main",
      "includes": ["includes"]
    }
  ]
```

I know not everyone is familiar with manipulating JSON, so assuming your
`pawn.json` looks like this:

```json
{
  "user": "Southclaws",
  "repo": "myserver",
  "entry": "gamemodes/gamemode.pwn",
  "output": "gamemodes/gamemode.amx",
  "dependencies": ["sampctl/samp-stdlib"]
}
```

After, it should look like this:

```json
{
  "user": "Southclaws",
  "repo": "myserver",
  "entry": "gamemodes/gamemode.pwn",
  "output": "gamemodes/gamemode.amx",
  "dependencies": ["sampctl/samp-stdlib"],
  "builds": [
    {
      "name": "main",
      "includes": ["legacy"]
    }
  ]
}
```

Note the extra comma after the `dependencies` block. Commas in JSON have the
same rules as commas in arrays in Pawn: the last element doesn't have a comma
after it.

#### Explanation

So what that does is it adds a `build config` to the package. A `build config`
is a way of changing how the compiler is run. A package can have multiple
builds, but you probably don't need to worry about that just yet.

The fact that we added a single build means it'll be used automatically when you
run `sampctl package build` because this command always picks the first build
config in the list and merges its settings with some defaults. By adding the
`includes` field, we initialise the list of _include paths_ with the name of the
folder we created. During the build process, this list grows in size as all the
directories in `dependencies` are added. You can read more about how this works
[here](https://github.com/Southclaws/sampctl/wiki/How-Builds-Work).

### Building

Now you have your legacy includes directory set up and the gamemode source and
output files defined, you should be able to run `sampctl package build` and
it'll just work.

Once you've got it building the same as it was before, you can start to migrate
from the old includes method to package dependencies.

### Migrating an Include

I'll quickly run through an example of migrating Incognito's Streamer from the
Pawno method to the sampctl method.

#### Step 1: Identify what version you're using

Go into your `legacy` folder, locate `streamer.inc`, open it and check the
version number. The latest (as of writing this) is 2.9.3 but some people had
issues with the 2.9.x branch so a lot of people stuck with 2.8.2.

Whatever the version, write it down somewhere.

#### Step 2: Delete `streamer.inc`

You need to delete the original because if two include files with the same name
exist in any of the include paths, the compiler doesn't know which one you want
to use. Fortunately, sampctl will warn you of this!

#### Step 3: Install the streamer package

Run `sampctl package install samp-incognito/samp-streamer-plugin:VERSION` where
`VERSION` is that version number you wrote earlier.

For example, if I wanted to stay on 2.8.2 for stability reasons:
`sampctl package install samp-incognito/samp-streamer-plugin:2.8.2`

_However_ you can omit the `:` and the version at the end of the command and
**live on the edge!** This means you will _always_ have the latest version
whenever you run `sampctl package ensure`.

## Migrating from `server.cfg`

Another great feature of sampctl is the ability to manage your server
configuration _and_ act as a live server runner and crash-restarter. It can also
automatically download plugins for you.

For this, all you need is a `samp.json` which is the new way to configure
servers. To set that up, simply run `sampctl server init`.

This will ask you a couple of questions and generate a `samp.json`/`samp.yaml`
file, you can now configure your server using this instead of the weird
proprietary server.cfg format.

Run `sampctl server run` to run your server.

## Conclusion

So now you have your server set up to work with sampctl!
[Go check out the docs](https://github.com/Southclaws/sampctl/wiki/) for more
information and to see what else sampctl can do to make your development process
and server management life easier.
