# Option Definitions {#sec-option-definitions}

Option definitions are generally straight-forward bindings of values to
option names, like

```nix
{
  config = {
    services.httpd.enable = true;
  };
}
```

However, sometimes you need to wrap an option definition or set of
option definitions in a *property* to achieve certain effects:

## Delaying Conditionals {#sec-option-definitions-delaying-conditionals}

If a set of option definitions is conditional on the value of another
option, you may need to use `mkIf`. Consider, for instance:

```nix
{
  config = if config.services.httpd.enable then {
    environment.systemPackages = [ /* ... */ ];
    # ...
  } else {};
}
```

This definition will cause Nix to fail with an "infinite recursion"
error. Why? Because the value of `config.services.httpd.enable` depends
on the value being constructed here. After all, you could also write the
clearly circular and contradictory:

```nix
{
  config = if config.services.httpd.enable then {
    services.httpd.enable = false;
  } else {
    services.httpd.enable = true;
  };
}
```

The solution is to write:

```nix
{
  config = mkIf config.services.httpd.enable {
    environment.systemPackages = [ /* ... */ ];
    # ...
  };
}
```

The special function `mkIf` causes the evaluation of the conditional to
be "pushed down" into the individual definitions, as if you had written:

```nix
{
  config = {
    environment.systemPackages = if config.services.httpd.enable then [ /* ... */ ] else [];
    # ...
  };
}
```

## Setting Priorities {#sec-option-definitions-setting-priorities}

A module can override the definitions of an option in other modules by
setting an *override priority*. All option definitions that do not have the lowest
priority value are discarded. By default, option definitions have
priority 100 and option defaults have priority 1500.
You can specify an explicit priority by using `mkOverride`, e.g.

```nix
{
  services.openssh.enable = mkOverride 10 false;
}
```

This definition causes all other definitions with priorities above 10 to
be discarded. The function `mkForce` is equal to `mkOverride 50`, and
`mkDefault` is equal to `mkOverride 1000`.

## Ordering Definitions {#sec-option-definitions-ordering}

It is also possible to influence the order in which the definitions for an option are
merged by setting an *order priority* with `mkOrder`. The default order priority is 1000.
The functions `mkBefore` and `mkAfter` are equal to `mkOrder 500` and `mkOrder 1500`, respectively.
As an example,

```nix
{
  hardware.firmware = mkBefore [ myFirmware ];
}
```

This definition ensures that `myFirmware` comes before other unordered
definitions in the final list value of `hardware.firmware`.

Note that this is different from [override priorities](#sec-option-definitions-setting-priorities):
setting an order does not affect whether the definition is included or not.

## Merging Configurations {#sec-option-definitions-merging}

In conjunction with `mkIf`, it is sometimes useful for a module to
return multiple sets of option definitions, to be merged together as if
they were declared in separate modules. This can be done using
`mkMerge`:

```nix
{
  config = mkMerge
    [ # Unconditional stuff.
      { environment.systemPackages = [ /* ... */ ];
      }
      # Conditional stuff.
      (mkIf config.services.bla.enable {
        environment.systemPackages = [ /* ... */ ];
      })
    ];
}
```

## Merging with Defaults Using Priorities {#sec-option-definitions-merging-with-defaults}

When an option has a default value and you want to extend it rather than replace it,
you need to understand how the module system's priority system works with different
option types.

### The Problem with Simple Assignment

When you define a value for an option, it normally replaces the default:

```nix
{
  options.myOption = mkOption {
    type = types.attrs;
    default = { a = 1; b = 2; };
  };
}
```

```nix
{
  # This completely replaces the default
  config.myOption = { c = 3; };
  # Result: { c = 3; }
  # Lost: { a = 1; b = 2; }
}
```

### Solution: Using `mkDefault` for Extensible Defaults

Module authors can make their defaults extensible by providing the default value
in the `config` section using `mkDefault`, rather than in the option's `default`
attribute. This assigns a priority of 1000, allowing user definitions to merge:

```nix
{
  options.myOption = mkOption {
    type = types.attrs;
    default = {};
  };

  config.myOption = mkDefault {
    a = 1;
    b = 2;
  };
}
```

```nix
{
  # User definitions merge with mkDefault defaults for types that support merging
  config.myOption = { c = 3; };
  # Result: { a = 1; b = 2; c = 3; }
}
```

This works because:
1. Option defaults in the `default` attribute have priority 1500 (`mkOptionDefault`)
2. Using `mkDefault` in `config` provides priority 1000
3. User definitions have priority 100 (highest priority)
4. For attribute set types like `types.attrs`, multiple definitions at the same
   priority level are merged using `//` (shallow merge)

Note: Using `mkDefault` inside the option's `default` attribute doesn't work as
expected because the module system automatically wraps the default value with
`mkOptionDefault`, making any inner priority wrapper ineffective.

### Alternative: Separate Extension Options

For complex scenarios where defaults are computed from other options (such as
aggregating values from submodules), the recommended pattern is to provide a
separate option for user extensions:

```nix
{
  options = {
    devices = mkOption {
      type = types.attrsOf (types.submodule { /* ... */ });
      default = {};
    };

    _config = mkOption {
      internal = true;
      description = "Aggregated configuration from all devices";
      default = {};
    };

    extraConfig = mkOption {
      type = types.attrs;
      default = {};
      description = "Additional configuration to merge with device configs";
    };
  };

  config._config = mkMerge [
    # Computed from all devices
    (lib.foldl' lib.recursiveUpdate {}
      (map (dev: dev._config) (lib.attrValues config.devices)))
    # User's extra configuration
    config.extraConfig
  ];
}
```

```nix
{
  devices.disk1 = { /* ... */ };
  devices.disk2 = { /* ... */ };
  
  # Extend using the dedicated extension point
  extraConfig = {
    fileSystems."/" = { mountPoint = "/"; };
  };
}
```

This pattern avoids infinite recursion and provides a clear, explicit way for
users to extend aggregated configurations.

### What If You Can't Modify the Module?

If you're using a third-party module that has a computed default but doesn't use
`mkDefault` or provide an extension option, you cannot cleanly extend the default
value. The module system will replace the default when you assign a value.

In this situation, you have limited options:

1. **Override the entire value**: Accept that you'll replace the default and
   manually include any values from it that you need. This requires understanding
   what the default computes.

2. **Use lower-level options**: Instead of overriding the aggregated option,
   configure the underlying options that feed into it. For example, with disko's
   `_config` that aggregates device configs, configure the devices themselves
   rather than trying to modify `_config`.

3. **Request an upstream change**: File an issue or pull request with the module
   maintainers to add `mkDefault` to their defaults or provide an extension option
   like `extraConfig`.

Example of working with underlying options:

```nix
{
  # Instead of trying to extend disko.devices._config
  # Work with the device-level options directly
  disko.devices.disk.main = {
    device = "/dev/sda";
    content = {
      # Configure device-specific settings here
    };
  };

  # Then configure your additional filesystems through NixOS options
  # rather than through disko's _config
  fileSystems."/" = {
    device = "/dev/mapper/root";
    fsType = "ext4";
  };
}
```

Unfortunately, there is no way to reference an option's default value from within
a configuration assignment without causing infinite recursion. The module system
evaluates options lazily, and referencing `config.foo` while defining `config.foo`
creates a circular dependency.

### When to Use Each Approach

- **Use `mkDefault` in the config section** when:
  - You're a module author providing a default that should be easily overridable
  - You want users to be able to extend the default through merging
  - The option type supports merging (like `types.attrs`)
  - The default is static or computed from other options

- **Provide a separate extension option** when:
  - You're aggregating values from submodules into a computed option
  - You need a clear, explicit extension point for users
  - You want to avoid confusion about how to extend the configuration
  - The aggregation logic is complex
