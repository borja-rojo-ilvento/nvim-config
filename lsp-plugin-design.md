# LSP Plugin Design Inquiry

## Original Problem

When opening `lua/plugins/lsp_config.lua`, Lua LSP shows diagnostic warnings: `'Undefined global vim'`.

### Root Causes

1. **Load order issue**: `lazydev.nvim` is configured with `ft = 'lua'` in `lua/plugins/lazydev.lua`, meaning it only loads when a Lua file is opened
2. **Timing conflict**: `lsp_config.lua`'s config function runs before lazydev loads, so `lua_ls` initializes without lazydev's type definitions
3. **No automatic reload**: Even when lazydev eventually loads, `lua_ls` doesn't automatically reparse the workspace with the new type definitions

## Design Goal

Restructure `lsp_config.lua` to:
- Lazy load each LSP configuration only when its filetype is needed
- Avoid the monolithic config function that sets up all LSPs at startup
- Maintain shared configuration (LspAttach autocmd, diagnostic config, capabilities)
- Solve the lazydev timing issue naturally

## lazy.nvim Merging Behavior

From [lazy.nvim documentation](https://lazy.folke.io/usage/structuring):

> "opts, dependencies, cmd, event, ft and keys are always merged with the parent spec. Any other property will override the property from the parent spec."

### Critical Implications

- **Merged fields**: `opts`, `dependencies`, `cmd`, `event`, `ft`, `keys`
- **Overridden fields**: `config` and all other fields
- **Best practice**: "Always use opts instead of config when possible. config is almost never needed."

### The Conflict

If multiple plugin specs exist for `neovim/nvim-lspconfig` with different `config` functions, only the last one loaded will execute. Previous config functions are overridden, not merged.

## Proposed Strategies

### Strategy A: Single config + ftplugin files

**Structure:**
```
lua/plugins/lsp/
  init.lua          # Single spec with shared config
after/ftplugin/
  lua.lua           # Calls vim.lsp.enable('lua_ls')
  rust.lua          # Calls vim.lsp.enable('rust_analyzer')
  nix.lua           # Calls vim.lsp.enable('nil_ls')
  python.lua        # Calls vim.lsp.enable('ruff')
```

**Advantages:**
- Uses Neovim's native ftplugin system
- No lazy.nvim merging conflicts
- Clear separation: shared setup in plugin, per-language in ftplugins
- Most idiomatic to Neovim's design

**Disadvantages:**
- Splits configuration between two directory structures
- Ftplugins run for every buffer of that filetype (though this is lightweight)

### Strategy B: Single config + helper module

**Structure:**
```
lua/plugins/lsp/
  init.lua          # Single spec with shared config + autocommands
lua/lsp/
  servers.lua       # Module with setup functions for each server
  servers/
    lua_ls.lua
    rust_analyzer.lua
    nil_ls.lua
    ruff.lua
```

**Advantages:**
- Everything stays in Lua module system
- More explicit control over when servers initialize
- Easy to share server-specific configuration

**Disadvantages:**
- Requires manual autocommand setup for each filetype
- More indirection than Strategy A

### Strategy C: Multiple specs with init instead of config

**Structure:**
```
lua/plugins/lsp/
  init.lua          # Base spec with config for shared setup
  lua_ls.lua        # Spec with ft='lua' and init function
  rust_analyzer.lua # Spec with ft='rust' and init function
  nil_ls.lua        # Spec with ft='nix' and init function
  ruff.lua          # Spec with ft='python' and init function
```

**Advantages:**
- All configuration remains in plugin specs
- Per-language specs can have their own dependencies
- Lazy loading behavior is explicit in each file

**Disadvantages:**
- Uses `init` instead of `config` (runs before plugin loads, not after)
- May need to ensure nvim-lspconfig is already loaded
- More complex mental model

## Shared Configuration Requirements

All strategies need to handle:

1. **Capabilities from blink.cmp**: `require('blink.cmp').get_lsp_capabilities()`
2. **LspAttach autocmd**: Keymaps, document highlighting, inlay hints
3. **Diagnostic configuration**: Signs, virtual text, float settings
4. **lazydev dependency**: Ensure it loads before lua_ls configures

## Current Code Structure

```lua
-- Current lsp_config.lua does:
vim.lsp.enable('lua_ls')       -- Enable each server
vim.lsp.config('lua_ls', {...}) -- Configure with capabilities
-- + LspAttach autocmd (lines 74-176)
-- + Diagnostic config (lines 180-205)
```

These `vim.lsp.enable()` and `vim.lsp.config()` calls are new APIs that can be called from anywhere, not just lazy.nvim's config function.

## Next Steps

1. Choose a strategy based on preferences:
   - Strategy A for native Neovim patterns
   - Strategy B for pure Lua module organization
   - Strategy C for keeping everything in lazy.nvim specs
2. Plan the file structure and split of responsibilities
3. Implement and test with one LSP first
4. Migrate remaining LSPs
5. Verify lazydev loads correctly for lua_ls

## References

- [lazy.nvim Documentation](https://lazy.folke.io/)
- [lazy.nvim Structuring Guide](https://lazy.folke.io/usage/structuring)
- [lazy.nvim Plugin Spec](https://lazy.folke.io/spec)
