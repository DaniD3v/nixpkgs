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

## Extending Default Values {#sec-option-definitions-extending-defaults}

Sometimes you need to extend an option's default value rather than replace it.
This is particularly relevant when working with options that have computed defaults,
such as those that aggregate values from submodules.

### The Problem

When you define a value for an option, it normally replaces any default value:

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

### Solution 1: Using `mkMerge` with `config`

To extend the default value, reference the option's computed value using `config`
and merge it with your additions using `mkMerge`:

```nix
{
  config.myOption = mkMerge [
    config.myOption  # Include the default/computed value
    { c = 3; }       # Add your own values
  ];
  # Result: { a = 1; b = 2; c = 3; }
}
```

This pattern is especially useful with options that aggregate values from submodules.
For example, when a module computes a configuration based on other module definitions:

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
      default =
        # Computed from all devices
        lib.foldl' lib.recursiveUpdate {}
          (map (dev: dev._config) (lib.attrValues config.devices));
    };
  };
}
```

```nix
{
  # Define some devices
  devices.disk1 = { /* ... */ };
  devices.disk2 = { /* ... */ };
  
  # Extend the aggregated config without losing device configs
  _config = mkMerge [
    config._config  # Preserve computed config from devices
    {
      # Add your own configuration
      fileSystems."/" = { mountPoint = "/"; };
    }
  ];
}
```

### Solution 2: Making Defaults Extensible

Module authors can make their defaults easier to extend by using `mkDefault` when
providing configuration values. This gives user definitions higher priority while
still providing a default:

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
  # User definitions automatically merge with mkDefault values
  config.myOption = { c = 3; };
  # Result: { a = 1; b = 2; c = 3; }
}
```

However, note that this only works when the option type naturally merges values
(like `types.attrs` or `types.attrsOf`). For complex computed defaults,
Solution 1 with `mkMerge` and `config` reference is more reliable.

### When to Use Each Solution

- **Use `mkMerge` with `config` reference** when:
  - You need to extend a computed default value
  - You're working with submodules that aggregate configurations
  - The option has a complex default that depends on other options

- **Use `mkDefault` in option declarations** when:
  - You're a module author providing a simple default
  - You want users to easily override parts of the default
  - The default is static and doesn't depend on other options
