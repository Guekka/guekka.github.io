# NixOS as a server, part 2: flake, tailscale, nginx

In the [previous part](https://guekka.github.io/nixos-server-1/), we configured our NixOS server to use impermanence. I have made a few changes since, most notably moving to a proper VM in Proxmox.

# Moving to flakes

> If you already know about flakes, you can safely ignore this part.

Have you heard about nix flakes? If you have been in the Nix ecosystem for more than a few days, most likely. They're the shiny new way, still experimental but used everywhere.
Their main advantage over traditional Nix is the purity, for example with the ability to state `inputs`. When we say Nix is reproducible, we assume `channels`, the way we get our packages, will stay the same. But this is not true, and flakes fix this: channels are now specified alongside Nix configuration.
For a more in depth introduction, have a look at [the wiki](https://nixos.wiki/wiki/Flakes#See_also).

Now, why do we want to migrate to flakes? We do not have external requirements, do we?
Well, yes, we do. Apart from the obvious `nixpkgs` dependency, which is configured system-wide, we are importing the impermanence module :
```nix
  impermanence = builtins.fetchTarball "https://github.com/nix-community/impermanence/archive/master.tar.gz";
```
This is bad by the way, as we do not specify the expected hash.
Now that I intend to add more modules, and possibly use the `unstable` channel, it is better to migrate. Let's see what our entry point would look like:
```nix
{
  # what is consumed (previously provided by channels and fetchTarball)
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-22.11";
    impermanence.url = "github:nix-community/impermanence";
  };

  # what will be produced (i.e. the build)
  outputs = { nixpkgs, ... }@inputs: { # (1)
    nixosConfigurations = {
      server = nixpkgs.lib.nixosSystem { # (2)
        system = "x86_64-linux"; # set system type
        specialArgs = inputs; # forward inputs to modules
        modules = [
          ./configuration.nix
        ];
      };
    };
  };
}
```
`(1)` is defining a function. Most complicated things are defined using functions in Nix. It takes a *named attribute set* argument. So this syntax:
```nix
{ nixpkgs, ... }@inputs: nixpkgs
```
is the same as:
```nix
inputs: inputs.nixpkgs
```
But we don't have to repeat `inputs.` everywhere.

`(2)` is actually making the system. We call the `nixosSystem` function and pass it some arguments. Yes, the whole system counts as an `output`.

That's pretty much it. With that `flake.nix` in `/etc/nixos`, `nixos-rebuild` will work as before. However, if you're using git, beware that your files all need to be under version control or flakes will not see them.

# Secrets with Sops

In order to setup Tailscale, we will use a pre-auth key. This will allow us to connect to our server without interaction. However, we must hide this key, or other people could join our Tailscale network, which would be very bad obviously.

There are 2 well-known solutions : age and sops. I've chosen sops for no particular reason.
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
What's up with this `follows`? Well, `sops-nix` already depends on `nixpkgs`, but it might use a different revision than ours. Making it use our own has several advantages:
- improve consistency
- reduce the number of evaluations required

And how do we know if a package needs to be redirected? Well, it's difficult. Either we look at the upstream `flake.nix`, or we can call `nix flake info` and get a graph like so:
```
Resolved URL:  git+file:///etc/nixos
Locked URL:    git+file:///etc/nixos
Path:          /nix/store/4b14z6ki7av3kid69sp5vgf50wzd3a73-source
Last modified: 2023-04-17 14:04:13
Inputs:
├───impermanence: github:nix-community/impermanence/6138e
├───nixpkgs: github:NixOS/nixpkgs/39fa0
└───sops-nix: github:mic92/sops-nix/de651
    ├───nixpkgs follows input 'nixpkgs'
    └───nixpkgs-stable: github:NixOS/nixpkgs/1040c
```
We can notice `sops-nix` also has a `nixpkgs-stable` input, that we might as well redirect.

## Generating a key

`sops-nix` works by encrypting our secrets with private keys.
We thus need to provide it with the keys we will use. We can generate an `age` key, or get one from our `SSH` host key.
Each secrets group can have different allowed keys, so that X user cannot access Y secrets.
I will use the SSH host key for my server:
```
$ nix-shell -p ssh-to-age --run 'cat /etc/ssh/ssh_host_ed25519_key.pub | ssh-to-age'
age1dt24qetqhy2ng53fyj69yq9hg8rdsg4ep0lvvhdg69xw9v4l0asqj6xzkh
```
> Isn't this insecure? The key is not password-locked.

Yes, it is. However, remember that if someone has access to our user account, they can already read our secrets, so it doesn't matter. Our goal is to be able to put these secrets on a public GitHub, that's all.

We now have to set `.sops.yaml` config:
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
Now we're able to decrypt secrets. However, we need to write them too. We can get the corresponding private key this way:
```sh
nix-shell -p ssh-to-age --run "sudo ssh-to-age -private-key -i /etc/ssh/ssh_host_ed25519_key | install -D /dev/stdin ~/.config/sops/age/keys.txt"
```

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

Then we import `sops`. Finally, we give him the keys we collected earlier. This avoids hardcoding keys, which is great!

We can now import this module in our config:
```nix
  imports =
    [
      impermanence.nixosModule
      ./hardware-configuration.nix
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
An editor should open. We can now write secrets, using yaml. Once we're done, we can save the file.

We now have to tell `sops` this secret exists. So we declare it somewhere in our configuration:
```
sops.secrets.tailscale_key = {
  sopsFile = ./secrets.yaml;
};
```

And that's all! To use it, we simply have to use `config.sops.secrets.tailscale_key.path` where we need it. Beware that this will not give you the secret, but a path to a file containing the raw secret, for security reasons.

## Note: adding a new host

If you ever need to add a new host, you will need to update your secrets with `sops updatekeys your_secret`. This command has to be run in a place with already authorized keys.

# Tailscale

We can finally get to a real feature, setting up Tailscale.
For those of you who haven't heard of it, Tailscale is a private meshed network, allowing you to connect to your machines privately and securely through Wireguard, a VPN protocol, without exposing them to the world.
This means being able to close port 22, while still being able to SSH into your computer for example.

Let's write a tailscale module.
```nix
{ config, pkgs, ... }:
{
  # enable the tailscale service
  services.tailscale.enable = true;

  # create a oneshot job to authenticate to Tailscale
  systemd.services.tailscale-autoconnect = {
    description = "Automatic connection to Tailscale";

    # make sure tailscale is running before trying to connect to tailscale
    after = [ "network-pre.target" "tailscale.service" ];
    wants = [ "network-pre.target" "tailscale.service" ];
    wantedBy = [ "multi-user.target" ];

    serviceConfig.Type = "oneshot";

    script = with pkgs; ''
      # wait for tailscaled to settle
      sleep 2

      # check if we are already authenticated to tailscale
      status="$(${tailscale}/bin/tailscale status -json | ${jq}/bin/jq -r .BackendState)"
      if [ $status = "Running" ]; then # if so, then do nothing
        exit 0
      fi

      # otherwise authenticate with tailscale
      ${tailscale}/bin/tailscale up --authkey $(cat ${config.sops.secrets.tailscale_key.path}) --login-server https://headscale.ozeliurs.com --advertise-exit-node
    '';
  };

  environment.systemPackages = with pkgs; [
    tailscale
  ];

  # Open ports in the firewall.
  networking.firewall = {
    # always allow traffic from your Tailscale network
    trustedInterfaces = [ "tailscale0" ];

    allowedUDPPorts = [ config.services.tailscale.port ];

    checkReversePath = "loose";
  };

  # for exit node
  boot.kernel.sysctl."net.ipv4.ip_forward" = 1;
  boot.kernel.sysctl."net.ipv6.conf.all.forwarding" = 1;

  sops.secrets.tailscale_key = {
    sopsFile = ./secrets.yaml;
  };

  environment.persistence = {
    "/persist".directories = [ "/var/lib/headscale" ];
  };
}
```
That's a bunch of code! What does it do?
- enable tailscale service
- `tailscale-autonnect` uses our preauth-key to connect automatically to Tailscale
- make the `tailscale` command available
- allow `tailscale` through the firewall
- configure some kernel parameters for allowing an exit node
- load the tailscale secret
- persist `/var/lib/tailscale` directory

We can simply import this module in our configuration, and there we go!

# Our first service : a simple file server

Our first service will be a basic web server. For that, we will use nginx. `nginx` is, obviously, a web server. However, it is often also used as a reverse proxy, forwarding the requests it gets to the internal services. `nginx.nix`:
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
`web-server.nix`:
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
We are enabling the `nginx` service. For now, it only serves `e1.oze.li`, our web server. You can notice the `addSSL` parameter, together with `enableACME`: thanks to them, we are getting a HTTPS certificate for free, provided by LetsEncrypt. We need to accept its terms. `acme.nix`:
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