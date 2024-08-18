---
title: "Parsing_kamailio_cfg"
date: 2024-08-18T09:20:33+02:00
---

## Hey there!

If you've ever wrangled with Kamailio's configuration files (kamailio.cfg), you know the pain is real. It’s like trying to solve a puzzle where the pieces are constantly changing shape! I had been using a syntax highlighter but found it a bit buggy/laggy sometimes. As someone who spends a lot of time with these files, I figured there had to be a better way to manage them. And guess what? I found it—Tree-sitter!

In case you haven't heard, Tree-sitter is this cool tool that lets you build parsers for pretty much any language out there. It’s like a Swiss Army knife for syntax—great for stuff like syntax highlighting, code navigation, and more. Plus, it’s already integrated into Neovim and VS Code, which means I didn’t have to jump through hoops to make things work.

So, I decided to take the plunge and build a custom [Tree-sitter parser for Kamailio’s configuration language](https://github.com/IbrahimShahzad/tree-sitter-kamailio-cfg). Here’s how it went down.

Examples of Kamailio configuration files:

```cfg
#!KAMAILIO
#!define WITH_MYSQL
#!define WITH_AUTH
#!define WITH_USRLOCDB

# ----- global configuration parameters -----

debug=3
fork=yes

log_stderror=no

# ----- modules to load -----

loadmodule "db_mysql.so"
loadmodule "auth.so"

# ----- setting module-specific parameters -----

modparam("auth", "db_url", "mysql://kamailio:kamailiorw@localhost/kamailio")
modparam("auth", "password_column", "password")


# ----- request routing logic -----

route {
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483","Too Many Hops");
        exit;
    };

    if (method=="REGISTER") {
        if (!www_authorize("kamailio.org", "subscriber")) {
            www_challenge("kamailio.org", "0");
            exit;
        };
        save("location");
        exit;
    };

    if (method=="INVITE") {
        if (!proxy_authorize("", "subscriber")) {
            proxy_challenge("", "0");
            exit;
        };
        if (!t_relay()) {
            sl_reply_error();
        };
        exit;
    };

    sl_send_reply("404","Not Here");
}
```

## Step 1: Setting the Stage

Before diving into the code, I needed to set up my environment. I installed the Tree-sitter CLI and the Kamailio parser generator. The parser generator is a handy tool that generates the boilerplate code for a new parser. It’s like having a head start on your project!

```bash
cargo install tree-sitter-cli
```
Next, I created a new directory for my project and ran the parser generator.

```bash
tree-sitter init
```

The generator asked me a few questions about my project, like the language I was parsing and the name of the parser. I answered them, and voilà! I had a new parser project ready to go.

## Step 2: Writing the Grammar

The heart of any Tree-sitter parser is its grammar. The grammar defines the rules for parsing the language, like what constitutes a comment, a string, or a function call. Writing a grammar can be a bit tricky, but the parser generator gave me a good starting point.

Kamailio’s configuration language is... well, let’s call it "quirky." You’ve got variables (pseudo_vars), routes, and expressions that seem straightforward until they’re not. The goal was to capture all these quirks without pulling my hair out.

I opened the grammar file and started adding rules for Kamailio’s configuration language. I defined rules for comments, strings, function calls, and more. It was like solving a puzzle, but in a good way!

Here's a sneak peek into what I did:

```javascript
module.exports = grammar({
  name: 'kamailio_cfg',

  extras: _ => [
    /\s|\\\r?\n/,
  ],

  conflicts: $ => [ ... ],

  rules: {
    source_file: $ => seq(
      repeat($.top_level_item)
    ),

    // Top level statements
    top_level_item: $ => choice(
      $.file_starter,
      $.include_file,
      // ...
    ),

    block_item: $ => choice(
      $.statement,
      $.routing_block,
      $.route_call,
      // ...    
      ),

    top_level_statement: $ => choice(
      $.case_statement,
      $.compound_statement,
      alias($.top_level_expression_statement, $.expression_statement),
      $.if_statement,
      $.switch_statement,
      $.while_statement,
      $.return_statement,
      $.break_statement,
      $.continue_statement,
    ),


    // file starter
    file_starter: _ => seq(
      choice(
        token(PREP_START.HASH_BANG),
        token(PREP_START.BANG_BANG)
      ),
      field('cfg_type',
        token(choice(
          KAMAILIO_CFG.SER,
          KAMAILIO_CFG.KAMAILIO,
          KAMAILIO_CFG.OPENSER,
          KAMAILIO_CFG.MAXCOMPAT,
          KAMAILIO_CFG.ALL
        )))),
    })
```

## Step 3: Integrating it with Neovim

Once I had my grammar in place, I compiled the parser and integrated it with Neovim. I installed the Tree-sitter plugin and added the Kamailio parser to my Neovim configuration. And just like that, I had syntax highlighting for Kamailio’s configuration files! I did this so that I could see the fruits of my labor in action.

I also installed [nvim-treesitter-playground](https://github.com/nvim-treesitter/playground) to go along with [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter), a plugin that lets you visualize the syntax tree of your code. It was super helpful for debugging my grammar and understanding how Tree-sitter works. From the documentation, I realized [how to add my custom parser](https://github.com/nvim-treesitter/nvim-treesitter?tab=readme-ov-file#adding-parsers) to Neovim.

In my init.lua file, I added the Kamailio parser to the list of parsers Neovim should use for Kamailio’s configuration files. Here’s a snippet of what I added:

```lua
-- [[ Our Local Parser for Kamailio CFG ]]
local parser_config = require('nvim-treesitter.parsers').get_parser_configs()
parser_config.kamailio_cfg = {
  install_info = {
    url = '~/work/tree-sitter-kamailio-cfg',
    files = { 'src/parser.c' },
  },
  filetype = 'cfg',
}
```

And that was it! I opened a Kamailio configuration file in Neovim, (made sure to run `:TSInstall`) and was able to see the Tree with `:TSPlaygroundToggle`. It was like magic! I could see the syntax tree of my code, however the highlighting was yet to be done, plus the tree was full of errors. BUT now i could see my work in real time and debug it.

## Step 4: Syntax Highlighting

What’s the point of parsing if you can’t make it look good? That’s where syntax highlighting comes in. I set up highlights.scm to map my grammar to highlight groups:

```scm
(comment) @comment

(preproc_def
  (preproc_arg) @variable.parameter)
(preproc_subst
  (preproc_arg) @string.regexp) @keyword.directive
(preproc_substdef
  (preproc_arg) @string.regexp) @keyword.directive.define
(preproc_substdefs
  (preproc_arg) @string.regexp) @keyword.directive.define
(preproc_trydef
  (preproc_arg) @variable.parameter)
(preproc_ifdef) @keyword.directive
(preproc_trydef) @keyword.directive.define
(preproc_def) @keyword.directive.define
(include_file
  file_name: (string) @string.special.path) @keyword.directive
(import_file
  file_name: (string) @string.special.path) @keyword.directive
(route_call) @function

(pseudo_variable
   pvar: (pvar_type) @attribute.builtin
   "(" @punctuation.builtin
   argument: (identifier) @variable.parameter
   ")" @punctuation.builtin) 

(pvar_type) @attribute.builtin
(modparam) @function.builtin
(loadmodule) @function.builtin
(modparamx) @function.builtin
(loadmodulex) @function.builtin
; ...
```

Now, when I pop open a Kamailio config file in Neovim, it’s like Christmas—everything’s highlighted just the way I want. I can see comments, strings, and function calls in all their glory. It’s a small thing, but it makes a big difference when you’re staring at code all day.

## Step 5: Using whats already there

For C-like grammar within the Kamailio configuration file, I used the grammar from [Tree-sitter-c](https://github.com/tree-sitter/tree-sitter-c). This way, I didn’t have to reinvent the wheel for things like strings, comments, and function calls. I just imported the C grammar and built on it to work with Kamailio’s quirks. It saved me a ton of time and effort!


## Step 5: The Grand Finale

I made the first release of my parser on GitHub, and it felt like a huge accomplishment. I was able to take a problem that had been bugging me for a while and turn it into a solution that I could share with others. It was a great feeling! It might have a few rough edges, but it’s a start. I will keep improving it as I go.

![Tree-sitter-kamailio-cfg](images/tree_sitter_kamailio_cfg_image.png)


## What’s Next?
Right now, I’m working on taking this to the next level—building a Language Server Protocol (LSP) for kamailio.cfg. The idea is to add cool features like auto-completion, diagnostics, and go-to-definition. Im thinking of building it in golang, and instead of parsing the whole thing again, I would use the bindings generated by the Tree-sitter CLI. I’m excited to see where this journey takes me! However, this parsing was done with the syntax highlighting in mind, so I might have to make some changes to the grammar to make it more suitable for LSP.

## Wrapping Up

Building a [Tree-sitter parser](https://github.com/IbrahimShahzad/tree-sitter-kamailio-cfg) for kamailio.cfg was a wild ride, but it’s already making my life so much easier. If you’re dealing with Kamailio regularly, I’d love for you to check it out, maybe even contribute!

If you’re into Kamailio, parsing, or just like building cool stuff, keep an eye on this space. We’re just getting started.
