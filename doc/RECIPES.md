# Recipes

The purpose of this guide is to showcase how you can extend the functionality of CodeCompanion by adding your own prompts to the config that are reflected in the _Action Palette_. The _Action Palette_ is a lua table which is parsed by the plugin and displayed as a `vim.ui.select` component. By specifying certain keys, the behaviour of the table can be customised further.

## Adding a prompt to the palette

A prompt can be added via the `setup` function:

```lua
require("codecompanion").setup({
  default_prompts = {
    ["My New Prompt"] = {
      strategy = "chat",
      description = "Some cool custom prompt you can do",
      prompts = {
        {
          role = "system",
          content = "You are an experienced developer with Lua and Neovim",
        },
        {
          role = "user_header",
          content = "Can you explain why ..."
        }
      },
    }
  }
})
```

In this example, if you run `:CodeCompanionActions`, you should see "My New Prompt" in the bottom of the _Prompts_ section of the palette. Clicking on your new action will initiate the _chat_ strategy and set the value of the chat buffer based on the _role_ and _content_ that's been specified in the prompt. We've set the role to be `user_header` which will automatically map to the value that the user may have specified in their config.

In the following sections, we'll explore how you can customise your prompts even more.

## Recipe #1: Creating boilerplate code

### Boilerplate HTML

As the years go by, I find myself writing less and less HTML. So when it comes to quickly scaffolding out a HTML page, I inevitably turn to a search engine. It would be great if I could have an action that could quickly generate some boilerplate HTML from the _Action Palette_.

Let's take a look at how we can achieve that:

```lua
require("codecompanion").setup({
  default_prompts = {
    ["Boilerplate HTML"] = {
      strategy = "inline",
      description = "Generate some boilerplate HTML",
      opts = {
        mapping = "<LocalLeader>ch"
      },
      prompts = {
        {
          role = "system",
          content = "You are an expert HTML programmer",
        },
        {
          role = "user_header",
          content = "Please generate some HTML boilerplate for me. Return the code only and no markdown codeblocks",
        },
      },
    },
  },
})
```

Nice! We've used some careful prompting to ensure that we get HTML boilerplate back from the LLM. Oh...and notice that I added a key map too!

### Leveraging pre-hooks

To make this example complete, we can leverage a pre-hook to create a new buffer and set the filetype to be html:

```lua
{
  ["Boilerplate HTML"] = {
    strategy = "inline",
    description = "Generate some boilerplate HTML",
    opts = {
      pre_hook = function()
        local bufnr = vim.api.nvim_create_buf(true, false)
        vim.api.nvim_buf_set_option(bufnr, "filetype", "html")
        vim.api.nvim_set_current_buf(bufnr)
        return bufnr
      end
    }
    ---
  }
}
```

For the inline strategy, the plugin will detect a number being returned from the `pre_hook` and assume that is the buffer number you wish any code to be streamed into.

### Conclusion

Whilst this example was useful at demonstrating the functionality of the _Action Palette_ and your custom prompts, it's not using LLMs to add any real value to your workflow (this boilerplate could be a snippet after all!). So let's step things up in the next section.

## Recipe #2: Using context in your prompts

Now let's look at how we can use an LLM to advise us on some code that we have visually selected in a buffer. Infact, this very example is builtin to the plugin as the _Code Advisor_ action:

```lua
require("codecompanion").setup({
  default_prompts = {
    ["Code Expert"] = {
      strategy = "chat",
      description = "Get some special advice from an LLM",
      opts = {
        mapping = "<LocalLeader>ce",
        modes = { "v" },
        slash_cmd = "expert",
        auto_submit = true,
        stop_context_insertion = true,
        user_prompt = true,
      },
      prompts = {
        {
          role = "system",
          content = function(context)
            return "I want you to act as a senior "
              .. context.filetype
              .. " developer. I will ask you specific questions and I want you to return concise explanations and codeblock examples."
          end,
        },
        {
          role = "user_header",
          contains_code = true,
          content = function(context)
            local text = require("codecompanion.helpers.actions").get_code(context.start_line, context.end_line)

            return "I have the following code:\n\n```" .. context.filetype .. "\n" .. text .. "\n```\n\n"
          end,
        },
      },
    },
  },
})
```

At first glance there's a lot of new stuff in this. Let's break it down.

### Palette options

```lua
opts = {
  mapping = "<LocalLeader>ce",
  modes = { "v" },
  slash_cmd = "expert",
  auto_submit = true,
  stop_context_insertion = true,
  user_prompt = true,
},
```

In the `opts` table we're specifying that we only want this action to appear in the _Action Palette_ if we're in visual mode. We're also asking the chat strategy to automatically submit the prompts to the LLM via the `auto_submit = true` value. We're also telling the picker that we want to get the user's input before we action the response with `user_prompt = true`. With the `slash_cmd = "expert"` option, the user can run `:CodeCompanion /expert` from the cmdline in order to trigger this prompt. Finally, as we define a prompt to add any visually selected text to the chat buffer, we need to add the `stop_context_insertion = true` option to prevent the chat buffer from duplicating this. Remember that visually selcting text and opening a chat buffer will result in that selection from being adding as a codeblock.

### Prompt options and context

In the example below you can see how we've structured the prompts to get advice on the code:

```lua
prompts = {
  {
    role = "system",
    content = function(context)
      return "I want you to act as a senior "
        .. context.filetype
        .. " developer. I will ask you specific questions and I want you to return concise explanations and codeblock examples."
    end,
  },
  {
    role = "user_header",
    contains_code = true,
    content = function(context)
      local text = require("codecompanion.helpers.actions").get_code(context.start_line, context.end_line)

      return "I have the following code:\n\n```" .. context.filetype .. "\n" .. text .. "\n```\n\n"
    end,
  },
},
```

One of the most useful features of the custom prompts is the ability to receive context about the current buffer and any lines of code we've selected. An example context table looks like:

```lua
{
  bufnr = 7,
  buftype = "",
  cursor_pos = { 10, 3 },
  end_col = 3,
  end_line = 10,
  filetype = "lua",
  is_normal = false,
  is_visual = true,
  lines = { "local function fire_autocmd(status)", '  vim.api.nvim_exec_autocmds("User", { pattern = "CodeCompanionInline", data = { status = status } })', "end" },
  mode = "V",
  start_col = 1,
  start_line = 8,
  winnr = 1000
}
```

Using the context above, our first prompt then makes more sense:

```lua
{
  role = "system",
  content = function(context)
    return "I want you to act as a senior "
      .. context.filetype
      .. " developer. I will ask you specific questions and I want you to return concise explanations and codeblock examples."
  end,
},
```

We are telling the LLM to act as a "senior _Lua_ developer" based on the filetype of the buffer we initiated the action from.

Lets now take a look at the second prompt:

```lua
{
  role = "user_header",
  contains_code = true,
  content = function(context)
    local text = require("codecompanion.helpers.actions").get_code(context.start_line, context.end_line)

    return "I have the following code:\n\n```" .. context.filetype .. "\n" .. text .. "\n```\n\n"
  end,
},
```

You can see that we're using a handy helper to get the code between two lines and formatting it into a markdown code block.

> [!IMPORTANT]
> We've also specifed a `contains_code = true` flag. If you've turned off the sending of code to LLMs then the plugin will block this from happening.

### Conditionals

It's also possible to conditionally set prompts via a `condition` function that returns a boolean:

```lua
{
  role = "user_header",
  ---
  condition = function(context)
    return context.is_visual
  end,
  ---
},
```

And to determine the visibility of actions in the palette itself:

```lua
{
  name = "Load saved chats ...",
  strategy = "saved_chats",
  description = "Load your previously saved chats",
  condition = function()
    local saved_chats = require("codecompanion.strategies.saved_chats")
    return saved_chats:has_chats()
  end,
  picker = {
    ---
  }
}
```

## Other Configuration Options

**Specifying an Adapter and Model**

```lua
["Your_New_Prompt"] = {
  strategy = "chat",
  description = "Your Special Prompt",
  opts = {
    adapter = {
      name = "ollama",
      model = "deepseek-coder:6.7b",
    },
  },
  -- Your prompts here
}
```

**Specifying a Placement for Inline Prompts**

As outlined in the [README](README.md), an inline prompt can place its response in many different ways. To override this, you can specify a specific placement:

```lua
["Your_New_Prompt"] = {
  strategy = "inline",
  description = "Your Special Inline Prompt",
  opts = {
    placement = "new|false"
  },
  -- Your prompts here
}
```

In this example, the LLM response will be placed in a new buffer and the user's code will not be returned back to them.

## Conclusion

Hopefully this serves as a useful introduction on how you can expand CodeCompanion to create prompts that suit your workflow. It's worth checking out the [actions.lua](https://github.com/olimorris/codecompanion.nvim/blob/main/lua/codecompanion/actions.lua) and [config.lua](https://github.com/olimorris/codecompanion.nvim/blob/main/lua/codecompanion/config.lua) files for more complex examples.
