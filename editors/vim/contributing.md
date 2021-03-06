---
layout: section
order: 5
title: Contributing
---

- TOC
{:toc}

Thank you for your interest in making the ENSIME experience better for Vim users!

If you have a question that isn't answered below, or an idea that you want to discuss before working on it, the best place for help is [the ensime-vim Gitter channel][gitter channel]. If nobody there can help you, please [open an issue][issues] on the repository.

Where to Start
--------------

The first thing to do is to decide what you want to contribute!

- If you think you've identified a bug, well then that was easy. Jump to [Fixing a bug](#fixing-a-bug).

- If you're looking for a small task to get acquainted, check our [issues labeled with **low-hanging fruit**][cherries]. We think these are well-suited for newcomers to make their first few contributions.

- If you're feeling a little more ambitious, try [the **enhancements** label][enhancements]. These extend or improve on existing features.

- If you're ready for a challenge, check [the **feature** label][features] for a new feature wish list.

Our labeling isn't always perfect, and some features are easy and some enhancements are hard. See what inspires you. Whatever you do run with, please leave a comment to let us know so that we can help, and to avoid overlapping with someone else.

[cherries]: https://github.com/ensime/ensime-vim/issues?q=is%3Aissue+is%3Aopen+label%3A%22low-hanging+fruit%22
[enhancements]: https://github.com/ensime/ensime-vim/issues?q=is%3Aissue+is%3Aopen+label%3Aenhancement
[features]: https://github.com/ensime/ensime-vim/issues?q=is%3Aissue+is%3Aopen+label%3Afeature

Working on the Code
-------------------

Once you're ready to get your hands dirty, there are a couple of workflows that you could follow to hack on the code:

1. You probably already have the code, we're guessing you've used ENSIME-Vim before if you're here now :-) Nearly all popular Vim plugin managers (Vundle, vim-plug, NeoBundle, Pathogen) will have used `git clone` to install ENSIME-Vim.
1. `cd` to the directory of your plugin installation.
1. Make sure it is up-to-date and hasn't been left on a detached head:

        $ git checkout master && git pull --unshallow
1. Create a topic branch for your work:

        $ git checkout --branch my-feature-name
1. Write awesome code. Commit it.
1. Fork `ensime/ensime-vim` on GitHub. [Add your fork as a new remote][add git remote] in your local clone:

        $ git remote add me git@github.com:YOURUSER/ensime-vim
1. Push your work to your fork:

        $ git push --set-upstream me my-feature-name
1. When you've gotten your changes working, open a pull request!

Bear in mind that your plugin manager still references `ensime/ensime-vim`. Don't carelessly run any plugin update commands that would affect it while you've got uncommitted work, the manager may try to switch branches, pull from upstream, etc. If you make this mistake and your plugin manager behaves poorly and loses work in this situation, you should consider a new plugin manager.

If you've committed your work it's highly unlikely your plugin manager can ruin anything, Git really does not lose data easily. Most managers have settings specifically geared toward treating a certain plugin as a working copy (e.g. [vim-plug][plug_url_format]), if their default mode of operation isn't suitable. It's how the people who wrote these things *work on their plugin manager* :-)

You can take a possibly more conservative route if you're more comfortable with this:

1. Fork the <https://github.com/ensime/ensime-vim> repo on GitHub.
1. Change your plugin manager's configuration to point to your fork repo. For most managers this means replacing the `ensime` organization name with your own in the argument to the install function (e.g. `Plugin` for Vundle, `Plug` for vim-plug, `NeoBundle` for NeoBundle, etc.).
1. Install the plugin, or manually clone it to your manager's usual plugin directory:

        $ git clone git@github.com:YOURUSER/ensime-vim
        $ git remote add upstream git@github.com:ensime/ensime-vim
1. Create a topic branch for your work:

        $ git checkout --branch my-feature-name
1. Write awesome code. Commit it.
1. Push your work to your fork:

        $ git push --set-upstream origin my-feature-name
1. When you've gotten your changes working, open a pull request!

Since `master` on your fork won't update until *you* update it, it's a bit less likely that a careless plugin update command will create trouble for your work in progress, but it's still possible. The popular managers *should* work safely with either workflow.

Also, generally all of them will let you set an arbitrary local filesystem path for a plugin, so you could keep your copy wherever you like to keep your projects if you wish. Check your manager's help.

*I hate how long this section is because there are a dozen damned Vim plugin managers. **Ommmm...***

[add git remote]: https://help.github.com/articles/adding-a-remote/
[plug_url_format]: https://github.com/junegunn/vim-plug/wiki/faq#whats-the-deal-with-git-in-the-url

Finding Your Way Around the Code
--------------------------------

ENSIME-Vim is written in a combination of VimL and Python. Interaction with ENSIME requires asynchrony, so the bulk of the code is in Python and this is where you'll spend most of your time hacking on the plugin. It communicates over ENSIME's "Jerky" protocol, which is JSON over WebSockets.

The VimL layer is thin but nevertheless important, since it not only defines the user interface from Vim, but also a public API for user scripting, integration, and customization. Our hope is to provide a plugin that feels like a "native", idiomatic extension of Vim, including its function interface. It's a known issue that improvement is still needed in this area and work on it will come.

### Tour of Initialization ###

As with any Vim plugin, Vim first loads the file in `plugin/ensime.vim` eagerly. For standard Vim, this defines the main commands that users invoke when using the plugin, registers event handlers for [autocommands], etc. For Neovim, we explicitly skip loading this file, instead defining the same set of commands with Neovim's [remote plugin host] API. This happens in `rplugin/python/ensime.py`, which Neovim (only) loads by default (almost, `:UpdateRemotePlugins` is needed to register it). This is confined to a minimal shim because we want to share as much code as possible between Vim and Neovim, but it does open the possibility for some commands to be non-blocking in Neovim.

Initialization stops at this point, if you never edit a Scala file during a Vim session. Once you do, though, ENSIME-Vim picks up where it left off: our registered autocommands fire and their callbacks trigger the loading of the rest of the plugin.

In Vim, the callbacks are VimL functions that trigger `autoload/ensime.vim` to load lazily by Vim's standard autoloading naming conventions (function names prefixed with `ensime#`). Functions defined there proxy to the Python core of the plugin, the `ensime_shared` directory. In Neovim, the autocommand callbacks dispatch directly to the Python core, so the `autoload/ensime.vim` is skipped over, it never loads.

That's how the plugin starts up, now you've come to the point where things get interesting.

[autocommands]: http://vimdoc.sourceforge.net/htmldoc/autocmd.html
[remote plugin host]: https://neovim.io/doc/user/remote_plugin.html

### ENSIME Integration

The majority of the interesting code lives in `ensime_shared`. The files there have module documentation describing their purpose, but here is an overview of the most salient:

  - `ensime.py` is the core of the plugin and the bridging point between client/server communication with ENSIME and Vim as the UI and event layer.
  - `client.py` implements logic for connecting to the server and exchanging requests and responses.
  - `protocol.py` implements the client behavior for handling messages received from the server.
  - `launcher.py` is responsible for starting the ENSIME server process using a project's `.ensime` config.

If your project has an `.ensime` file, the `Ensime` class in `ensime.py` will detect it and call on `EnsimeLauncher` to launch a server. It also starts an `EnsimeClient` which will take over communicating with the server and updating Vim when responses come back.

The tour bus stops here, from this point you've just got to dive into the code!

Common Development Tasks
------------------------

### Fixing a bug ###

First understand if the bug is happening in ENSIME or the Vim plugin. The best way to determine this is to watch the debug-level log while you issue a plugin command. You will see errors as well as pretty-printed messages exchanged between the client and server.

Enable debug logging by setting the environment variable `ENSIME_VIM_DEBUG=1` when you run Vim (or put `let $ENSIME_VIM_DEBUG = 1` in your vimrc). Then analyze the output in `.ensime_cache/ensime-vim.log` of your project:

  - If the bug is on what we're sending to ENSIME, simply follow the path of the command through the code until you reach the `send` call.
  - For bugs in response handling, you'll want to check the `handlers` dictionary in `protocol.py` and find the function mapped to the ENSIME server response message type you're seeing (the value of the top-level `typehint` key). A few handler functions are implemented in mixins in other files.

If you've just landed here and this sounds utterly bewildering, you might want to see [Finding Your Way Around the Code](#finding-your-way-around-the-code) above.

### Extending an existing Command ###

  - All the existing commands are defined in `rplugin/python/ensime.py` and `plugin/ensime.vim`, for Neovim and Vim respectively.
  - Each of the `com_do_something` calls there are implemented in the `Ensime` class in `ensime_shared/ensime.py`, which indicates which API handling functions are called.
  - To change a command's behavior, you'll likely only need to make changes in the request/response handling code, as the command has already been wired up.

### Adding a new Command ###

  - New ENSIME-Vim commands are a very similar to existing ones, except you're responsible for determining whether it is a standard command or an autocommand. Standard commands are initiated by the user, while autocommands are executed by Vim whenever certain events occur. For a good overview of commands & autocommands, see Steve Losh's book [Learn VimScript the Hard way](http://learnvimscriptthehardway.stevelosh.com/chapters/12.html)
  - The new command should be added to `plugin/ensime.vim`, `rplugin/python/ensime.py`, `autoload/ensime.vim`, and of course implemented in `ensime_shared/ensime.py`, plus any helper files necessary.
  - To expose your new command, Neovim users may need to run `:UpdateRemotePlugins`.

### Exposing New Vim Functionality to the Plugin Core ###

We do our best to encapsulate Vim functionality in the `Editor` class defined in `editor.py`. This wraps both complex raw Vim commands that are `eval`ed plus features covered by the Vim Python API in a constrained and hopefully consistent interface. The intention is to provide better maintainability than scattering `vim.command('foo')` usage throughout the codebase and to promote single-responsibility design.

Review the functionality exposed there, and if what you need isn't covered, add it in a style consistent with the rest. At time of writing the `Editor` class is going through a design overhaul with a mix of old and new API—the new API (which you should model after) has thorough unit tests; the old, incidentally, does not.

Testing
-------

The test suite consists of unit tests implemented with the [pytest][] framework. These are located in the `test` directory and can be run with `make unit` or the `py.test` runner command as you like.

Pytest is a somewhat unconventional test framework, but a popular one in the Python community and quite pleasant once you're familiar with a few of its idioms. One of the most fundamental to understand is its [dependency injection-style "fixtures" as function parameters][pytest di]—you'll find a mock of Vim's `vim` Python object provided to many unit tests such as in the `test_editor` module, for example. That `vim` mock fixture is defined in `test/conftest.py`, [which pytest loads][conftest].

Addition of a BDD integration/acceptance test framework is [under review][pull 312] for more complex and end-to-end coverage situations against an embedded Neovim. It is also hoped that higher-level acceptance specs expressed in the Gherkin format—implemented in many languages—could eventually be shared with other editor plugins as a TCK. If you're interested in that, see if you can help to push the pull request over the finish line!

### Using Tools in virtualenv ###

Running `make` will automatically create a Python `virtualenv` sandbox for you in the `.venv` subdirectory of ensime-vim. If you want to invoke tools like the `py.test` CLI or a `python` REPL where you can import the project modules or dependencies to experiment, you can activate that virtualenv by sourcing its script in your shell:

```sh
$ . .venv/bin/activate
```

Running `python` then runs the sandboxed executable with access to our dependencies.

If you're Python development-savvy and you want to manage your own virtualenv for the project so you can do things like integrate with `vim-pyenv`, you can specify your virtualenv with an environment variable pointing to its root:

```sh
$ export VENV=/Users/ches/.pyenv/versions/2.7-ensime-vim
```

Our `Makefile` will then honor use of your virtualenv location.

[pytest]: http://pytest.org/
[pytest di]: http://docs.pytest.org/en/latest/fixture.html#fixtures-as-function-arguments
[conftest]: http://docs.pytest.org/en/latest/writing_plugins.html#plugin-discovery-order-at-tool-startup
[pull 312]: https://github.com/ensime/ensime-vim/pull/312

Code Style
----------

Generally speaking, we try to automate compliance with project code style preferences: run `make lint format` and heed any output.

As always, code style is a guideline, not a rule. Readability trumps dogma. If you have good reasoning for a style transgression, you should feel welcome to propose a change to our tool settings, or silence one-offs with the override comment syntax supported by respective tools (consult the docs for `flake8` and `autopep8`).

Useful References
-----------------

- [ENSIME Contributing](/contributing/)
- [ENSIME API Source](https://github.com/ensime/ensime-server)
- [Learn VimScript The Hard Way](http://learnvimscriptthehardway.stevelosh.com/)
- Vim Documentation (`:h <your-term-here>`) – some useful tags for how Vim plugins should work are `write-plugin` and `write-filetype-plugin`, as well as `python` for the Python plugin API supported by both Vim and Neovim)
- Exemplary Vim Plugins:
    - [vim-go](https://github.com/fatih/vim-go) is a full-featured language support plugin that integrates with many external semantic tools, and tastefully uses and extends Vim's core features.
    - [NerdTree](https://github.com/scrooloose/nerdtree) has great insights on how to extend Vim's UI.
    - [vimmode-python](https://github.com/klen/python-mode) is helpful for learning about autocomplete, location vs. quickfix lists, preview buffers, etc. And it might be helpful for working on ensime-vim itself :-)

[gitter channel]: https://gitter.im/ensime/ensime-vim
[issues]: https://github.com/ensime/ensime-vim/issues
