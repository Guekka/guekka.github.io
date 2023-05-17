+++
title = "NixOS as a server, part 2: Flake, tailscale"
date = 2023-05-17

[taxonomies]
categories = ["Projects"]
tags = ["nix", "self-hosting"]
+++

In the [previous part](https://guekka.github.io/nixos-server-1/), we configured our NixOS server to use impermanence. I have made a few changes since, most notably moving to a proper VM in Proxmox.

The following instructions might lack some details, but you can follow [the GitHub repo](https://github.com/Guekka/nixos-server/commits/2-tailscale) to see the full code.

# Moving to flakes

> If you already know about flakes, you can safely ignore this part.

Have you heard about Nix flakes? If you have been in the Nix ecosystem for more than a few days, most likely. They're the shiny new way to write Nix code, still experimental but used everywhere.
Their main advantage over traditional Nix is *purity*, mainly with their defined `inputs` and `outputs`. 

Remember when I told you Nix was reproducible? It was a lie. Let me explain myself: when writing Nix code, we always have some kind of input. For example, `nixpkgs` will be required almost all the time. There are two ways to obtain it.
- fetch it: `import builtins.fetchTarball "https://github.com/nixos/nixpkgs/archive/nixos-22.11.tar.gz";`
- or more commonly, use *channels*: `import <nixpkgs> {}`

This second way uses a globally-defined configuration, which can change externally to our Nix files. We thus lose complete reproducibility. Instead, flakes allow us to avoid channels by specifying inputs alongside Nix configuration, as well as blocking some actions that could hinder reproducibility.

For a more in depth introduction, have a look at [the wiki](https://nixos.wiki/wiki/Flakes#See_also).

Now, why do we want to migrate to flakes? We do not have external requirements, do we?
Well, yes, we do. Apart from the obvious `nixpkgs` dependency, which is configured system-wide, the impermanence module is being imported:
```nix
  impermanence = builtins.fetchTarball "https://github.com/Nix-community/impermanence/archive/master.tar.gz";
```
This `fetchTarball` call is bad, by the way, as we do not specify the expected hash. We could be a victim of a man-in-the-middle attack and not notice it.

Now that I intend to add more modules, and possibly use the `unstable` channel, it is better to migrate. Let's see what our entry point would look like:
```nix
{
  # what is consumed (previously provided by channels and fetchTarball)
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-22.11"; # (1)
    impermanence.url = "github:Nix-community/impermanence";
  };

  # what will be produced (i.e. the build)
  outputs = { nixpkgs, ... }@inputs: { # (2)
    nixosConfigurations = { # (3)
      server = nixpkgs.lib.nixosSystem { # (4)
        packages = nixpkgs.legacyPackages.x86_64-linux; # (5)
        specialArgs = inputs; # forward inputs to modules
        modules = [
          ./configuration.Nix
        ];
      };
    };
  };
}
```
That's a lot to understand at once. Let's study it one line at a time.
Firstly, we have to understand this file is simply describing `outputs` as a function taking `inputs`. Like in mathematics, we create the same output given the same input: a *pure* function, would say functional programmers.

`(1)` is defining an input: we simply give it an url. That line can be translated as *Grab the `nixos-22.11` branch from the GitHub repo `nixpkgs` owned by `NixOS`.

`(2)` is defining the `outputs` function. Most complicated things are defined using functions in Nix. It takes a *named attribute set* argument as an input. So this syntax:
```nix
{ nixpkgs, ... }@inputs: nixpkgs
```
is the same as:
```nix
inputs: inputs.nixpkgs
```
In both cases, we're accessing the `nixpkgs` property of the `inputs` set.

But in the first case, we don't have to repeat `inputs.` everywhere. In JS, you would call that *destructuring*: it is just making inner elements easier to access. If you have troubles understanding the Nix syntax, I personally like [FasterThanLime article](https://fasterthanli.me/series/building-a-rust-service-with-Nix/part-9).

`(3)`: NixOS configuration have to be placed specifically in the `nixosConfigurations` set.

`(4)` is the place where we actually define the system. We call the `nixosSystem` function and pass it some arguments. Yes, the whole system is an `output` too!

`(5)`: we give the packages instance to our system. In our case, we are passing the default packages, but we might want to modify them before. We also have to specify our architecture (`x86_64`).

That's pretty much it. With that `flake.Nix` in `/etc/nixos`, `nixos-rebuild` will work as before. However, if you're using git, beware that your files all need to be under version control or Nix will not see them.

# Secrets with Sops

In order to set up Tailscale, we will use a pre-auth key. This allows us to connect to our server without interaction. However, we must hide this key, or other people could join our Tailscale network, which could obviously have dangerous consequences.

There are 2 well-known solutions : agenix and sops-nix. I've chosen sops for no particular reason.
The first step will be to add it to our flake. See, we already get a use for it!

## Importing `sops-nix`

Let's change our `inputs`:
```nix
  inputs = {
    # ...
    sops-nix = {
      url = "github:mic92/sops-nix";
      inputs.nixpkgs.follows ="nixpkgs";
    };
  };
```

### `Follows`

What's up with this `follows`? `sops-nix` already depends on `nixpkgs`, but it might use a different revision than ours. Making it use our own has several advantages:
- improve consistency
- reduce the number of evaluations required

And how do we know if a package has inputs that need to be redirected? That's the neat thing, we don't. Either we have to look at the upstream `flake.nix`, or we can call `nix flake info` and get a graph like so:
```
Resolved URL:  git+file:///etc/nixos
Locked URL:    git+file:///etc/nixos
Path:          /Nix/store/4b14z6ki7av3kid69sp5vgf50wzd3a73-source
Last modified: 2023-04-17 14:04:13
Inputs:
├───impermanence: github:Nix-community/impermanence/6138e
├───nixpkgs: github:NixOS/nixpkgs/39fa0
└───sops-Nix: github:mic92/sops-nix/de651
    ├───nixpkgs follows input 'nixpkgs'
    └───nixpkgs-stable: github:NixOS/nixpkgs/1040c
```
We can notice `sops-nix` also has a `nixpkgs-stable` input, that we might as well redirect.

## Generating a key

`sops-nix` works by encrypting our secrets with private keys.
We thus need to provide it with the keys we will use. We can generate an `age` key, or get one from our `SSH` host key.
Each secrets group can have different allowed keys, so that one user cannot access another's secrets.
I will use the SSH host key for my server:
```
$ nix-shell -p ssh-to-age --run 'cat /etc/ssh/ssh_host_ed25519_key.pub | ssh-to-age'
age1dt24qetqhy2ng53fyj69yq9hg8rdsg4ep0lvvhdg69xw9v4l0asqj6xzkh
```
We now have to write `.sops.yaml` file in order to configure which keys can access which secrets.
```yaml
keys:
  - &horus age1dt24qetqhy2ng53fyj69yq9hg8rdsg4ep0lvvhdg69xw9v4l0asqj6xzkh
creation_rules:
  - path_regex: hosts/horus/secrets.yaml$
    key_groups:
    - age:
      - *horus
  - path_regex: hosts/common/secrets.yaml$
    key_groups:
    - age:
      - *horus
```
That's it for decryption. However, we need to write secrets too. For that, we can get the corresponding private key:
```sh
nix-shell -p ssh-to-age --run "sudo ssh-to-age -private-key -i /etc/ssh/ssh_host_ed25519_key | install -D -m 400 /dev/stdin ~/.config/sops/age/keys.txt"
```
That `install` bit is here to create the directory if it doesn't exist and set the right permissions.

> Isn't this insecure? The key is not password-locked.

Indeed, if someone has access to our user account, they can read that key and decrypt the secrets. However, we can probably assume our user already has access to the local secrets, so it doesn't matter much. Our goal is to be able to put these secrets on a public GitHub, not to protect them locally.

## Configuring `sops-nix`

Our last step is to configure `sops`.
We're going to get fancy here, as I'm ~~stealing~~ borrowing a module from Misterio's config. In the future, this will often happen, as his config happens to be a great resource. Let's have a look at `sops.nix`:
```nix
{ sops-nix, lib, config, ... }:
let
  isEd25519 = k: k.type == "ed25519";
  getKeyPath = k: k.path;
  keys = builtins.filter isEd25519 config.services.openssh.hostKeys;
in
{
  imports = [
    sops-nix.nixosModules.sops
  ];

  sops = {
    age.sshKeyPaths = map getKeyPath keys;
  };
}
```
This looks complicated, but it is not. First, we are declaring some functions in the `let` block.
- `isEd25519` simply tells if an SSH key uses `ed25519`
- `getKeyPath` gets the path of an SSH key
- `keys` is the list of `ed25519` keys, taken from `openssh`

Then we import `sops`. Finally, we give it the keys we collected earlier. This avoids hardcoding keys, which is great!

We can now import this module in our config:
```nix
  imports =
    [
      impermanence.nixosModule
      ./hardware-configuration.Nix
      ../../modules/sops.nix
    ];
```

`sops-nix` is now ready to use. Do not forget to rebuild the config.

## Our first secret

Let's write a secret:
```sh
mkdir -p hosts/horus
nix-shell -p sops --run "sops hosts/horus/secrets.yaml"
```
An editor should open. We can now write secrets, using yaml. Once we're done, we can save the file. Example content:
```yaml
tailscale_key: e2b6595884993e001da58d2995af65df489582a702e3a2f3
```
We now have to tell `sops` this secret exists. So we declare it somewhere in our configuration:
```
sops.secrets.tailscale_key = {
  sopsFile = ./secrets.yaml;
};
```
And that's all! To use it, we simply have to use `config.sops.secrets.tailscale_key.path` where we need it. Beware that this will not give you the secret, but a path to a file containing the raw secret, for security reasons. Otherwise, the secret would be in the Nix store, and thus accessible to any user on the system.

## Note: adding a new host

If you ever need to add a new host, you will need to update your secrets with `sops updatekeys your_secret`. This command has to be on a system with already authorized keys.

# Tailscale

We can finally get to a real feature, setting up Tailscale.
For those of you who haven't heard of it, Tailscale is a private meshed network, allowing you to connect to your machines privately and securely through Wireguard, a VPN protocol, without exposing them to the world.
This means being able to close port 22, while still being able to SSH into your computer.

Furthermore, Tailscale offers some additional features, such as a fancy file sending tool or hole punching, which allows you to connect to your computer even if it is behind a NAT. I won't go into details here, but you can read more about it on [their website](https://tailscale.com/).

I've chosen to write a full-fledged NixOS module for Tailscale, as it is a service that needs to be configured and started. This is a good example of a module that can be reused in other configurations, so it's worth writing it. Let's get started!

## Boilerplate for NixOS modules

We're going to write a module, so we need to create a directory for it:
```sh
mkdir -p modules/nixos
```

Inside, we'll need a `default.nix` file:
```nix
{
  tailscale-autoconnect = import ./tailscale-autoconnect.nix;
}
```
 By convention, this file is automatically imported when you import a directory. Then, in our `flake.nix`, we can import our module:
```nix
outputs = # ...
{
  nixosModules = import ./modules/nixos;
}
```
As before, the `nixosModules` attribute has a special meaning.

Finally, we have to import the module we're writing in our configuration:
```nix
  imports =
    [
      # ...
      outputs.nixosModules.tailscale-autoconnect
    ];
```

## Writing the module

We're going to write a module to start Tailscale and connect to it automatically. This is a good example of a module that can be reused in other configurations, so it's worth writing it. Let's get started!

First, we need to create a `tailscale-autoconnect.nix` file in our `modules/nixos` directory. We'll start with the boilerplate:
```nix
{ config, lib, pkgs, ... }:
{
  with lib; let
    cfg = config.services.tailscaleAutoconnect; 
  in {
    options = {
      services.tailscaleAutoconnect = {
        enable = mkEnableOption "tailscaleAutoconnect";
      };
    };

    config = mkIf cfg.services.tailscaleAutoconnect.enable {
      # ...
    };
  };
}
```
This is the basic structure of a module. We declare an option, and then we use it to conditionally change the configuration. So:
- What we write in `options` is the option declaration
- What we write in `config` is the consequence of the option being enabled, the configuration change

Let's declare all the options first.
```nix
  options.services.tailscaleAutoconnect = {
    enable = mkEnableOption "tailscaleAutoconnect";
    authkeyFile = mkOption {
      type = types.str;
      description = "The authkey to use for authentication with Tailscale";
    };

    loginServer = mkOption {
      type = types.str;
      default = "";
      description = "The login server to use for authentication with Tailscale";
    };

    advertiseExitNode = mkOption {
      type = types.bool;
      default = false;
      description = "Whether to advertise this node as an exit node";
    };

    exitNode = mkOption {
      type = types.str;
      default = "";
      description = "The exit node to use for this node";
    };

    exitNodeAllowLanAccess = mkOption {
      type = types.bool;
      default = false;
      description = "Whether to allow LAN access to this node";
    };
  };
```
This looks like a lot of code, but we're simply declaring options. We need to give them a type, and we can also give a default value and a description.
Now, the actually useful code:
```nix
  config = mkIf cfg.enable {
    assertions = [
      {
        assertion = cfg.authkeyFile != "";
        message = "authkeyFile must be set";
      }
      {
        assertion = cfg.exitNodeAllowLanAccess -> cfg.exitNode != "";
        message = "exitNodeAllowLanAccess must be false if exitNode is not set";
      }
      {
        assertion = cfg.advertiseExitNode -> cfg.exitNode == "";d
        message = "advertiseExitNode must be false if exitNode is set";
      }
    ];

    systemd.services.tailscale-autoconnect = {
      description = "Automatic connection to Tailscale";

      # make sure tailscale is running before trying to connect to tailscale
      after = ["network-pre.target" "tailscale.service"];
      wants = ["network-pre.target" "tailscale.service"];
      wantedBy = ["multi-user.target"];

      serviceConfig.Type = "oneshot";

      script = with pkgs; ''
        # wait for tailscaled to settle
        sleep 2

        # check if we are already authenticated to tailscale
        status="$(${tailscale}/bin/tailscale status -json | ${jq}/bin/jq -r .BackendState)"
        # if status is not null, then we are already authenticated
        echo "tailscale status: $status"
        if [ "$status" != "NeedsLogin" ]; then
            exit 0
        fi

        # otherwise authenticate with tailscale
        # timeout after 10 seconds to avoid hanging the boot process
        ${coreutils}/bin/timeout 10 ${tailscale}/bin/tailscale up \
          ${lib.optionalString (cfg.loginServer != "") "--login-server=${cfg.loginServer}"} \
          --authkey=$(cat "${cfg.authkeyFile}")

        # we have to proceed in two steps because some options are only available
        # after authentication
        ${coreutils}/bin/timeout 10 ${tailscale}/bin/tailscale up \
          ${lib.optionalString (cfg.loginServer != "") "--login-server=${cfg.loginServer}"} \
          ${lib.optionalString (cfg.advertiseExitNode) "--advertise-exit-node"} \
          ${lib.optionalString (cfg.exitNode != "") "--exit-node=${cfg.exitNode}"} \
          ${lib.optionalString (cfg.exitNodeAllowLanAccess) "--exit-node-allow-lan-access"}
      '';
    };

    networking.firewall = {
      trustedInterfaces = [ "tailscale0" ];
      allowedUDPPorts = [ config.services.tailscale.port ];
    };

    services.tailscale = {
      enable = true;
      useRoutingFeatures = if cfg.advertiseExitNode then "server" else "client";
    };
  };
```
First, the assertions. They're here to make sure that the user doesn't make any mistake when configuring the module. For example, a user cannot both advertise an exit node and set an exit node.
Then, the service. We're using systemd to run a script that will connect to Tailscale. The `after`, `wants` and `wantedBy` options make the script run after the network is up and after Tailscale daemon is started. The `Type` option is here to make sure that the script is run only once. The script itself is a bit long, but it's just a bunch of bash commands. It's pretty straightforward. First, we wait for the Tailscale daemon to settle. Then, we check if we're already authenticated. If we are, we exit. Otherwise, we authenticate. Finally, we connect to Tailscale. We have to do it in two steps because some options are only available after authentication.

At the end, we configure the firewall to allow Tailscale traffic, and we enable the Tailscale service.

Now, an example of how to use this module:
```nix
{ outputs, ...}:
{
  imports = [
    outputs.nixosModules.tailscale-autoconnect
  ];

  services.tailscaleAutoconnect = {
    enable = true;
    authkeyFile = config.sops.secrets.tailscale_key.path;
    loginServer = "https://login.tailscale.com";
    exitNode = "some-node-id";
    exitNodeAllowLanAccess = true;
  };

  sops.secrets.tailscale_key = {
    sopsFile = ./secrets.yaml;
  };

  environment.persistence = {
    "/persist".directories = ["/var/lib/tailscale"];
  };
}
```
The module is imported and configured. We also use the `sops` secret we created earlier. Finally, we persist the Tailscale state, so that we don't have to authenticate again after a reboot. This is especially important if the authkey can expire.

---

That's all for this post. Thanks for reading! If you have any question, feel free to ask in the comments. The final code can be found [here](https://github.com/Guekka/nixos-server/tree/2-tailscale).

<!--
# Our first service : a simple file server

Our first service will be a basic web server. For that, we will use nginx. `nginx` is, obviously, a web server. However, it is often also used as a reverse proxy, forwarding the requests it gets to the internal services. `nginx.Nix`:
```nix
{
  services.nginx = {
    enable = true;
    recommendedTlsSettings = true;
    recommendedProxySettings = true;
    recommendedGzipSettings = true;
    recommendedOptimisation = true;
    clientMaxBodySize = "300m";
  };
  networking.firewall.allowedTCPPorts = [ 80 443 ];
}
```
`web-server.Nix`:
```nix
{
    services.nginx.virtualHosts."e1.oze.li" = {
      addSSL = true;
      enableACME = true;
      root = "/var/www/files";
    };

    environment.persistence = {
      "/persist".directories = [ "/var/www/files" ];
    };
}
```
We are enabling the `nginx` service. For now, it only serves `e1.oze.li`, our web server. You can notice the `addSSL` parameter, together with `enableACME`: thanks to them, we are getting a HTTPS certificate for free, provided by LetsEncrypt. We need to accept its terms. `acme.Nix`:
```nix
  # Enable acme for usage with nginx vhosts
  security.acme = {
    defaults.email = "some@email.com";
    acceptTerms = true;
  };

  environment.persistence = {
    "/persist".directories = [ "/var/lib/acme" ];
  };
}
```
That's all! That is so simple and straightforward. I couldn't imagine myself using only Docker now.
-->
